# Architecture

## Module Overview

### `Capture/MRC/` — Network Capture Core

| File | Responsibility |
|------|---------------|
| `OculusCapture.swift` | Orchestrates capture: feeds raw TCP bytes into `CaptureFrameCollection`, dispatches complete frames to `CapturePayload`, routes video to `VideoDecoder` and audio to the delegate |
| `CaptureFrame.swift` | Stateful frame assembler; buffers byte stream, detects frame boundaries via magic number, exposes complete `CaptureFrame` values |
| `CapturePayload.swift` | Decodes a raw `CaptureFrame` into a typed enum (`videoDimension`, `videoData`, `audioSampleRate`, `audioData`) |
| `VideoDecoder.swift` | Wraps `VTDecompressionSession`; parses H.264 NAL units, extracts SPS/PPS, outputs `CVPixelBuffer` via `DecoderDelegate` |

### `Capture/ViewControllers/` — AR Compositor

`MixedRealityViewController` is the single orchestrator for the live session:

1. Reads TCP bytes from `TCPClient` every display-link tick.
2. Passes bytes to `OculusCapture`, which decodes video frames.
3. Implements `ARSessionDelegate` to receive `ARFrame` events from ARKit.
4. On the first ARKit frame, creates three SceneKit plane nodes anchored to the camera:
   - **Background plane** (distance 100 m) — renders the VR background layer.
   - **Middle plane** (distance 0.02 m) — renders the VR foreground with chroma-key compositing (green-screen mode only).
   - **Foreground plane** (distance 0.01 m) — renders the VR foreground layer (AR mode).
5. Each plane's material uses a custom GLSL surface shader (from `Shaders.swift`) to convert YCbCr textures to RGB and apply transparency.

### `Calibration/` — Camera-to-VR Calibration

`CalibrationBuilder` computes the affine transform from iPhone camera space to Oculus Quest world space using marker positions detected in `ARFrame`. `OculusCalibration` implements the binary calibration protocol over TCP and stores the result via `CalibrationResultStorage`.

### `Capture/Avatars/` — Avatar System

All avatar types conform to `AvatarProtocol`:

```swift
protocol AvatarProtocol {
    var mainNode: SCNNode { get }
    func update(bodyAnchor: ARBodyAnchor)
}
```

`ARConfigurationFactory.buildAvatar(bodyAnchor:)` is the factory entry point. `Skeleton` drives bones using `ARSkeleton3D` joint transforms. `RobotAvatar` and `ReadyPlayerMeAvatar` are concrete implementations.

### `Capture/Misc/`

| File | Responsibility |
|------|---------------|
| `ARConfigurationFactory` | Creates the correct `ARConfiguration` subclass based on `MixedRealityConfiguration.captureMode` |
| `ChromaKeyMaskBuilder` | Uses ARKit's `personSegmentationWithDepth` (or `personSegmentation`) to build a static mask image saved to disk |
| `CameraPoseSender` | Serialises the iPhone's `ARCamera.transform` and sends it over the existing TCP connection so the Quest can adjust its virtual camera |

## Data Flow

```
TCPClient  ──bytes──►  OculusCapture.add(data:)
                            │
                    CaptureFrameCollection
                            │  (complete CaptureFrame)
                    CapturePayload (enum)
                       ├── .videoData  ──►  VideoDecoder  ──CVPixelBuffer──►  OculusCaptureDelegate
                       ├── .audioData  ──────────────────────────────────────►  OculusCaptureDelegate
                       ├── .videoDimension  (stored locally)
                       └── .audioSampleRate (stored locally)

ARKit session ──ARFrame──►  MixedRealityViewController
                                ├── (first frame) build SCNNode layers
                                ├── update YCbCr textures on planes
                                └── (body tracking) update avatar
```

## Design Patterns

| Pattern | Where |
|---------|-------|
| Delegate | `OculusCaptureDelegate`, `DecoderDelegate`, `ARSessionDelegate` |
| Factory | `ARConfigurationFactory`, `CalibrationBuilder` |
| Enum-as-union | `CapturePayload`, `CaptureMode`, `LayerVisibility` |
| Protocol-oriented | `AvatarProtocol`, `OculusCaptureDelegate` |
| Lazy stored property | `VideoDecoder` in `OculusCapture` |
| GLSL embedded in Swift | `Shaders.swift` static string properties / methods |
| Display-link driven loop | `CADisplayLink` calling `update(with:)` at 60 fps |
| UserDefaults persistence | `ChromaKeyConfigurationStorage`, `CalibrationResultStorage` |

## Configuration Model

`MixedRealityConfiguration` is the central value type passed through the capture flow. It holds:

- `captureMode: CaptureMode` — `.personSegmentation`, `.greenScreen`, `.bodyTracking(avatar:)`, `.raw`
- `backgroundLayerOptions` — `BackgroundLayerOptions` with `BackgroundVisibility` (`.visible`, `.hidden`, `.chromaKey(color:)`)
- `foregroundLayerOptions` — `ForegroundLayerOptions` with `ForegroundVisibility` (`.visible(useMagentaAsTransparency:)`, `.hidden`)
- `shouldFlipOutput: Bool`
- `enableMovingCamera: Bool`
- `enableAutoFocus: Bool`

Changing the capture mode changes which `ARConfiguration` is built (`ARBodyTrackingConfiguration` vs `ARWorldTrackingConfiguration` with different frame semantics) and which SceneKit shader is applied.
