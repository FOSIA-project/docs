# FOSIA - Second Timing Analysis

## 1. Objective

This analysis evaluates the time required for FOSIA to process a spoken
input and produce a final transcription.

For a voice assistant, the most important metric is the **response
latency**: the time between the end of the user's speech and the
availability of the final Speech-to-Text (STT) result.

The analysis is based on application logs from a test in which an Opus
audio file was read by the microphone module, transmitted using WebRTC,
processed by the server receiver, and passed through the Hub to Google
Speech-to-Text.

The client runs in the `Europe/Vienna` time zone (CEST, UTC+02:00),
whereas the server uses UTC. Server timestamps are therefore converted
to CEST before being compared with client timestamps.

Compared with the initial timing analysis, the separate signaling server
has been removed. WebRTC offer/answer exchange now uses the tunnel
provided by the Hub framework. Audio is still transported using WebRTC,
and the receiver continues to pass the audio stream to STT.

All elapsed times are measured relative to the microphone module
starting at **23:06:03.112 CEST**.

## 2. Detailed Timing Analysis

### 2.1 Microphone Start and First Audio Frame

The microphone workaround starts reading the audio file at 23:06:03.112.
The first audio frame is received by the WebRTC client at 23:06:03.166.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| First frame received by WebRTC client | 54 ms | 0.054 s |

The first audio frame becomes available to the WebRTC client 54 ms after
the microphone module starts.

### 2.2 WebRTC Connection Setup

The WebRTC client creates its offer and sends it through the
framework-provided tunnel. The client receives the answer and completes
the WebRTC connection.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| First frame received by WebRTC client | 54 ms | 0.054 s |
| Local description set / offer sent | 5.035 s since first frame | 5.089 s |
| Answer received by client | 58 ms | 5.147 s |
| Remote description set | 1 ms | 5.148 s |
| WebRTC connected | 409 ms | 5.557 s |

The client reports the connection as established at 23:06:08.669 CEST, 5.557 seconds after microphone start. Most of this setup time occurs before the local description is set: the client creates the offer and completes its local setup over approximately five seconds.

The client receives the tunnel response at 23:06:08.259 CEST. The server
log records receipt of the offer at 23:06:08.482 CEST after converting
its UTC timestamp. This ordering is inconsistent with perfectly
synchronized clocks, so cross-machine timestamps should be treated as
approximate. The elapsed durations measured within each process are more
reliable.

### 2.3 Audio Transfer and End of Stream

The client reports WebRTC connected at 23:06:08.669 CEST. The microphone
module finishes reading the file and queues the end-of-stream (EOS)
marker at 23:06:15.737--23:06:15.738 CEST.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| WebRTC connected | —   | 5.557 s |
| Microphone finishes reading the file | 7.068 s | 12.625 s |
| Client queues EOS | 1 ms | 12.626 s |

The total time from microphone start until EOS is approximately 12.626
seconds. The microphone workaround reports 592 frames. This is a
file-based test, so the timing includes the startup and connection setup
before the audio transfer finishes.

### 2.4 Server Audio Processing and STT

The client queues EOS at 23:06:15.738 CEST. The receiver detects that
the audio stream has ended at 21:06:15.952 UTC, equivalent to
23:06:15.952 CEST. STT logs EOS at 21:06:15.954 UTC and produces the
final transcription at 21:06:16.209 UTC.

| Event | Step duration | Total elapsed time |
| --- | --- | --- |
| Microphone starts | —   | 0.000 s |
| Client queues EOS | —   | 12.626 s |
| Receiver detects the end of the audio stream | Approx. 214 ms after client EOS | 12.840 s |
| STT receives EOS | 2 ms after receiver stream-end event | 12.842 s |
| Final STT result becomes available | 255 ms after STT EOS | 13.097 s |

Using the timestamps as recorded, the **apparent response latency is approximately 471 ms**, measured from the client queuing EOS to the final STT result. This consists of approximately 214 ms until the receiver detects the stream end, followed by approximately 257 ms from receiver stream-end detection to the final result (including the 2 ms between the receiver and STT EOS logs).
logs show a small clock-alignment inconsistency, 471 ms should be
treated as an approximate end-to-end figure rather than a precision
measurement. The server-side interval from receiver stream-end detection
at 23:06:15.952 to the final STT result at 23:06:16.209 is 257 ms.

