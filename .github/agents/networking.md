# Networking and Capture Protocol

## TCP Connection

The app connects to the Oculus Quest over **5 GHz Wi-Fi** using **SwiftSocket's `TCPClient`**. The Quest must be running the Oculus MRC app (or a compatible app) and both devices must be on the same LAN.

```swift
let client = TCPClient(address: host, port: port)
let result = client.connect(timeout: 10)
```

After a successful connection the app enters either the calibration flow or the live capture flow. The same `TCPClient` object is reused throughout the session; `client.close()` is called on disconnect.

## Frame Protocol

All data from the Quest is sent as a stream of binary frames. `CaptureFrameCollection` reassembles the byte stream into complete frames.

### Frame Layout

```
┌─────────────────────────────────────────────────────┐
│ Magic  (4 bytes, UInt32 LE) = 0x4D524300             │
│ Payload type (4 bytes, UInt32 LE)                    │
│ Payload length (4 bytes, UInt32 LE)                  │
│ Payload (variable, `payloadLength` bytes)            │
└─────────────────────────────────────────────────────┘
```

`CaptureFrame` holds the decoded header fields and the payload `Data`. The magic number guards against byte-stream desynchronisation.

### Payload Types (`CapturePayloadType`)

| Raw value | Case | Payload layout |
|-----------|------|---------------|
| `10` | `.videoDimension` | `Int32` width + `Int32` height (8 bytes) |
| `11` | `.videoData` | Raw H.264 byte stream |
| `12` | `.audioSampleRate` | `UInt32` sample rate (4 bytes) |
| `13` | `.audioData` | `AudioDataHeader` (16 bytes) + PCM float32 samples |

`CapturePayload.init?(from:)` parses raw `CaptureFrame` values into the typed enum. Return `nil` for unknown or malformed frames (logged with a commented-out `print`).

## H.264 Video Decoding (`VideoDecoder`)

`VideoDecoder` wraps `VTDecompressionSession` for hardware-accelerated H.264 decoding.

### NAL Unit Processing

H.264 data arrives as a stream prefixed with 4-byte start codes (`0x00 0x00 0x00 0x01`). The decoder:

1. Splits the stream on start codes to extract individual NAL units.
2. Identifies **SPS** (NAL type 7) and **PPS** (NAL type 8) units.
3. Once both SPS and PPS are available, creates a `CMVideoFormatDescription` via `CMVideoFormatDescriptionCreateFromH264ParameterSets`.
4. Creates a `VTDecompressionSession` with the format description.
5. Converts subsequent non-parameter-set NAL units from Annex-B (start-code) format to AVCC (length-prefixed) format before passing to `VTDecompressionSessionDecodeFrame`.
6. Delivers decoded `CVPixelBuffer` values via `DecoderDelegate.didDecodeFrame(_:)`.

### Decoder Delegate

```swift
protocol DecoderDelegate: AnyObject {
    func didDecodeFrame(_ buffer: CVPixelBuffer)
}
```

`OculusCapture` implements `DecoderDelegate` and forwards the buffer to its own `OculusCaptureDelegate`.

### Thread Safety

`VTDecompressionSession` callbacks arrive on an internal VideoToolbox thread. `OculusCapture` stores the latest decoded buffer and reads it back on the main thread via the display-link tick — the intermediate `lastFrame` property in `MixedRealityViewController` bridges this boundary. If you add multithreading, protect `lastFrame` with a lock or use a dedicated dispatch queue.

## Audio Decoding

The Quest sends PCM float32 samples. `OculusCapture.processAudio(_:data:)`:

1. Reads `AudioDataHeader` to get channel count (1 or 2) and sample count.
2. Creates an `AVAudioPCMBuffer` with `pcmFormatFloat32`, non-interleaved.
3. De-interleaves the incoming interleaved samples manually into separate channel buffers.
4. Forwards the buffer with its `UInt64` timestamp to `OculusCaptureDelegate.oculusCapture(_:didReceiveAudio:timestamp:)`.

`AudioManager` wraps `AVAudioEngine` and schedules buffers for low-latency playback.

## Camera Pose Protocol (Moving Camera)

When `enableMovingCamera` is `true`, `CameraPoseSender` serialises the iPhone's AR camera transform and sends it back to the Quest on every `ARFrame` update:

```swift
func didUpdate(frame: ARFrame) {
    let transform = frame.camera.transform  // simd_float4x4
    // serialise and write to client
}
```

This allows the Quest application to adjust the virtual camera viewpoint in real time to match the iPhone's position and orientation, enabling a "moving camera" MRC effect.

## Calibration Protocol

`OculusCalibration` uses the same `TCPClient` during the calibration phase. It sends and receives a binary calibration handshake; `CalibrationBuilder` then computes the camera-to-world transform from the agreed-upon marker positions. The result is stored via `CalibrationResultStorage` (UserDefaults) and reused in subsequent sessions.

## Error Handling and Disconnection

- `client.read(65536, timeout: 0)` returns `nil` or an empty array when the connection is closed; the display-link loop exits naturally.
- On `UIApplication.willResignActiveNotification`, `disconnect()` is called immediately to close the TCP socket and stop the AR session. This prevents the app from holding a stale connection in the background.
- Reconnection requires navigating back to the initial screen and starting a new session.
