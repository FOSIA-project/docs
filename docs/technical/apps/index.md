# Client and server apps

The **client** and **server** packages are thin Hub processes. They load a YAML config, start a local module bus (and peer mesh), and install app-level handlers that glue the two hubs together.

They do **not** replace modules. Modules still register with each hub and move PCM/text on data sockets. The apps add cross-hub control: which modules the remote side should run, when to open a session there, how text results return, and optional control tunnels (for example WebRTC signaling).

```text
 device (client hub)                         cloud (server hub)
 ┌─────────────────────┐                    ┌─────────────────────┐
 │ audio-in → … → ww   │   peer WS          │ webrtc-server       │
 │      ↘ where:server │◄──────────────────►│ stt-whisper         │
 │ llm ← … ← (inject)  │  control frames    │ return sink (UDS)   │
 │ webrtc-client       │                    │                     │
 └─────────────────────┘                    └─────────────────────┘
```

| App | Role |
| --- | --- |
| [Client](client.md) | Device hub: announces remote modules, syncs sessions outward, reinjects returned text, tunnels to remote modules |
| [Server](server.md) | Cloud hub: accepts announcements, opens ACL’d sessions, return sink for text, tunnels back to the client |

Hub primitives these apps rely on: [Hub](../framework/hub/index.md), [Modules and sessions](../framework/hub/modules.md), [Peers](../framework/hub/peers.md).

## Peer frame catalog

| Frame type | Direction | RPC | Purpose |
| --- | --- | --- | --- |
| `needs-modules` | client → server | `send` | Announce chain/audio-transmission rows placed on the peer |
| `sync-new-session` | client → server | `send` | Open the same session id on the server for announced modules |
| `sync-pipeline-data` | server → client | `notify` | Return text result for a session |
| `webrtc-tunnel-*` (config) | either way | `send` | Module↔module control tunnel (same Message type over the peer) |

## Typical data path

1. Local chain runs until a `where: server` hop (audio crosses via WebRTC / audio-transmission, not through the hub).
2. Wake word (or another creator) `create-session` → client syncs the id to the server.
3. Server opens the session only for modules the client announced (for example `stt-whisper`), with the last hop’s dest set to the hub return socket.
4. STT writes text into that return sink → server notifies `sync-pipeline-data` → client injects a `text` Message into the next local module after the peer segment (for example `llm`).

Control tunnels are independent of that path: a local module `send(Message(type="webrtc-tunnel-…"))` RPCs to the configured remote module whenever both sides declare a matching `tunnel:` row and the peer is ready.
