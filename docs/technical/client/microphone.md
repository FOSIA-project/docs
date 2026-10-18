# Microphone

The `Mic` module is the client-side audio source in the FOSIA module
pipeline. It captures microphone audio, creates fixed-size PCM chunks
and forwards them through the Module SDK. The Hub handles module
registration and routing; it does not carry the audio payload itself.

## Responsibilities

- Capture mono audio at 16 kHz using `sounddevice` / PortAudio.
- Produce 320-sample blocks (20 ms at 16 kHz).
- Safely bridge the audio callback thread to the asyncio event loop.
- Fall back to PulseAudio-compatible capture through `parec` if
  PortAudio fails during startup.
- Forward frames to configured fan-out destinations.
- Send `EOS` on shutdown.
- Optionally save captured samples to `recording.wav` for diagnostics.

## Pipeline position

```text
Microphone -> sounddevice / PortAudio
                   | startup failure
                   +----> parec / PulseAudio
                                |
                                v
                         320-sample PCM blocks
                                |
                                v
                            Mic module
                                |
                                v
                       Module SDK fan-out
                                |
                                v
                       WebRTC client module
```

See the [Hub
documentation](https://docs.fosia.org/en/latest/technical/framework/hub/)
and [Module SDK
documentation](https://docs.fosia.org/en/latest/technical/framework/module-sdk/).

## Audio format

| Parameter | Value |
| --- | --- |
| Sample rate | 16,000 Hz |
| Channels | 1 (mono) |
| Sample format | Signed 16-bit PCM (`int16`) |
| Block size | 320 samples |
| Duration per block | 20 ms |
| Bytes per full block | 640 bytes |

`320 / 16000 = 0.02 s`. Each `int16` sample uses two bytes, so a full
mono block contains 640 bytes.

### Sample Rate

FOSIA captures audio at **16 kHz**. This sample rate is sufficient for speech recognition while keeping the amount of data that needs to be processed and transmitted relatively low.<sup>[[1]](https://docs.cloud.google.com/speech-to-text/docs/v1/best-practices?hl=en)</sup>

At 16,000 samples per second, one second of mono audio consists of:

`16,000 samples`

[1] : https://docs.cloud.google.com/speech-to-text/docs/v1/best-practices?hl=en

### Block Size

The microphone captures audio in blocks of **320 samples**.

At a sample rate of 16 kHz, this corresponds to:

`320 / 16,000 = 0.02 s = 20 ms`

Therefore, every callback from `sounddevice` normally provides approximately **20 ms of audio data**.

The 20 ms block size was selected because it corresponds to one of the typical audio frame duration used by the Opus codec and therefore provides a suitable input granularity for the WebRTC audio pipeline.<sup>[[2]](https://www.rfc-editor.org/info/rfc6716/)</sup>

During testing, other block sizes resulted in WAV files on the server that were significantly distorted and difficult to understand. Therefore, the block size is currently fixed to 320 samples.

[2] : https://www.rfc-editor.org/info/rfc6716/

## PortAudio capture and thread synchronization

The module opens `sounddevice.InputStream` with the format above. Its
callback copies the input buffer because PortAudio may reuse it after
the callback returns. Since the callback can run on a separate thread,
it uses `loop.call_soon_threadsafe()` to schedule the queue write on the
asyncio event loop.

## PulseAudio fallback

If `sd.InputStream` raises `sd.PortAudioError` before recording has
started, the module looks for `parec` on `PATH` and starts it as a
subprocess. It requests raw signed 16-bit little-endian audio:

```bash
parec --raw --format=s16le --rate=16000 --channels=1
```

The module reads 640 bytes at a time and decodes each block into a NumPy
array of shape `(320, 1)`.

`parec` must be installed and a default PulseAudio source must be
available. On Debian/Ubuntu/WSL it is commonly supplied by:

```bash
sudo apt install pulseaudio-utils
```

To check whether WSL sees an audio source:

```bash
pactl list short sources
```

The fallback is only attempted when PortAudio fails during startup. A
PortAudio error after recording has started is re-raised; the code does
not automatically switch backends mid-stream.

## Module lifecycle

- `on_init()` creates the capture object and initializes task state.
- `on_ready()` starts capture and waits for either the `started` event
  or a startup failure. After capture starts, it launches
  `_forward_audio()`.
- `_forward_audio()` reads frames from the local queue and calls the
  Module SDK fan-out method. Missing destinations produce warnings.
- `stop()` signals capture shutdown, terminates `parec` if active, and
  queues `EOS`.
- `on_stop()` waits for the tasks and, if enabled, writes
  `recording.wav`.

The module instance currently enables WAV recording with
`Mic(save_wavefile=True)`. Use `False` when diagnostic recording is not
required. The file is written to the process working directory.

## Current error handling

| Situation | Behavior |
| --- | --- |
| PortAudio fails during startup | Attempts `parec` fallback |
| `parec` is missing | Raises a descriptive `RuntimeError` |
| PulseAudio fails before the first full block | Reports stderr or process exit code in a `RuntimeError` |
| PulseAudio exits during capture | Raises a `RuntimeError` |
| Callback reports a status condition | Logs the status |
| No fan-out destinations | Logs warnings; audio cannot be forwarded |
| Normal shutdown | Queues `EOS` and waits for capture/forwarding tasks |