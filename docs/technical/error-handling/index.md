# Error Handling

This page documents the error handling implemented in the current FOSIA modules. The modules run as part of a Hub-managed pipeline. The Hub and Module SDK provide module registration, readiness, routing and control-message delivery, while the modules handle microphone capture, WebRTC communication, audio processing and speech recognition.

For details about the framework, see the [Hub documentation](https://docs.fosia.org/en/latest/technical/framework/hub/) and the [Module SDK documentation](https://docs.fosia.org/en/latest/technical/framework/module-sdk/writing-a-module/).

## Current error-case overview

| Module | Error case | Current behavior | Recovery |
|---|---|---|---|
| Microphone | PortAudio fails during startup | Attempts to start audio capture using `parec` as a PulseAudio fallback. | Automatic fallback if `parec` is available and can start successfully. |
| Microphone | `parec` is unavailable | Raises a descriptive `RuntimeError`. | Install or configure the required audio capture backend. |
| Microphone | PulseAudio capture fails | Reports the error using process output or the exit code. | Check the audio source and PulseAudio configuration. |
| Microphone | `parec` stops during capture | Raises a `RuntimeError`. | The capture process must be restarted; no automatic restart is implemented. |
| Microphone | Audio callback reports a status message | Logs the status. | Capture continues unless the stream itself fails. |
| Microphone | No downstream destination is available | Logs a warning when audio cannot be forwarded. | Check module readiness and Hub routing configuration. |
| WebRTC client | Hub reply is missing `sdp` or `type` | Logs an error and stops the current run. | Check the request/reply exchange and returned message. |
| WebRTC client | Hub reply is not an SDP answer | Logs an error and stops the current run. | Check the receiver's control-message handler. |
| WebRTC client | Applying the remote SDP fails | Logs the exception and exits the current run. | Check the SDP exchange and WebRTC negotiation. |
| WebRTC client | Audio track ends | Logs the end of the stream and closes the peer connection. | Normal stream termination and cleanup. |
| WebRTC client | Connection state changes or fails | Logs connection-state changes. | No complete automatic reconnection or renegotiation strategy is implemented. |
| Receiver | Audio-frame conversion fails | Logs and re-raises the conversion error. | The receive loop handles the error and performs cleanup. |
| Receiver | Unexpected receive-loop error | Logs the exception and runs cleanup. | The outer loop attempts to continue where possible. |
| Receiver | No downstream destination is available | Logs a warning when audio cannot be forwarded. | Check module readiness and Hub routing configuration. |
| Receiver | Audio stream ends | Attempts to flush buffered audio processing and sends `EOS` downstream. | Normal stream termination. |
| STT | Configured test audio file is missing | Raises `FileNotFoundError`. | Restore the file or disable file-based test mode. |
| STT | Google Speech-to-Text API raises an error | The error propagates through the module runtime; no dedicated retry policy is implemented. | Inspect the API error, credentials, network connection and request configuration. |
| STT | `EOS` is received | Processes the remaining partial audio bytes and queues an end marker for the request stream. | Normal termination of the recognition request. |

## Lifecycle and cleanup

### Microphone

The microphone module waits for the selected capture backend to start. If PortAudio fails during startup, it attempts to use `parec` as a fallback. During shutdown, the module signals the capture tasks to stop, terminates the fallback process if necessary, sends `EOS` downstream and waits for the recording and forwarding tasks to finish.

### WebRTC client

The client sends a `webrtc-tunnel-offer` message through the Hub and validates the returned SDP answer before applying it to the peer connection. Its `finally` block closes the `RTCPeerConnection`, including when negotiation fails or the reply is invalid. Connection-state changes are logged, but automatic reconnection and renegotiation are not fully implemented.

### Receiver

The receiver processes incoming WebRTC audio, converts it to mono, 16 kHz signed 16-bit PCM and forwards the resulting audio data downstream. When the stream ends or an error occurs, it attempts to flush buffered audio processing, close the recording container and send `EOS` downstream. Some cleanup exceptions are suppressed.

### Speech-to-Text

The STT module consumes PCM audio data and streams it to Google Speech-to-Text. When `EOS` arrives, it flushes the remaining partial audio chunk and terminates the request stream. There is no explicit retry or backoff strategy for Google API errors.

## Known implementation limitations

The following limitations are relevant when operating or testing the current modules:

1. **WebRTC recovery:** Connection-state changes are logged, but a complete automatic reconnection and renegotiation strategy is not implemented.
2. **Google API recovery:** Speech-to-Text errors do not have a dedicated retry or backoff mechanism.
3. **STT test mode:** The file-based test path currently yields PyAV `AudioFrame` objects, although the request generator expects audio bytes. The module also waits for data on the live audio queue. File-based testing should be corrected before relying on it.
4. **Cleanup visibility:** Some receiver cleanup exceptions are suppressed and may therefore not appear in the logs.
5. **Module readiness:** Starting a process does not necessarily mean that its module has registered with the Hub and is ready to receive data.

## Troubleshooting

- **No microphone audio:** Check `MIC` logs for capture startup, PortAudio errors and fallback messages.
- **WebRTC negotiation fails:** Check `CLIENT` logs and the receiver's Hub control-message handler.
- **No audio reaches the server:** Check WebRTC connection-state logs, receiver track handling and Hub routing.
- **No transcript is produced:** Check receiver forwarding, `EOS` delivery, the `STT` logs and Google API configuration.

Only document retries, timeouts or automatic recovery when the current implementation actually provides them.