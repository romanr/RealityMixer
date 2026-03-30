# Swift Conventions

## File Structure

Every Swift file follows this order:

```swift
//
//  FileName.swift
//  RealityMixer
//
//  Created by <Author> on <date>.
//

import Foundation     // or UIKit / ARKit / etc.

// MARK: - Protocol / Type definitions

// MARK: - Extensions
```

Use `// MARK: -` comment dividers to separate logical sections within a file (same style as the Xcode default). Do not use `// MARK:` (without the dash) for top-level sections.

## Access Control

- `private` for all implementation details: stored properties, helper methods, IBOutlet/IBAction connections.
- `internal` (default, no keyword) for types and members that must be visible within the module.
- `public` is not needed — this is an app target, not a framework.
- Mark every class that is not designed for subclassing `final`.

```swift
// Correct
final class OculusCapture {
    weak var delegate: OculusCaptureDelegate?
    private let frameCollection = CaptureFrameCollection()

    private func process(_ payload: CapturePayload) { ... }
}
```

## Delegate Pattern

- All delegate protocols are named `<Type>Delegate`.
- Delegate protocols inherit from `AnyObject`.
- Delegate references are always `weak var delegate: SomeDelegate?`.
- Delegate method signatures use the delegating object as the first parameter, following Cocoa naming conventions:

```swift
protocol OculusCaptureDelegate: AnyObject {
    func oculusCapture(_ oculusCapture: OculusCapture, didReceive pixelBuffer: CVPixelBuffer)
    func oculusCapture(_ oculusCapture: OculusCapture, didReceiveAudio audio: AVAudioPCMBuffer, timestamp: UInt64)
}
```

## Lazy Properties

Use `lazy` stored properties for objects whose initialisation requires `self` or is expensive:

```swift
private lazy var decoder: VideoDecoder = {
    VideoDecoder(delegate: self)
}()
```

Do not mark properties `lazy` just to defer initialization; only use it when there is a genuine reason.

## Enums for Type-Safe Payloads

Prefer enums with associated values over stringly-typed dictionaries or `Any` casts:

```swift
enum CapturePayload {
    case videoDimension(VideoDimension)
    case videoData(Data)
    case audioSampleRate(UInt32)
    case audioData(AudioDataHeader, Data)
}
```

Nested structs inside an enum are acceptable for grouping related fields (see `CapturePayload.VideoDimension`, `CapturePayload.AudioDataHeader`).

## Struct vs Class

- Use `struct` for pure value types (configuration, math helpers, parsed payloads).
- Use `final class` for reference types that own resources (view controllers, network objects, decoders).
- Use `class` (without `final`) only when you intend the type to be subclassed.

## Error Handling

- Prefer failable initialisers (`init?`) for parsing where failure is a normal expected case (see `CapturePayload.init?(from:)`).
- Use early-return `guard let` / `guard` statements rather than deeply nested `if let`.
- Avoid `try!` and `try?` in production code; catch and handle errors explicitly when they can occur.
- `fatalError("init(coder:) has not been implemented")` is acceptable in the `required init?(coder:)` of view controllers initialised programmatically.

## Naming

- Follow [Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/).
- Type names: `UpperCamelCase`.
- Properties, methods, local variables: `lowerCamelCase`.
- Boolean properties should read as assertions: `isHidden`, `shouldFlipOutput`, `enableMovingCamera`.
- Delegate method parameter labels omit the delegate type name redundancy at the call site (Cocoa style: `oculusCapture(_:didReceive:)` not `didReceivePixelBuffer(_:)`).

## Memory Management

- Use `[weak self]` in closures and block callbacks that might outlive `self`.
- Use `[weak oculusCapture, weak client]` when capturing multiple objects in a background thread block.
- Call `invalidate()` in `deinit` to release timers, display links, and network resources.

## UIKit Patterns

- ViewControllers are initialised programmatically using `init(nibName:bundle:)` with the nib named after the class (`String(describing: type(of: self))`).
- IBOutlets are `@IBOutlet private weak var`.
- IBActions are `@IBAction private func`.
- Override `prefersStatusBarHidden` and `prefersHomeIndicatorAutoHidden` as computed properties when needed.

## Commented-out Code

Commented-out code blocks representing alternative implementations (e.g., the network thread approach in `MixedRealityViewController`) may be left as documentation of design decisions. Do not remove them without understanding the trade-off they document. New commented-out experiments should include a brief explanation.
