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

States are `connected`, then `ready` after hello, or `hello-failed`. Public `peer.send(frame)` is allowed only when ready. It waits up to 5 seconds for a reply and returns `None` on timeout. Hello itself uses the internal send, same as module register.

## Frames

A frame is a small JSON object, not an ipc-lib `Message`:

```json
{"type": "hello", "id": "<uuid hex>", "data": {"name": "client"}}
```

Replies add `reply_to` set to the request `id`. The only built-in type is `hello`.

`hub.peers.on(type, handler)` registers more types, optionally for one peer name. A named handler replaces the global one for that name. The handler is `async (peer, frame)` and returns the same kinds of values as a module handler: a dict (ack), `False` (nack), a `Frame`, or `ANSWERED`.

`hub.peers["server"]` is the live `Peer` after hello. Sending before ready is dropped and logged.
