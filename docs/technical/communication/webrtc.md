WebRTC Audio Transport

FOSIA uses WebRTC to transport live audio from the client to the server.
The client-side `WebRTC_Client` and server-side `Receiver` are framework
modules. The Hub coordinates module registration and relays the
offer/answer control message; it does not carry the actual audio stream.

The current implementation does not use a standalone HTTP
signaling server or HTTP answer polling. The client sends a
`webrtc-tunnel-offer` message through `Module.send()` and receives the
SDP answer as the reply.

## Architecture

```text
CLIENT PIPELINE
Mic module -> WebRTC_Client -> MicrophoneAudioTrack -> RTCPeerConnection
                                                        ||
                                                        || WebRTC media
                                                        \/
SERVER PIPELINE
Receiver -> PyAV resampler -> 16 kHz mono PCM bytes -> STT module
     |
     +-> diagnostic recording: received.opus
```

Offer/answer control flow:

```text
WebRTC_Client -- webrtc-tunnel-offer --> Hub handler registered by Receiver
WebRTC_Client <-- SDP answer/reply ---- Hub
```

See the [Hubdocs](https://docs.fosia.org/en/latest/technical/framework/hub/) and [Module SDKdocs](https://docs.fosia.org/en/latest/technical/framework/module-sdk/).

## Client-side input

`on_data()` reads `msg.payload` from incoming module messages. Byte
payloads are appended to an internal buffer. Each complete 640-byte
block is converted into 320 mono `int16` samples and placed in the queue
consumed by `MicrophoneAudioTrack`. `EOS` is queued as the end marker.

A full block is 320 samples × 2 bytes/sample = 640 bytes, which
represents 20 ms at 16 kHz.

## Audio frames and timestamps

`MicrophoneAudioTrack.recv()` converts a queued NumPy array into a PyAV
audio frame:

```python
frame = av.AudioFrame.from_ndarray(
    audio_data.T,
    format="s16",
    layout="mono",
)
frame.sample_rate = 16000
frame.pts = self._timestamp
frame.time_base = Fraction(1, 16000)
self._timestamp += len(audio_data)
```

The timestamp increases by the number of samples in the frame. At 16
kHz, one timestamp unit represents one sample.

## Offer/answer exchange

After the first audio block arrives, the client creates an
`RTCPeerConnection`, adds the microphone track, creates an SDP offer and
sets it as the local description. It then sends a `Message` with type
`webrtc-tunnel-offer` and the SDP/type fields through the Module SDK.

The receiver's registered handler applies the offer as its remote
description, creates an answer, sets its local description and returns
the answer data. The client validates that the reply contains `sdp` and
`type`, checks that the type is `answer`, and applies it as the remote
description.

## Server-side receiver

The receiver creates an `RTCPeerConnection` and waits for the offer
handler to establish the connection. When an audio track arrives, a
receive task repeatedly calls `track.recv()`.

For each frame, the receiver:

1. Encodes the original frame into an Ogg/Opus file named `received.opus`.
  
2. Resamples the frame with PyAV to signed 16-bit PCM, mono, 16 kHz.
  
3. Converts the resampled data to bytes.
  
4. Forwards those bytes to the configured next module when a fan-out destination exists.
  

The STT module receives PCM bytes, not the original WebRTC `AudioFrame`.

## End of stream

The microphone's `EOS` marker is turned into a `MediaStreamError` by
`MicrophoneAudioTrack`. The client observes the end of the media track
and completes its transfer loop. The receiver handles `MediaStreamError`
as a normal end of stream, flushes buffered encoder/resampler data where
possible and sends `EOS` downstream. STT uses this marker to flush its
partial byte buffer and finish the request stream.

`EOS` is a module-level marker; it is not audio sent over the WebRTC
media track.

## Connection states and cleanup

The client logs `connectionState` and `iceConnectionState`.

The receiver logs `connectionState`.

The client closes its peer connection in a `finally` block.

The receiver closes the output container during cleanup and loops for a later connection.

These logs are diagnostic only. The current code does not implement a
complete reconnect/renegotiation strategy for all failed or disconnected
states.

## Current error handling

| Situation | Behavior |
| --- | --- |
| Hub reply lacks `sdp` or `type` | Logs invalid-answer error and exits the current run path |
| Reply type is not `answer` | Logs error and exits the current run path |
| Applying remote SDP fails | Logs exception and exits the current run path |
| Audio track ends | Logs normal end of stream |
| Receive loop raises another exception | Logs exception and runs cleanup |
| No receiver fan-out destination | Logs warnings; audio is not forwarded |
| Connection becomes failed/disconnected/closed | Logs state; no complete reconnect policy |