The total elapsed time from microphone start to the final STT result is
approximately **13.097 seconds**, based on the client start time and
converted server final-result timestamp. This combines audio playback,
connection setup, and finalization; it is not the response latency after
speech ends.

### 2.5 Final Transcription

The final transcription returned by STT was:

> watch what happens if I slowly lift up the cardboard the square starts
> to blur and the light slowly morphs into this perfect circle and you
> can try this with any shape a triangular hole you still get a circle

The final result is logged at 21:06:16.209 UTC, corresponding to
23:06:16.209 CEST.

## 3. Key Timing Metrics

| Metric | Duration |
| --- | --- |
| Microphone start → first frame at WebRTC client | 54 ms |
| Microphone start → WebRTC connected | 5.557 s |
| Microphone start → client queues EOS | 12.626 s |
| Client queues EOS → receiver detects stream end (timestamp-based estimate) | Approx. 214 ms |
| Receiver stream-end event → final STT result | 257 ms |
| **Client queues EOS → final STT result (apparent end-to-end)** | **Approx. 471 ms** |
| Microphone start → final STT result | Approx. 13.097 s |

## 4. Conclusion

This test shows that the WebRTC connection is established 5.557 seconds
after microphone start, compared with 8.336 seconds in the initial
timing analysis. The client finishes reading the audio file and queues
EOS after approximately 12.626 seconds.

The separate signaling server is no longer used. Instead, the Hub
framework's tunnel carries the WebRTC offer/answer exchange. The logs
show that this setup completes successfully and that the receiver
receives the offer, establishes the peer connection, and processes 587
audio frames (588 packets). The microphone module reports 592 frames;
these counts come from different stages and should not be expected to
match exactly.

The apparent end-of-speech-to-final-transcription latency is
approximately 471 ms, compared with 674 ms in the initial analysis. This
suggests an improvement of approximately 203 ms, or about 30%. However,
the client and server timestamps show a clock-alignment inconsistency,
so the end-to-end comparison should be treated as indicative. The
server-side interval from stream-end detection to final STT output is
257 ms.

The total time from microphone start to the final STT result is
approximately 13.097 seconds. This is lower than the initial result of
15.088 seconds, but it includes audio duration and connection setup and
should not be used as the primary response-latency metric.

Overall, the measurements indicate improved connection setup and a lower
apparent finalization latency. A repeat test with synchronized
client/server clocks and identical test conditions would make the
comparison more conclusive.

## 5. Changes Since the Initial Analysis and Latency Comparison

| Metric | Initial analysis | Second analysis | Difference |
| --- | --- | --- | --- |
| Microphone start → first frame | 42 ms | 54 ms | 12 ms slower |
| Microphone start → WebRTC connected | 8.336 s | 5.557 s | 2.779 s faster |
| Microphone start → client queues EOS | 14.414 s | 12.626 s | 1.788 s shorter |
| Client EOS → final STT result | 674 ms | Approx. 471 ms | Approx. 203 ms faster |
| Receiver stream-end → final STT result | 279 ms | 257 ms | 22 ms faster |
| Microphone start → final STT result | 15.088 s | Approx. 13.097 s | Approx. 1.991 s shorter |

The architecture has changed in one important respect: the dedicated signaling server has been replaced by the Hub framework's tunnel. The client now sends a `webrtc-tunnel-offer` through the Hub, and the receiver returns the answer through the same framework-managed mechanism. This removes the separate signaling component from the application path.

The measured WebRTC setup time improved by approximately 2.779 seconds
(about 33%). The apparent EOS-to-final-STT latency improved by
approximately 203 ms (about 30%). The initial-frame delay increased by
12 ms, which is small in absolute terms. The shorter total time to the
final transcript also reflects a shorter measured file-transfer interval
and should not be attributed entirely to the tunnel change.

These are comparisons between two individual runs, not controlled
averages. The logs do not provide a new ping/RTT test, so no conclusion
can be drawn about whether underlying network RTT improved or worsened.
The client/server clock discrepancy also limits the precision of
cross-machine latency calculations.