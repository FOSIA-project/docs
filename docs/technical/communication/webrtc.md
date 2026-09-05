# WebRTC

FOSIA uses **WebRTC** to transmit audio from the client to the server.

The WebRTC connection is established after the required session information has been exchanged through the [Signaling](signaling.md) component.

Unlike the signaling connection, which uses HTTP, the WebRTC connection is responsible for the actual real-time audio transmission.

## Architecture

The audio transmission follows this general path:

```text
System Microphone
       │
       ▼
 Microphone
       │
       ▼
 Client Queue
       │
       ▼
MicrophoneAudioTrack
       │
       ▼
RTCPeerConnection
       │
       ▼
     WebRTC
       │
       ▼
RTCPeerConnection
       │
       ▼
    Receiver
       │
       ├──────────────► Audio Queue
       │
       ▼
  Audio Processing
```

The WebRTC layer therefore connects the client-side audio source with the server-side audio receiver.

## Peer Connection

Both the client and receiver create an `RTCPeerConnection`.

On the client:

```python
pc = RTCPeerConnection()
```

On the receiver:

```python
pc = RTCPeerConnection()
```

The two peer connections represent the two endpoints of the WebRTC connection.

The signaling server is only involved during connection establishment.

## Client Audio Track

The client provides microphone data to WebRTC through a custom `MediaStreamTrack` implementation called `MicrophoneAudioTrack`.

```python
class MicrophoneAudioTrack(MediaStreamTrack):
    kind = "audio"
```

The track obtains audio data from the microphone's client queue:

```python
audio_data = await self.mic.read_client()
```

This is an important boundary in the audio architecture.

Only audio placed in the `client_queue` can reach the WebRTC audio track. The WWD queue is not accessed by the WebRTC client.

## Converting Audio to an AudioFrame

The microphone provides audio as NumPy arrays containing signed 16-bit PCM samples.

Before the data can be processed by aiortc, it is converted into an `av.AudioFrame`:

```python
frame = av.AudioFrame.from_ndarray(
    audio_data.T,
    format="s16",
    layout="mono",
)
```

The sample rate is then assigned:

```python
frame.sample_rate = self.mic.SAMPLE_RATE
```

The resulting frame represents a mono PCM audio frame that can be consumed by the WebRTC pipeline.

## Timestamps

Each audio frame receives a presentation timestamp (`pts`).

The initial timestamp is:

```python
self._timestamp = 0
```

After each frame, the timestamp is increased by the number of samples contained in that frame:

```python
self._timestamp += len(audio_data)
```

The time base is configured according to the microphone sampling rate:

```python
frame.time_base = Fraction(1, self.mic.SAMPLE_RATE)
```

At 16 kHz, one timestamp unit therefore represents one audio sample.

This allows aiortc to determine the temporal position of each audio frame in the continuous audio stream.

## Audio Frame Duration

The microphone currently produces blocks of 320 samples at a sample rate of 16 kHz.

```text
320 samples / 16000 samples/s = 0.020 s = 20 ms
```

Therefore, each microphone callback normally provides approximately 20 ms of audio.

