# RealityMixer – GitHub Copilot Instructions

RealityMixer is an **iOS/iPadOS app** (Swift, Xcode) that enables Mixed Reality capture of Oculus Quest VR games without a PC or physical green screen. It streams H.264 video and PCM audio from the Quest over TCP, composites the person (extracted via ARKit segmentation) into the VR scene, and outputs a recording.

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Language | Swift 5 |
| Platform | iOS 14+ (iPhone/iPad, A12+) |
| AR / 3D | ARKit, SceneKit |
| GPU shaders | SceneKit GLSL surface shaders (Metal-backed) |
| Video decoding | VideoToolbox (VTDecompressionSession) |
| Audio | AVFoundation (AVAudioPCMBuffer, AVAudioEngine) |
| Networking | SwiftSocket (TCPClient) |
| Build system | Xcode (no CocoaPods / SPM / Carthage) |

## Project Layout

```
RealityMixer/
  Calibration/         Camera-to-VR world calibration
    Builders/          CalibrationBuilder (homography solver)
    Extensions/        ARKit/SIMD helpers
    Model/             Math.swift (quaternion / SIMD helpers)
    Network/           OculusCalibration (calibration TCP protocol)
    ViewControllers/   CalibrationViewController
  Capture/             MRC streaming and AR compositing
    Audio/             AudioManager, AudioQueue
    Avatars/           AvatarProtocol, Skeleton, RobotAvatar, ReadyPlayerMeAvatar
    Configuration/     MixedRealityConfiguration, capture mode enums
    Extensions/        ARKit / SceneKit helpers (ARKitHelpers)
    MRC/               OculusCapture, VideoDecoder, CaptureFrame, CapturePayload
    Misc/              ChromaKeyMaskBuilder, CameraPoseSender, ARConfigurationFactory
    Shaders/           Shaders.swift (GLSL strings for SceneKit surface shaders)
    ViewControllers/   MixedRealityViewController (main AR compositor)
  Main/
    ViewControllers/   InitialViewController, connection flow
  Resources/           Assets, 3D models (.stl), avatar files, sounds
```

## Key Conventions

- Follow the existing file-level style: copyright header comment, then `import` statements, then `// MARK: -` sections.
- Use `final class` for concrete types that are not designed for subclassing.
- Use `weak var delegate` for all delegate references.
- Prefer `private` access control on implementation details; expose only what is genuinely needed.
- Use `lazy` stored properties for expensive objects whose initialization depends on `self` (e.g., `VideoDecoder`).
- Enums are the canonical way to represent type-safe union payloads (see `CapturePayload`).
- SceneKit shader customisation is done via `SCNMaterial.shaderModifiers[.surface]` with GLSL strings defined in `Shaders.swift`.
- There are **no automated tests** in this project. Manual testing on device is the validation path.

## Detailed Guidance

See the additional steering files in `.github/agents/` for deeper guidance on specific topics:

- **[architecture.md](.github/agents/architecture.md)** – Module map, data flow, design patterns
- **[swift-conventions.md](.github/agents/swift-conventions.md)** – Swift style, naming, error handling
- **[arkit-rendering.md](.github/agents/arkit-rendering.md)** – ARKit session setup, SceneKit layers, GLSL shaders
- **[networking.md](.github/agents/networking.md)** – TCP protocol, frame assembly, H.264 decoding
