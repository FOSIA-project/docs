# Peers

A peer mesh links hub processes. It carries control frames between hubs. It does not carry module audio, and it does not use ipc-lib.

The mesh is optional. Install it with the `hub[mesh]` extra (`websockets`). If the config enables peers and the extra is missing, constructing `Hub` fails with that import error. With no peers block, `hub.peers` is `None` and the extra is unused.

## Config

Peers are enabled when `hub.peers.listen` is set, or at least one `peers` row has `connect`.

```yaml
hub:
  name: server
  peers:
    listen: ws://0.0.0.0:7443/peer

peers:
  - name: client
```

A device hub that only dials:

```yaml
hub:
  name: client

peers:
  - name: server
    connect: ws://127.0.0.1:7443/peer
```

`hub.name` is required. Each peer row needs `name`. That name cannot be this hub, and names must be unique. `connect` and `listen` must be `ws://` or `wss://`. Listening on `wss://` is rejected until certificates are wired; dialing `wss://` is allowed by the URI check. A config with peer rows but neither `listen` nor any `connect` is rejected.

Inbound connections must use the path in `hub.peers.listen`. For `ws://0.0.0.0:7443/peer` that path is `/peer`. Any other path is closed.

## Hello

A dialing hub sends `hello` with its own name and waits for an ack. The ack must name the peer that was configured for that URI. A mismatch, a failed hello, or a name that is already connected closes the socket. The dial loop then retries, starting at 0.2 seconds and backing off to 5 seconds.

An inbound `hello` is accepted only if `name` is in the allowlist and that peer is not already up. The ack body is this hub's name. An unknown or duplicate peer is nacked and the socket is closed.

```text
client hub                         server hub
    │  hello { name: client }           │
    │ ─────────────────────────────────►│
    │                                   │ allowlist check
    │  ack { name: server }             │
    │ ◄─────────────────────────────────│
    │                                   │
    both sides: state = ready
```

States are `connected`, then `ready` after hello, or `hello-failed`. Hello itself uses the internal send, same as module register.

### Waiting for ready

```python
peer = await hub.peers.wait_ready("server")          # blocks until hello
peer = await hub.peers.wait_ready("server", timeout=5.0)
```

On disconnect the ready event is cleared so a later `wait_ready` waits for the next hello.

### Config helpers on the mesh

| Accessor | Meaning |
| -------- | ------- |
| `hub.peers.primary` | First `peers:` row `name` |
| `hub.peers.configured` | All configured peer names, in config order |

## Frames

A frame is a small JSON object, not an ipc-lib `Message`:

```json
{"type": "hello", "id": "<uuid hex>", "data": {"name": "client"}}
```

Replies add `reply_to` set to the request `id`. The only built-in type is `hello`.

## send vs notify

| API | Behavior |
| --- | -------- |
| `await peer.send(frame)` | Ready-only RPC. Waits up to 5 seconds for ack/nack/reply. Returns `None` on timeout or not ready |
| `await peer.notify(frame)` | Ready-only write. Does not wait. Returns `True` if the write succeeded. Remote replies are ignored (logged at debug) |

Use `send` when you need the answer (`needs-modules`, `sync-new-session`). Use `notify` for one-way traffic (`sync-pipeline-data`).

## Custom handlers

A ready peer sends a frame with `send()` or `notify()`. The other hub looks up `frame.type` and runs the handler from `peers.on`. For `send()`, the handler's return value is the reply the caller is waiting for. This is the same request and reply shape as a module control message, on a `Frame` instead of an ipc-lib `Message`.

```python
from hub.peers import Frame

async def on_ping(peer, frame):
    return {"from": peer.name, "n": (frame.data or {}).get("n")}

hub.peers.on("ping", on_ping)                    # every peer
hub.peers.on("ping", on_ping, name="client")     # this name wins
hub.peers.on("webrtc-tunnel-*", on_tunnel)       # prefix match
```

A type ending in `*` is a **prefix pattern** (same lookup rules as module handlers: exact before prefix, longest prefix wins, named before global).

```python
# on the other hub, after hello
reply = await hub.peers["server"].send(Frame(type="ping", data={"n": 1}))
await hub.peers["server"].notify(Frame(type="sync-pipeline-data", data={...}))
```

```text
peers["server"].send(Frame type=ping)
        │
        ▼
remote hub.peers.on("ping")  →  handler(peer, frame)
        │
        ▼
ack / nack / reply           →  the Frame send() returns
```

A handler is `async (peer, frame)`. Return a dict to ack with that data, `False` to nack, a `Frame` to send as the reply, or `ANSWERED` if the handler already wrote the reply. An unknown type is nacked with `unhandled`. A handler that raises is nacked with the exception text. `send()` returns `None` on timeout (5 seconds) or when that peer is not `ready`.

`hub.peers["server"]` is the live `Peer` after hello. Sending before ready is dropped and logged. The only built-in type is `hello`. Registering `on` for `hello` replaces that handshake for the peers it covers.

### after hooks

`hub.peers.after(type, hook)` registers an observer. It runs after the primary handler returns and before the reply is written. The hook is `async (peer, frame, result)`. It must not change the reply; exceptions are logged and the reply is still sent.

```python
async def after_ping(peer, frame, result):
    LOG.info("ping from %s -> %s", peer.name, result)

hub.peers.after("ping", after_ping)
```

