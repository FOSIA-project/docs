# FOSIA – Initial Timing Analysis

## 1. Objective

This analysis evaluates the time required for FOSIA to process a spoken input and produce a final transcription.

For a voice assistant, the most important metric is the **response latency**: the time between the end of the user's speech and the availability of the final Speech-to-Text (STT) result.

The analysis is based on application logs from a test in which an Opus audio file was read by the microphone module, transmitted using WebRTC, processed by the server receiver, and passed through the Hub to Google Speech-to-Text.

The client runs in the `Europe/Vienna` time zone (CEST, UTC+02:00), whereas the server uses UTC. Therefore, server timestamps are converted to CEST before being compared with client timestamps.

All elapsed times are measured relative to the microphone start at **16:49:25.812 CEST**.

## 2. Detailed Timing Analysis

### 2.1 Microphone Start and First Audio Frame

The microphone module starts reading the audio file at 16:49:25.812. The first audio frame is received by the WebRTC client at 16:49:25.854.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| First frame received by WebRTC client | 42 ms | 0.042 s |

The first audio frame becomes available to the WebRTC client only 42 ms after the microphone module starts.

### 2.2 WebRTC Connection Setup

The WebRTC client prepares the offer, sends it to the signaling server, and waits for a valid answer.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| First frame received by WebRTC client | 42 ms | 0.042 s |
| Local description set | 5.033 s since first frame | 5.075 s |
| Offer sent to signaling server | 87 ms | 5.162 s |
| Valid answer received | 3.003 s | 8.165 s |
| Remote description set | 1 ms | 8.166 s |
| WebRTC connected | 170 ms | 8.336 s |

The connection is established 8.336 seconds after microphone start. Most of this time is spent preparing the local description and waiting for the signaling answer.

### 2.3 Audio Transfer and End of Stream

The WebRTC connection becomes active at 16:49:34.148. The microphone module finishes reading the file at 16:49:40.225, and EOS is queued at 16:49:40.226.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| WebRTC connected | —   | 8.336 s |
| Microphone finishes reading the file | 6.077 s | 14.413 s |
| Client queues EOS | 1 ms | 14.414 s |

The total time from microphone start until EOS is 14.414 seconds. Since the audio file contains no silence at the end, the EOS timestamp is a suitable approximation of the end of speech.

The microphone module reports 592 frames. The average interval calculated from the total processing time is approximately 24.3 ms per frame. This is an average derived from the logs, not a direct measurement of individual frame duration.

### 2.4 Server Audio Processing and STT

The client does **not** send an EOS marker to the server. Instead, the receiver detects the end of the audio stream when no further audio frames arrive. The receiver then signals the end of the stream to the downstream processing pipeline, allowing STT to finalize the transcription.

The receiver logs the end of the audio stream at 14:49:40.621 UTC, which corresponds to 16:49:40.621 CEST. The final STT result becomes available at 14:49:40.900 UTC, corresponding to 16:49:40.900 CEST.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| Client queues EOS | —   | 14.414 s |
| Receiver detects the end of the audio stream | 395 ms after client EOS | 14.809 s |
| Final STT result becomes available | 279 ms after receiver detects stream end | 15.088 s |

The **final response latency is 674 ms**, measured from the microphone module queuing EOS at 16:49:40.226 CEST to the final STT result becoming available at 16:49:40.900 CEST.

This interval consists of two parts:

- **395 ms** from the client queuing EOS until the server receiver detects that no more audio frames are arriving.
  
- **279 ms** from the receiver detecting the end of the stream until STT produces the final transcription.
  

The measured end-to-end time from microphone start to the final STT result is approximately **15.088 seconds**. This includes the entire audio input duration and must not be confused with the response latency after the end of speech.

### 2.5 Final Transcription

The final transcription returned by STT was:

> watch what happens if I slowly lift up the cardboard the square starts to blur and the light slowly morphs into this perfect circle and you can try this with any shape a triangular hole you still get a circle

The result was produced at 16:49:40.900 CEST, 674 ms after the microphone module queued EOS.

## 3. Key Timing Metrics

| Metric | Duration |
| --- | --- |
| Microphone start → first frame at WebRTC client | 42 ms |
| Microphone start → WebRTC connected | 8.336 s |
| Microphone start → client queues EOS | 14.414 s |
| Client queues EOS → receiver detects stream end | 395 ms |
| Receiver detects stream end → final STT result | 279 ms |
| **Client queues EOS → final STT result** | **674 ms** |
| Microphone start → final STT result | 15.088 s |

## 4. Conclusion

The initial test shows that the WebRTC connection is established 8.336 seconds after microphone start. The client finishes reading the audio file and queues EOS after 14.414 seconds.

The client does not send an EOS marker to the server. Instead, the receiver detects the end of the audio stream when no further frames arrive. After the receiver detects the stream end, STT requires another 279 ms to produce the final transcription.

The most relevant metric for a voice assistant is the response latency after the user stops speaking. In this test, the interval from the microphone module queuing EOS to the final STT result is **674 ms**. Of this time, 395 ms elapse before the receiver detects the end of the stream, followed by another 279 ms until STT returns the final transcription.

The total elapsed time from microphone start to the final STT result is approximately **15.088 seconds**. This includes the audio input duration and the connection setup, so it should not be used as the response-latency metric.

This test establishes a baseline for the current FOSIA communication and STT pipeline. Future tests can use the same timestamps to compare latency after changes to audio transport, receiver processing, or STT integration.

### 5. Network Latency Analysis

A ping test from the WSL client to the server IP address (`193.219.97.171`) returned 19 of 19 packets, with **0% packet loss**. The measured round-trip time (RTT) was **28.929 ms on average**, ranging from 17.667 ms to 46.798 ms, with a standard deviation (`mdev`) of 8.511 ms. These results indicate a stable network connection during the test. Although network latency contributes to WebRTC signaling and audio transport delays, the average RTT of approximately 29 ms is small compared with the 8.336-second connection setup time and the 674 ms end-of-speech-to-final-transcription latency measured in the timing analysis. Therefore, network latency alone is unlikely to explain the longer delays; WebRTC connection establishment, audio processing, and STT finalization are also relevant factors. Note that ping measures ICMP round-trip latency and does not directly measure WebRTC or STT processing time.