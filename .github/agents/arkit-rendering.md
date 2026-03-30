# ARKit, SceneKit and Rendering

## ARKit Session Configuration

`ARConfigurationFactory` selects and configures the ARKit session based on `MixedRealityConfiguration.captureMode`:

| Capture mode | ARKit configuration class | Key frame semantics |
|-------------|--------------------------|-------------------|
| `.personSegmentation` | `ARWorldTrackingConfiguration` | `personSegmentationWithDepth` (or `personSegmentation` without LiDAR) |
| `.greenScreen` | `ARWorldTrackingConfiguration` | Default frame semantics |
| `.bodyTracking(avatar:)` | `ARBodyTrackingConfiguration` | Tracks body joints via `ARBodyAnchor` |
| `.raw` | `ARWorldTrackingConfiguration` | Default frame semantics |

Always check `ARBodyTrackingConfiguration.isSupported` before using body tracking. Use `ARWorldTrackingConfiguration.supportsFrameSemantics(.personSegmentationWithDepth)` to decide between the depth-enhanced and simple segmentation paths. `CaptureMode.isSupported` (defined in an extension) provides this check.

## SceneKit Layer Architecture

The AR scene uses a flat, camera-relative node hierarchy. Three invisible planes are attached to `sceneView.pointOfView` on the first `ARSessionDelegate.session(_:didUpdate frame:)` call:

```
pointOfView (camera)
  ├── backgroundPlane  (distance 100 m)  — VR background layer
  ├── middlePlane      (distance  0.02 m) — chroma-keyed VR foreground (green-screen mode)
  └── foregroundPlane  (distance  0.01 m) — VR foreground with alpha (AR mode)
```

Each plane is sized to fill the viewport. Distance ordering ensures correct depth-buffer sorting. Planes are created in `ARKitHelpers.makePlaneNodeForDistance(_:frame:)`.

A far opaque plane (9999 × 9999 at distance 120 m) is optionally attached to prevent the ARKit camera feed from showing when the background layer is visible.

## YCbCr Texture Binding

The Quest streams H.264 video; decoded `CVPixelBuffer` values are in **YCbCr (biplanar 420)** format. Each frame is split into two Metal textures:

```swift
let luma   = ARKitHelpers.texture(from: pixelBuffer, format: .r8Unorm,  planeIndex: 0, textureCache: textureCache)
let chroma = ARKitHelpers.texture(from: pixelBuffer, format: .rg8Unorm, planeIndex: 1, textureCache: textureCache)
```

- `luma` is assigned to `SCNMaterialProperty.transparent` (used in shaders as `u_transparentTexture`).
- `chroma` is assigned to `SCNMaterialProperty.diffuse` (used as `u_diffuseTexture`).

`CVMetalTextureCache` must be created once (`ARKitHelpers.create(textureCache:for:)`) and reused for every frame to avoid GPU resource leaks.

## Surface Shaders (GLSL)

All shaders live in `Capture/Shaders/Shaders.swift` as Swift static strings/methods. They are injected via:

```swift
material.shaderModifiers = [.surface: Shaders.backgroundSurface]
```

### Shared Helper Functions

**`yCbCrToRGB`** — BT.709 YCbCr → linear RGB. Called in every shader that renders video data.

**`rgbToYCrCb`** — RGB → YCrCb. Used as input to chroma-key distance calculation.

**`smoothChromaKey`** — Computes a [0, 1] blend value based on the CrCb distance to a target colour, with `smoothstep` for anti-aliasing. Parameters:
- `sensitivity` — minimum distance (hard edge start)
- `smoothness` — width of the transition region

### Texture Coordinate Mapping

The Quest MRC protocol sends a single 1920 × 1080 frame whose horizontal space is divided into quadrants:

| Quadrant (x range) | Content |
|-------------------|---------|
| 0.0 – 0.5 | Background layer |
| 0.5 – 0.75 | Foreground layer |
| 0.75 – 1.0 | Foreground alpha mask |

Background shader maps `diffuseTexcoord.x` to `[0, 0.5]`:
```glsl
vec2 backgroundCoords = vec2(_surface.diffuseTexcoord.x * 0.5, _surface.diffuseTexcoord.y);
```

Foreground shader maps to `[0.5, 0.75]`:
```glsl
vec2 foregroundCoords = vec2((_surface.diffuseTexcoord.x * 0.25) + 0.5, _surface.diffuseTexcoord.y);
```

### Transparency Model

SceneKit's `.rgbZero` transparency mode means: **black = opaque, white = fully transparent**. Shaders set `_surface.transparent` to a greyscale vec4 where the value encodes the desired transparency level.

## Flip Transform

When `shouldFlipOutput` is `true` the output is vertically mirrored using a SceneKit material `contentsTransform`:

```swift
let flipTransform = SCNMatrix4Translate(SCNMatrix4MakeScale(1, -1, 1), 0, 1, 0)
material.diffuse.contentsTransform    = flipTransform
material.transparent.contentsTransform = flipTransform
```

## Avatar Rendering (Body Tracking Mode)

When `captureMode == .bodyTracking(avatar:)`:

1. `ARBodyTrackingConfiguration` is used; ARKit emits `ARBodyAnchor` updates via `session(_:didUpdate anchors:)`.
2. `ARConfigurationFactory.buildAvatar(bodyAnchor:)` creates the avatar on the first anchor update.
3. The avatar's root `SCNNode` is added to `sceneView.scene.rootNode` (world space, not camera space).
4. Subsequent anchor updates call `avatar.update(bodyAnchor:)` which repositions joint nodes using `ARSkeleton3D.modelTransform(for:)`.
5. `sceneView.autoenablesDefaultLighting = true` is set for this mode to illuminate the 3D model.

## ARKit Person Segmentation (Virtual Green Screen)

`ChromaKeyMaskBuilder` uses `ARFrame.segmentationBuffer` (a greyscale `CVPixelBuffer` where white = person) to generate a static mask image that is baked to disk. At runtime the mask is loaded into `middlePlaneNode.geometry?.firstMaterial?.ambient.contents` to restrict the chroma-key effect to the person silhouette area.

## Display Link Loop

`CADisplayLink` drives the main update loop at 60 fps:

```swift
@objc func update(with sender: CADisplayLink) {
    while let data = client.read(65536, timeout: 0), data.count > 0 {
        oculusCapture?.add(data: .init(data))
    }
    oculusCapture?.update()
    if let lastFrame = lastFrame {
        updateForegroundBackground(with: lastFrame)
    }
}
```

Network I/O, frame decoding, and texture uploads all happen on the main thread synchronised with the display link. Do not move network reads to a background thread without also ensuring thread-safe access to `OculusCapture` and the SceneKit scene.
