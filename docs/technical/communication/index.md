# Communication

The communication between the FOSIA client and server is divided into two main parts: **Signaling** and **WebRTC**.

Signaling is used to establish the connection between the client and the server. WebRTC is then used to transmit the audio data.

## Communication Flow

The communication follows these basic steps:

1. **Connection setup**
   The client creates a WebRTC offer and sends it to the signaling server using `POST /offer`.

2. **Offer retrieval**
   The receiver requests the offer from the signaling server using `GET /offer`. The signaling server returns the stored offer.

3. **Answer creation**
   The receiver uses the offer to create a WebRTC answer and sends it to the signaling server using `POST /answer`.

4. **Answer retrieval**
   The client requests the answer using `GET /answer`. The signaling server returns the stored answer.

5. **WebRTC connection**
   The client uses the received answer to complete the WebRTC connection with the receiver.

6. **Audio transmission**
   Once the WebRTC connection is established, the client sends the recorded audio directly to the receiver.

7. **Audio processing**
   The receiver receives the audio and places it in the audio queue. The audio is then passed to the Speech-to-Text component for further processing.

## Complete Communication Flow

```text
┌──────────────┐        ┌──────────────────┐        ┌──────────────┐
│    Client    │        │ Signaling Server │        │   Receiver   │
└──────┬───────┘        └────────┬─────────┘        └──────┬───────┘
       │                         │                         │
       │ POST /offer             │                         │
       │────────────────────────>│                         │
       │                         │                         │
       │                         │       GET /offer        │
       │                         │<────────────────────────│
       │                         │                         │
       │                         │       Offer             │
       │                         │────────────────────────>│
       │                         │                         │
       │                         │                  Create Answer
       │                         │                         │
       │                         │       POST /answer      │
       │                         │<────────────────────────│
       │                         │                         │
       │ GET /answer             │                         │
       │────────────────────────>│                         │
       │                         │                         │
       │ Answer                  │                         │
       │<────────────────────────│                         │
       │                         │                         │
       │                         │                         │
       │<═════════════════════════════════════════════════>│
       │              WebRTC Connection                    │
       │                         │                         │
       │────────────── Audio ─────────────────────────────>│
       │                         │                         │
       │                         │                  Audio Queue
       │                         │                         │
       │                         │                         ▼
       │                         │                        STT
```

The diagram shows that the **Signaling Server is only involved in establishing the WebRTC connection**. The actual audio data does not pass through the signaling server.

After the connection has been established, the audio is transmitted directly from the **Client** to the **Receiver** using WebRTC.

## Signaling vs. WebRTC

The two components therefore have different responsibilities:

* **Signaling** – exchanges the information required to establish the connection.
* **WebRTC** – provides the real-time connection used to transmit audio.

The detailed implementation of both components is described in the following sections:

* [Signaling](signaling.md)
* [WebRTC](webrtc.md)
