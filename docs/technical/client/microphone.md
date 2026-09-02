# Microphone

The `Microphone` component is responsible for capturing audio from the client's microphone and distributing the recorded audio data to the components that require it.

It acts as the central audio input of the FOSIA client. The captured audio is distributed to the **Wake Word Detection (WWD)** component and, when enabled, to the **WebRTC client** for transmission to the server.

## Responsibilities

The `Microphone` component has the following responsibilities:

- Capturing audio from the system microphone

- Converting the microphone input into a consistent audio format

- Splitting the audio stream into fixed-size chunks

- Providing audio data to the Wake Word Detection component

- Providing audio data to the WebRTC client when recording is enabled

- Handling the synchronization between the `sounddevice` callback thread and the asyncio event loop

- Signaling the end of a recording

The microphone itself does not perform any speech recognition or audio processing beyond the configuration required for capturing the audio.

## Audio Configuration

The microphone is configured with the following default parameters:

| Parameter   | Value       | Description                                 |
| ----------- | ----------- | ------------------------------------------- |
| Sample rate | 16,000 Hz   | Number of audio samples captured per second |
| Channels    | 1           | Mono audio                                  |
| Data type*  | `int16`     | 16-bit signed PCM samples                   |
| Block size* | 320 samples | 20 ms of audio at 16 kHz                    |

*not configurable through the class constructor.

### Sample Rate

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

## Audio Data Flow

The captured audio is distributed through two separate queues:

```text
                 ┌────────────────────┐
                 │ System Microphone  │
                 └─────────┬──────────┘
                           │
                           ▼
                 ┌────────────────────┐
                 │ sounddevice        │
                 │ InputStream        │
                 └─────────┬──────────┘
                           │
                           ▼
                    _callback()
                       │      │
             ┌─────────┘      └──────────┐
             ▼                           ▼
      WWD Queue                   Client Queue
             │                           │
             ▼                           ▼
       Wake Word Detection          WebRTC Client
```

Every captured audio block is copied and placed into the WWD queue.

The client queue is only populated while `client_enabled` is set to `True`. This allows FOSIA to continuously listen for a wake word without continuously transmitting audio to the server.

!!! warning "Audio transmission boundary"

    Audio is only transmitted to the server through the `client_queue`.
    The WebRTC client reads audio exclusively from this queue using
    `read_client()`.
    
    The microphone continuously provides audio to the `wwd_queue`, which
    is used only by the Wake Word Detection component. Audio from the
    `wwd_queue` is never accessed by the WebRTC client.
    
    The `client_queue` only receives audio while `client_enabled` is `True`.
    When the recording is stopped, `disable_client()` sets
    `client_enabled` to `False` and adds an `END-OF-STREAM` marker to the
    queue. Therefore, no newly captured audio is added to the client queue
    after the recording has ended.
    
    This separation ensures that audio captured before wake-word activation
    or after the recording has ended cannot be transmitted through the
    WebRTC client.

## Queue Architecture

The component uses two `asyncio.Queue` instances:

### WWD Queue

`wwd_queue` continuously receives microphone data.

The Wake Word Detection component can therefore consume audio independently from whether a WebRTC recording is currently active.

```python
self.wwd_queue = asyncio.Queue()
```

### Client Queue

`client_queue` receives audio only while the client recording is enabled.

```python
self.client_queue = asyncio.Queue()
```

This prevents audio from being sent to the server before the wake word has been detected.

## Client Recording Control

The WebRTC audio transmission can be enabled and disabled using:

```python
def enable_client(self):
    self.client_enabled = True
    self.client_done.clear()
```

When enabled, newly captured audio blocks are added to the client queue.

When the recording ends, the client is disabled:

```python
def disable_client(self):
    self.client_enabled = False
    self.client_done.set()
    self.client_queue.put_nowait("END-OF-STREAM")
```

The `END-OF-STREAM` marker informs the consumer that no more audio data will be added to the client queue for the current recording.

This allows the WebRTC client and subsequent components to detect the end of the audio stream without having to continuously check the `client_enabled` state.

## Thread Synchronization

One important implementation detail is that the `sounddevice` callback does **not** run inside the asyncio event loop.

The callback is executed by the audio input thread created by `sounddevice`. Directly performing asyncio operations from this thread would therefore be unsafe.

For this reason, the component stores the currently running asyncio event loop:

```python
self.loop = asyncio.get_running_loop()
```

The callback then schedules queue operations on the asyncio event loop using:

```python
self.loop.call_soon_threadsafe(...)
```

