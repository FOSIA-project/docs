# Signaling

The **Signaling** component is responsible for exchanging the session information required to establish a WebRTC connection between the FOSIA client and server.

Signaling is only used during the **connection establishment phase**. The actual audio data is transmitted afterwards through the [WebRTC](webrtc.md) connection and does not pass through the signaling server.

## Architecture

The signaling server provides a simple HTTP-based interface for exchanging the WebRTC **SDP offer** and **SDP answer**.

```text
┌──────────────┐                         ┌──────────────────┐
│    Client    │                         │ Signaling Server │
│              │                         │                  │
│ RTCPeerConn. │                         │     aiohttp      │
└──────┬───────┘                         └────────┬─────────┘
       │                                          │
       │  POST /offer                             │
       │─────────────────────────────────────────►│
       │                                          │
       │  GET /answer                             │
       │─────────────────────────────────────────►│
       │                                          │
       │  SDP Answer                              │
       │◄─────────────────────────────────────────│
       │                                          │
       ▼                                          │
   WebRTC connection                              │
```

The server stores the offer and answer in memory until they are requested by the other component.

## Signaling Server

The signaling server is implemented using `aiohttp`.

It listens on:

```text
http://127.0.0.1:8080
```

The following HTTP endpoints are provided:

| Method | Endpoint  | Purpose                                   |
| ------ | --------- | ----------------------------------------- |
| `POST` | `/offer`  | Receives an SDP offer from the client     |
| `GET`  | `/offer`  | Provides the stored offer to the receiver |
| `POST` | `/answer` | Receives an SDP answer from the receiver  |
| `GET`  | `/answer` | Provides the stored answer to the client  |

The server therefore acts as an intermediary for the exchange of the WebRTC session descriptions.

## Offer Exchange

The connection establishment starts on the client.

The client creates an `RTCPeerConnection` and adds its audio track:

```python
pc = RTCPeerConnection()

microphone_track = MicrophoneAudioTrack(mic)
pc.addTrack(LoggingAudioTrack(microphone_track, done))
```

The client then creates an SDP offer:

```python
offer = await pc.createOffer()
await pc.setLocalDescription(offer)
```

The resulting SDP is sent to the signaling server using an HTTP POST request:

```python
requests.post(SIGNALING + "/offer", json={
    "sdp": pc.localDescription.sdp,
    "type": pc.localDescription.type
})
```

The signaling server stores the received offer:

```python
offers["offer"] = data
```

The server does not modify the SDP. It only stores and forwards it.

## Receiver Polling

The receiver does not receive the offer through a persistent connection. Instead, it periodically requests the current offer from the signaling server.

```python
while True:
    async with session.get(SIGNALING + "/offer") as response:
        offer = await response.json()

    if offer:
        break

    await asyncio.sleep(1)
```

The receiver therefore continues polling until an offer becomes available.

Once the offer has been received, it is applied as the remote description:

```python
await pc.setRemoteDescription(
    RTCSessionDescription(
        sdp=offer["sdp"],
        type=offer["type"]
    )
)
```

## Answer Exchange

After receiving and applying the offer, the receiver creates an SDP answer:

```python
answer = await pc.createAnswer()
await pc.setLocalDescription(answer)
```

The answer is then sent to the signaling server:

```python
session.post(
    SIGNALING + "/answer",
    json={
        "sdp": pc.localDescription.sdp,
        "type": pc.localDescription.type,
    }
)
```

The signaling server stores the answer:

```python
answers["answer"] = data
```

The client periodically requests the answer:

```python
while True:
    r = requests.get(SIGNALING + "/answer")
    data = r.json()

    if data and "sdp" in data:
        break

    await asyncio.sleep(1)
```

Once the answer has been received, the client applies it as its remote description:

```python
await pc.setRemoteDescription(
    RTCSessionDescription(
        sdp=data["sdp"],
        type=data["type"]
    )
)
```

At this point, the signaling exchange required by the current implementation is complete.

## Complete Signaling Sequence

