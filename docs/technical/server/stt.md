# Speech-to-Text (Google)

The `STT` module converts audio received from the server-side `Receiver`
into text using Google Cloud Speech-to-Text's asynchronous streaming
API. It inherits from `module_sdk.Module`; the Hub manages registration
and module routing.

## Pipeline position

```text
Client microphone -> WebRTC_Client
                          |
                          | WebRTC media
                          v
                       Receiver
                          |
                          | 16 kHz mono signed 16-bit PCM bytes
                          v
                       STT module
                          |
                          v
              Google Cloud Speech-to-Text
                          |
                          v
               Interim / final results
```

The Hub is the control plane. Audio payloads travel between modules over
the framework data path, not through the Hub control connection. See the
[Hub docs](https://docs.fosia.org/en/latest/technical/framework/hub/) and [Writing a module](https://docs.fosia.org/en/latest/technical/framework/module-sdk/writing-a-module/).

## Input format

| Parameter | Value |
| --- | --- |
| Encoding | Signed 16-bit PCM / LINEAR16 |
| Sample rate | 16,000 Hz |
| Channels | Mono |
| Full chunk size | 640 bytes |
| Duration per full chunk | 20 ms |

The receiver converts WebRTC frames to this format with PyAV's

`AudioResampler` before forwarding the bytes.

## Buffering and request generation

`on_data()` reads `msg.payload`. Incoming byte payloads are appended to
`audio_buffer`. Complete 640-byte chunks are removed from the buffer and
placed in `audio_queue`, allowing arbitrary input message boundaries
without changing the chunk size sent to Google.

When `EOS` arrives, any remaining bytes are queued as a final partial
chunk, the buffer is cleared, and `EOS` is added to the queue.
`_audio_frames_from_queue()` reads chunks until it sees that marker.
`_request_generator()` sends the streaming configuration as the first
`StreamingRecognizeRequest`, followed by requests containing the audio
bytes.

## Google API configuration

The module creates a `speech_v1.SpeechAsyncClient` and uses the
following configuration:

```python
speech_v1.RecognitionConfig(
    encoding=speech_v1.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz=16000,
    language_code="en-US",
)

speech_v1.StreamingRecognitionConfig(
    config=self.stt_config,
    interim_results=interim_results,
)
```

The language is currently hard-coded to US English (`en-US`).
`interim_results` defaults to `True`. Google Cloud credentials must be
available to the runtime; this module does not provision them.

## Results

- **Interim results** are provisional and logged when enabled.
- **Final results** are logged and appended to `final_transcripts`.

When the response stream ends, `on_run()` returns the list of final
transcript segments. The current implementation does not join them into
a single string or forward them to a downstream decision module.

## End of stream

`EOS` is a FOSIA module-level marker, not part of the audio sent to
Google. On receiving it, the module flushes any partial PCM bytes and
queues the marker. The queue reader then ends, which terminates the
audio request generator after the final chunk.

## Test mode and limitations

The source includes an optional `stt_test` mode intended to read
`audio-10s.opus`. In the current code, this path is not wired
consistently: - `_audio_frames_from_file()` yields PyAV `AudioFrame`
objects, while `_request_generator()` expects chunks suitable for
Google's `audio_content` field. - `on_run()` still waits for
`audio_queue` to become non-empty even when file mode is selected, so it
may wait indefinitely if no live data arrives.

Treat file-based testing as unfinished until these paths are reconciled.
The live pipeline uses PCM byte payloads and `EOS`.

## Error handling

| Situation | Behavior |
| --- | --- |
| Byte payload arrives | Buffering/chunking occurs in`on_data()` |
| `EOS` arrives | Flushes partial buffer and queues<br/> `EOS` |
| Interim result arrives | Logs it when enabled |
| Final result arrives | Logs and collects the transcript segment |
| Test file is missing | Raises `FileNotFoundError` |
| Google client/API raises error propagates through the module runtime | No dedicated retry/recovery policy |

Current limitations: language is fixed to `en-US`; no retry/backoff is
implemented; final segments are collected but not joined or forwarded;
file-based test mode needs correction.