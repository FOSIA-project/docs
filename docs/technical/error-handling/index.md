# Error Handling

FOSIA handles errors explicitly to prevent communication failures from causing an uncontrolled application termination. The following table provides an overview of the currently considered error cases and their handling status.

## Error Cases

| Side   | Category  | Error Case                                                                                          | Handling                        | Tested |
| ------ | --------- | --------------------------------------------------------------------------------------------------- | ------------------------------- | ------ |
| Client | Signaling | [Signaling server unreachable during offer](#signaling-server-unreachable-during-offer)             | 3 retries, then abort           | ✅      |
| Client | Signaling | [No answer received](#no-answer-received)                                                           | 30 s timeout                    | ✅      |
| Client | Signaling | [Signaling connection lost during answer polling](#signaling-connection-lost-during-answer-polling) | Continue polling until timeout  | ✅      |
| Client | WebRTC | [Invalid SDP answer](#invalid-sdp-answer) | Validate response and SDP; abort on invalid data | ✅ |

---

## Client

### Signaling

#### Signaling Server Unreachable During Offer

The client retries the offer request up to three times if the signaling server cannot be reached.

Each request has a timeout of five seconds. A one-second delay is used between attempts. If all attempts fail, the connection attempt is aborted and the `RTCPeerConnection` is closed during cleanup.

```python
MAX_RETRIES = 3

for attempt in range(1, MAX_RETRIES + 1):
    try:
        r = requests.post(
            SIGNALING + "/offer",
            json={
                "sdp": pc.localDescription.sdp,
                "type": pc.localDescription.type
            },
            timeout=5
        )

        r.raise_for_status()
        break

    except requests.exceptions.RequestException as e:
        logger.warning(
            f"Could not reach signaling server "
            f"(attempt {attempt}/{MAX_RETRIES}): {e}"
        )

        if attempt == MAX_RETRIES:
            logger.error("Signaling server not reachable")
            return

        await asyncio.sleep(1)
```

Test result:

```text
Could not reach signaling server (attempt 1/3)
Could not reach signaling server (attempt 2/3)
Could not reach signaling server (attempt 3/3)
ERROR - Signaling server not reachable
Closing PeerConnection
PeerConnection closed, exiting...
```

#### No Answer Received

After sending the offer, the client polls the signaling server for an SDP answer.

If no answer is available yet, the signaling server returns an empty JSON object:

```json
{}
```

This is not treated as an error. The client continues polling once per second.

An overall timeout of 30 seconds prevents the client from waiting indefinitely. If no answer is received within this period, the connection attempt is aborted.

Test result:

```text
Polling answer: {}
Polling answer: {}
Polling answer: {}
...
ERROR - Signaling Server connection timed out
Closing PeerConnection
PeerConnection closed, exiting...
```

#### Signaling Connection Lost During Answer Polling

The signaling server can become unavailable while the client is waiting for the answer.

Network and HTTP request errors are caught and logged as warnings. The client continues polling instead of immediately aborting the connection attempt.

The overall 30-second answer timeout remains active. Therefore, temporary connection problems can be tolerated, while a permanently unavailable signaling server eventually causes a timeout.

Relevant code:

```python
except requests.exceptions.RequestException as e:
    logger.warning(
        f"Connection to signaling server lost: {e}"
    )
```

Test result:

```text
Polling answer: {}
Polling answer: {}

WARNING - Connection to signaling server lost: ...

Polling answer: {}
Polling answer: {}

ERROR - Signaling Server connection timed out
Closing PeerConnection
PeerConnection closed, exiting...
```

### WebRTC

#### Invalid SDP Answer

The client validates the SDP answer received from the signaling server before applying it as the remote description.

First, the response is checked for the required `sdp` and `type` fields. The answer type must also be `"answer"`. If any of these checks fail, the connection attempt is aborted.

If the answer passes the initial validation, the SDP is passed to `aiortc` using `setRemoteDescription()`. If the SDP is invalid or incompatible with the previously created offer, `aiortc` raises an exception. The exception is caught, logged, and the connection attempt is aborted.

No further request for the same answer is made in these cases. Since the signaling server successfully delivered an answer, retrying the request would normally return the same invalid data. Retries are therefore only used for communication errors during answer polling.

In all cases, the `RTCPeerConnection` is closed during cleanup.

Relevant code:

```python
# Check if answer has all required fields
if not data or "sdp" not in data or "type" not in data:
    logger.error("Invalid answer from signaling server")
    return

# Check if answer is of type "answer"
if data["type"] != "answer":
    logger.error(
        f"Invalid SDP answer type from signaling server: {data['type']}"
    )
    return

try:
    await pc.setRemoteDescription(
        RTCSessionDescription(
            sdp=data["sdp"],
            type=data["type"]
        )
    )
except Exception as e:
    logger.exception(f"Failed to set remote description: {e}")
    return
```
*Test Results*

Test 1 – Missing type:
```
 INFO - [CLIENT] - Valid Polling Answer received
 INFO - [CLIENT] - Setting remote description...
 ERROR - [CLIENT] - Invalid answer from signaling server
 INFO - [CLIENT] - Closing PeerConnection
 INFO - [CLIENT] - PeerConnection closed, exiting...
```
The missing type field was detected by the client-side validation before the answer was passed to aiortc.

Test 2 – Invalid SDP:
```
 INFO - [CLIENT] - Valid Polling Answer received
 INFO - [CLIENT] - Setting remote description...
 ERROR - [CLIENT] - Failed to set remote description: Media sections in answer do not match offer
 INFO - [CLIENT] - Closing PeerConnection
 INFO - [CLIENT] - PeerConnection closed, exiting...
```
The SDP passed the initial validation but was rejected by aiortc because its media sections did not match the offer.

Test 3 – Invalid answer type:
```
 INFO - [CLIENT] - Valid Polling Answer received
 INFO - [CLIENT] - Setting remote description...
 ERROR - [CLIENT] - Invalid SDP answer type from signaling server: blubu
 INFO - [CLIENT] - Closing PeerConnection
 INFO - [CLIENT] - PeerConnection closed, exiting...
```
The invalid answer type was detected by the client-side validation before creating the RTCSessionDescription.

All three test cases resulted in a controlled termination and proper RTCPeerConnection cleanup.
### Audio

### Lifecycle



## Server

### Signaling

### WebRTC

### Audio

### STT
