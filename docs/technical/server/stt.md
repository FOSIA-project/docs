# Speech-to-Text

The Speech-to-Text (STT) component converts the audio received from the FOSIA client into text. It acts as the interface between the audio processing part of the server and the later decision-making components.

FOSIA currently uses the **Google Cloud Speech-to-Text API** for speech recognition.

## Processing Flow

The STT component receives audio frames from the receiver and processes them in several steps:

```text
WebRTC AudioFrame
       │
       ▼
 Audio Queue
       │
       ▼
 AudioFrame
       │
       ▼
 Resampling
       │
       ▼
16 kHz / Mono / LINEAR16
       │
       ▼
 Google Speech-to-Text
       │
       ▼
 Speech Recognition Results
       │
       ▼
 Final Transcript
```

The individual processing steps are described below.

## Audio Input

The STT component receives audio frames as `av.AudioFrame` objects. These frames are produced by the WebRTC connection and placed into an `asyncio.Queue` by the receiver.

The STT component reads the frames from this queue asynchronously. This separates the reception of audio from its processing and allows both components to operate independently.

For testing purposes, the STT component can also read audio frames directly from an Opus file instead of using the audio queue.

## Audio Conversion

The audio received from WebRTC is not sent directly to Google Speech-to-Text. First, the audio frames are converted into a format supported by the API.

An `AudioResampler` from PyAV is used to convert the frames to:

* **Sample rate:** 16,000 Hz
* **Channels:** Mono
* **Format:** Signed 16-bit PCM (`s16` / LINEAR16)

The resulting PCM data is converted into raw bytes before being sent to the Speech-to-Text API.

The resampler may internally buffer audio data. When the end of the audio stream is reached, the resampler is therefore flushed to ensure that any remaining audio is also processed.

## Streaming Recognition

FOSIA uses Google's **streaming speech recognition** interface. Instead of waiting until the complete recording is available, audio data is continuously sent to the API while it is being received.

The first request contains the streaming configuration. Subsequent requests contain the actual PCM audio data.

This allows the Speech-to-Text component to receive recognition results while the user is still speaking.

## Interim and Final Results

Google Speech-to-Text can return two types of recognition results:

* **Interim results** are temporary results that may still change as more audio is received.
* **Final results** are completed parts of the transcription that will no longer be changed.

Interim results are mainly used for logging and monitoring. Only final results are stored by the STT component.

The final results are collected in a list and returned to the calling component after the audio stream has ended.

For example, a spoken input may produce several final results:

```text
"turn on"
"the living room"
"light"
```

These results can later be combined into the complete transcript:

```text
"turn on the living room light"
```

## End of Stream

The end of the audio stream is indicated using an **End-of-Stream (EOS)** marker.

The receiver places the EOS marker into the audio queue after the WebRTC audio transmission has ended. The STT component detects this marker and stops reading from the queue.

Before finishing, the audio resampler is flushed to make sure that no buffered audio data is lost.

After all final recognition results have been received, the STT component returns them to the calling component.

## Interface

The main entry point of the STT component is the `run()` function:

```python
async def run(audio_queue=None, interim_results=True):
```

The `audio_queue` parameter determines the source of the audio:

* If a queue is provided, audio is received from the live WebRTC stream.
* If no queue is provided, an audio file is used instead.

The file-based input is mainly intended for testing and development.

The `interim_results` parameter controls whether interim recognition results are requested and logged.

The component returns a list containing all final transcript segments:

```python
final_transcripts
```

This keeps the STT implementation independent from the later decision-making logic.

## Current Architecture

The current audio processing path is therefore:

```text
Microphone
    │
    ▼
Client
    │
    │ WebRTC
    ▼
Receiver
    │
    ▼
Audio Queue
    │
    ▼
STT
    │
    ▼
Final Transcript
    │
    ▼
Decision Model
```

The **Decision Model is not part of the STT component**. STT only converts the received speech into text and provides the resulting transcript to the next component.

## Current Limitations

The current implementation uses Google Cloud Speech-to-Text and therefore requires access to the corresponding external service.

The recognition configuration currently uses the `en-US` language. Support for additional languages can be added by changing the recognition configuration.

The streaming recognition is currently designed for a single continuous audio stream. More advanced handling of multiple simultaneous clients will be addressed as the server architecture is expanded.