The 20-ms block size is compatible with one of the standard frame durations supported by the Opus codec and provides a suitable granularity for the WebRTC audio pipeline.<sup>[[1]](https://www.rfc-editor.org/info/rfc6716/)</sup>


During testing, other microphone block sizes resulted in WAV files on the server that were significantly distorted and difficult to understand. Therefore, the microphone currently uses a fixed block size of 320 samples.

[1] : https://www.rfc-editor.org/info/rfc6716/
## Adding the Audio Track

The microphone track is wrapped by `LoggingAudioTrack`:

```python
microphone_track = MicrophoneAudioTrack(mic)

pc.addTrack(
    LoggingAudioTrack(microphone_track, done)
)
```

`LoggingAudioTrack` forwards the frames received from the microphone track while additionally monitoring the stream for its termination.

Every 100 forwarded frames, a log message is generated:

```python
if self._frame_count % 100 == 0:
    logger.info(
        f"Audio frame #{self._frame_count} forwarded"
    )
```

This provides a simple way to monitor whether audio frames are continuously being transmitted.

## WebRTC Connection Establishment

The connection establishment consists of several steps.

First, the client creates an SDP offer:

```python
offer = await pc.createOffer()
await pc.setLocalDescription(offer)
```

The offer is exchanged through the signaling server.

The receiver applies the offer and creates an answer:

```python
await pc.setRemoteDescription(
    RTCSessionDescription(
        sdp=offer["sdp"],
        type=offer["type"]
    )
)

answer = await pc.createAnswer()
await pc.setLocalDescription(answer)
```

The client then applies the received answer:

```python
await pc.setRemoteDescription(
    RTCSessionDescription(
        sdp=data["sdp"],
        type=data["type"]
    )
)
```

After this exchange, the WebRTC peers can establish the media connection.

The complete signaling process is documented in [Signaling](signaling.md).

## Audio Transmission

Once the connection is established, aiortc repeatedly calls the `recv()` method of the microphone audio track whenever another audio frame is required.

The track obtains the next block from the client queue:

```python
audio_data = await self.mic.read_client()
```

It then converts the block into an `AudioFrame` and returns it to aiortc.

Conceptually, the process is:

```text
client_queue
     │
     ▼
read_client()
     │
     ▼
NumPy int16 PCM
     │
     ▼
av.AudioFrame
     │
     ▼
MediaStreamTrack
     │
     ▼
aiortc
     │
     ▼
WebRTC audio transport
```

The encoding and transport of the audio are handled by the WebRTC/aiortc stack.

## Server-Side Receiver

On the server, the receiver listens for incoming media tracks:

```python
@pc.on("track")
def on_track(track):
```

When the audio track is connected, a receive task is created:

```python
task = asyncio.create_task(recv())
```

The receive loop continuously obtains frames:

```python
while True:
    frame = await track.recv()
```

Each received frame is then made available to the rest of the server through the audio queue:

```python
if audio_queue is not None:
    await audio_queue.put(frame)
```

This separates WebRTC from the subsequent audio-processing components.

The receiver is responsible for receiving the WebRTC frames, while components such as STT can consume the frames from the audio queue independently.

## Server-Side Audio Storage

For testing and debugging purposes, the receiver additionally writes the received audio to an output file.

An Ogg container with an Opus audio stream is created:

```python
container = av.open(
    "received.opus",
    mode="w",
    format="ogg"
)

stream = container.add_stream(
    "libopus",
    rate=48000
)
```

Each received frame is passed to the encoder:

```python
for packet in stream.encode(frame):
    container.mux(packet)
```

The resulting packets are written to the output container.

This allows the received WebRTC audio to be inspected independently from the STT processing pipeline.

The output file is primarily a development and testing mechanism and is not required for the normal WebRTC data flow.

## End of Stream

The client queue uses an `END-OF-STREAM` marker to signal that no more audio data is available for the current recording.

The microphone inserts the marker when the client recording is disabled:

```python
self.client_queue.put_nowait("END-OF-STREAM")
```

`MicrophoneAudioTrack` detects the marker:

```python
if isinstance(audio_data, str) and audio_data == "END-OF-STREAM":
    logger.info("EOS detected, stopping audio stream...")
    raise MediaStreamError
```

The `MediaStreamError` indicates to aiortc that the media stream has ended.

The `LoggingAudioTrack` also handles the resulting stream termination:

```python
except MediaStreamError:
    logger.info(
        "FRAME TRANSFER ENDED: no more audio frames"
    )
    self.done.set()
    raise
```

The client can then wait for the `done` event:

```python
await done.wait()
```

and subsequently close the peer connection:

```python
await pc.close()
```

## Server-Side Stream Termination

The receiver handles the end of the incoming media stream:

```python
except MediaStreamError:
    logger.info(
        "Audio stream ended (no more frames)"
    )
```

The receive task sets the `done` event in its `finally` block:

```python
finally:
    done.set()
```

The receiver can therefore detect that the incoming media stream has ended and perform its cleanup operations.

The server also places an `END-OF-STREAM` marker into the audio queue so that downstream components such as STT can detect the end of the current audio stream.

## Connection State Monitoring

Both sides monitor the WebRTC connection state.

The client registers a handler for:

```python
@pc.on("connectionstatechange")
```

and logs states such as:

```text
new
connecting
connected
disconnected
failed
closed
```

The ICE connection state is monitored separately:

```python
@pc.on("iceconnectionstatechange")
```

On the receiver, the connection state is also monitored:

```python
@pc.on("connectionstatechange")
```

If the connection reaches a terminal or failed state, the receiver sets its `done` event:

```python
if pc.connectionState in (
    "failed",
    "closed",
    "disconnected"
):
    done.set()
```

This allows the receiving task to terminate when the WebRTC connection is no longer usable.

## Audio Transmission Boundary

The microphone architecture creates a deliberate boundary between continuous wake-word detection and network transmission.

```text
                   Microphone
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
         WWD Queue        Client Queue
              │                 │
              ▼                 ▼
             WWD            WebRTC
                                │
                                ▼
                              Server
```

The WWD queue continuously receives microphone audio.

The client queue, however, only receives audio while `client_enabled` is enabled.

Consequently, the WebRTC audio track can only access audio through the client queue. Audio captured before activation is therefore not made available to the WebRTC track.

When the recording is disabled, `client_enabled` is set to `False`, preventing newly captured audio from being added to the client queue.
!!! note
    This architecture separates **local continuous wake-word detection** from **network audio transmission**.

## Lifecycle

The simplified WebRTC lifecycle is:

```text
Create RTCPeerConnection
          │
          ▼
Create microphone AudioTrack
          │
          ▼
Add track to PeerConnection
          │
          ▼
Create SDP Offer
          │
          ▼
Signaling exchange
          │
          ▼
Set Remote Description
          │
          ▼
WebRTC connection established
          │
          ▼
Receive / transmit audio frames
          │
          ▼
END-OF-STREAM or connection failure
          │
          ▼
Close PeerConnection
```

The signaling exchange is therefore part of the connection setup, while audio transmission occurs through the established WebRTC connection.

## Separation from STT

The WebRTC receiver does not perform speech recognition itself.

Instead, received `AudioFrame` objects are placed into the server-side audio queue:

```python
await audio_queue.put(frame)
```

The STT component can then consume these frames independently.

This creates a clear separation between:

```text
WebRTC
   │
   ▼
AudioFrame
   │
   ▼
Audio Queue
   │
   ▼
STT
   │
   ▼
Transcript
```

This separation allows the WebRTC transport layer and the speech-recognition implementation to be changed independently.

## Current Limitations

The current WebRTC implementation is primarily designed for the FOSIA prototype.

Current limitations include:

* The signaling server supports only one active offer and answer.
* The signaling process uses polling.
* WebRTC connection timeouts are not yet handled explicitly.
* The current implementation focuses on a single audio track.
* The server writes a local Opus file for testing purposes.
* Connection and stream errors are logged, but more detailed error handling is planned.

These limitations are addressed progressively as part of the FOSIA development roadmap.