The complete sequence can be summarized as follows:

```text
Client                 Signaling Server                 Receiver
  │                           │                            │
  │ createOffer()             │                            │
  │ setLocalDescription()     │                            │
  │                           │                            │
  │ POST /offer               │                            │
  │──────────────────────────►│                            │
  │                           │ store offer                │
  │                           │                            │
  │                           │◄──── GET /offer ──────────│
  │                           │                            │
  │                           │────── Offer ─────────────►│
  │                           │                            │
  │                           │                    setRemoteDescription()
  │                           │                    createAnswer()
  │                           │                    setLocalDescription()
  │                           │                            │
  │                           │◄──── POST /answer ─────────│
  │                           │ store answer               │
  │                           │                            │
  │ GET /answer               │                            │
  │──────────────────────────►│                            │
  │                           │                            │
  │◄──── Answer ──────────────│                            │
  │                           │                            │
  │ setRemoteDescription()    │                            │
  │                           │                            │
  ▼                           ▼                            ▼
                   WebRTC connection established
```

## Separation from Audio Transmission

The signaling server does **not** transmit the microphone audio.

Its purpose is limited to exchanging the session descriptions needed for WebRTC.

After the client has received the answer and applied it as its remote description, the audio is transmitted through the established WebRTC connection:

```text
             Signaling
       ┌───────────────────┐
       │                   │
       ▼                   ▼
    Client              Receiver
       │                   │
       └────── WebRTC ─────┘
              Audio
```

This separation allows the signaling mechanism to remain relatively simple while WebRTC handles the actual media transport.

## Data Storage

The current signaling implementation stores offers and answers in Python dictionaries:

```python
offers = {}
answers = {}
```

For example:

```python
offers["offer"] = data
answers["answer"] = data
```

The data is therefore held only in the memory of the running signaling server.

The current implementation does not use a database or persistent storage for signaling information.

## Polling

Both the client and receiver currently use polling to retrieve the session information.

The client polls:

```text
GET /answer
```

while the receiver polls:

```text
GET /offer
```

with a one-second delay between requests.

This approach was selected because it provides a simple implementation for the current single-client prototype without requiring a persistent signaling connection.

## Current Limitations

The current signaling implementation is designed for the FOSIA prototype and has several limitations.

### Single Connection

Offers and answers are stored under fixed dictionary keys:

```python
offers["offer"]
answers["answer"]
```

Consequently, the current implementation does not distinguish between multiple simultaneous clients.

A production implementation supporting multiple clients would require connection-specific identifiers and separate signaling state.

### No Authentication

The signaling endpoints currently do not authenticate clients or receivers.

The signaling server is therefore intended for controlled environments during the current development stage.

### No Persistent Storage

Signaling data is stored only in memory and is lost when the signaling server terminates.

This is appropriate because SDP offers and answers are temporary connection-establishment data.

### Polling-Based Communication

The current implementation uses repeated HTTP requests rather than a persistent signaling connection.

This is simple but introduces additional requests and a potential delay of up to approximately one polling interval before newly available session information is detected.

## Error Handling

The signaling server validates incoming JSON requests and returns HTTP status `400` when an exception occurs while processing an offer or answer.

For example:

```python
except Exception as e:
    logger.error(f"Error receiving offer: {e}")
    return web.json_response(
        {"status": "error", "message": str(e)},
        status=400
    )
```

The client and receiver additionally log errors that occur while communicating with the signaling server.

More specific timeout and connection error handling is planned as part of the overall FOSIA error-handling implementation.

## Lifecycle

The signaling server is started asynchronously:

```python
runner = web.AppRunner(app)
await runner.setup()

site = web.TCPSite(
    runner,
    "127.0.0.1",
    8080
)

await site.start()
```

The `run_signaling_server()` function returns the `AppRunner`, allowing the caller to shut the server down later:

```python
await runner.cleanup()
```

When executed directly, the signaling server currently remains active until the process is terminated.

The returned runner also allows the signaling server to be integrated into the overall FOSIA application lifecycle.