For example:

```python
self.loop.call_soon_threadsafe(
    self.wwd_queue.put_nowait,
    indata.copy(),
)
```

This ensures that queue operations are executed by the asyncio event-loop thread.

### Why `call_soon_threadsafe()` is required

The audio callback and the asyncio event loop run in different threads:

```text
sounddevice thread                 asyncio thread
       │                                  │
       │ _callback()                      │
       │                                  │
       └── call_soon_threadsafe() ───────►│
                                          │
                                          ▼
                                   asyncio.Queue
```

Using `call_soon_threadsafe()` prevents race conditions and ensures that the asyncio queues are accessed from the correct thread.

## Copying the Audio Data

The callback receives the audio data through `indata`.

The data is copied before it is passed to the queues:

```python
indata.copy()
```

This is necessary because `sounddevice` reuses the memory associated with the callback buffer. Keeping a reference to `indata` itself could therefore result in previously queued audio data being overwritten when the next audio block is captured.

Each queued block consequently owns its own copy of the audio data.

## Audio Format

The microphone uses:

```python
dtype="int16"
```

This results in signed 16-bit PCM samples.

The resulting NumPy array has the following general structure for mono audio:

```text
[ sample_1
  sample_2
  sample_3
  ...
  sample_320 ]
```

With a block size of 320 samples, each callback therefore produces approximately 20 ms of mono PCM audio.

The audio remains in this raw PCM representation inside the microphone component. Encoding for transmission is handled later by the WebRTC/Opus pipeline.

## Lifecycle

The microphone follows the following lifecycle:

```text
start()
   │
   ▼
InputStream opened
   │
   ▼
Microphone captures audio
   │
   ├──────────────► WWD Queue
   │
   └── if enabled ─► Client Queue
   │
   ▼
stop()
   │
   ▼
done event set
   │
   ▼
InputStream context exits
   │
   ▼
Microphone stopped
```

The `start()` method remains active until the `done` event is set:

```python
await self.done.wait()
```

The microphone can therefore be stopped asynchronously using:

```python
await mic.stop()
```

which sets the event:

```python
self.done.set()
```

This allows the microphone to integrate cleanly into the overall asyncio-based lifecycle of the FOSIA client.

## Design Decisions

### Continuous WWD Audio

The microphone continuously places audio data into the WWD queue, even when no command is currently being recorded.

This is required because Wake Word Detection needs access to the live microphone stream in order to detect the activation phrase.

### Separate WWD and Client Queues

Two separate queues are used instead of having the WWD and WebRTC client consume the same queue.

The WWD component needs a continuous audio stream, while the WebRTC client should only receive audio after activation.

Using separate queues allows both components to operate independently.

### Client Enable Flag

The `client_enabled` flag controls whether captured audio is forwarded to the WebRTC client.

This keeps the decision whether audio should be transmitted separate from the microphone capture itself.

The microphone therefore remains responsible only for capturing and distributing audio, while the wake-word logic controls when transmission starts.

### Asynchronous Interface

The microphone exposes asynchronous read methods:

```python
async def read_client(self):
    return await self.client_queue.get()

async def read_wwd(self):
    return await self.wwd_queue.get()
```

This allows consumers to wait for new audio without polling.

For example:

```python
audio_data = await mic.read_client()
```

The consumer is suspended until another audio block becomes available.

## Error Handling

The `sounddevice` callback receives a `status` object that can contain information about problems occurring during audio capture.

Currently, these statuses are logged:

```python
if status:
    logger.info(f"Audio status: {status}")
```

This makes problems such as input overflows or other audio-device issues visible in the logs.

## Current Limitations

The current implementation assumes that a valid microphone is available and that `sounddevice` can successfully open the configured input stream.

Device selection is currently handled by the underlying audio system rather than explicitly selecting a particular microphone.

## Interface

The main public interface of the component consists of:

| Method             | Type  | Purpose                                          |
| ------------------ | ----- | ------------------------------------------------ |
| `start()`          | Async | Starts microphone capture                        |
| `stop()`           | Async | Stops microphone capture                         |
| `read_wwd()`       | Async | Reads the next WWD audio block                   |
| `read_client()`    | Async | Reads the next client audio block                |
| `enable_client()`  | Sync  | Enables forwarding audio to the client           |
| `disable_client()` | Sync  | Stops forwarding audio and signals end of stream |

The `Microphone` component therefore provides a simple asynchronous interface to the rest of the FOSIA client while hiding the thread handling and audio-device implementation details.
