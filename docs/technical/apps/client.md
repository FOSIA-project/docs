# Client app

Package: `client` (Hub process on the device). Entrypoint: `client.app:app` / `python -m client`.

## Usage

From `apps/client` (with the project venv and editable `hub[mesh]`):

```bash
.venv/bin/python -m client
.venv/bin/python -m client /path/to/config.yaml
```

Default config path is `config.yaml` in the working directory.

Optional config argument is the only CLI flag. Logging is configured at import time (INFO, module name in the format).

### What must be up

- Local modules that register on `hub.modules.listen`
- Peer mesh: at least one `peers:` row with `connect` (dial the server)
- Server hub listening and ready (hello completed) before announce / session sync / tunnels succeed

### Tests

```bash
.venv/bin/python -m pytest -q
```

## Configuration

Standard hub YAML ([Hub configuration](../framework/hub/configuration.md)) plus app expectations:

| Area | Client expectation |
| --- | --- |
| `hub.name` | Required for session sync (`user_id` on `sync-new-session`) |
| `hub.chain` | Typically `true` with a local `session-creator` |
| `peers` | Dial the server (`connect: ws://…`) |
| `modules.chain` | Local pipeline; rows with `where: <peer name>` are announced to that peer |
| `modules.audio-transmission` | Optional (for example `webrtc-client`) |
| `tunnel` on a module row | Optional control tunnel to a remote module |

Example shape (abbreviated):

```yaml
hub:
  name: client
  modules:
    listen: unix:///run/fosia/client.control.sock
  chain: true

peers:
  - name: server
    connect: ws://127.0.0.1:7443/peer

modules:
  chain:
    - id: 3
      name: wakeword
      session-creator: true
      where: local
    - id: 4
      name: stt-whisper
      where: server          # announced to peer "server"
    - id: 5
      name: llm
      where: local           # inject target after the peer segment

  audio-transmission:
    - name: webrtc-client
      where: local
      tunnel:
        - to: webrtc-server   # remote module name
          on: server          # peer hub name
          type: webrtc-tunnel-*
```

## Startup wiring

`app.py` installs handlers, then runs the hub; module announce runs as a background task (reconnect loop):

1. `install_session_sync(hub)` — after `create-session` ack → `sync-new-session`
2. `install_pipeline_return(hub)` — peer `sync-pipeline-data` → local `send_data` inject
3. `install_tunnels(hub)` — config `tunnel:` rows
4. `announce_modules(hub)` task — `needs-modules` whenever the primary peer is ready
5. `await hub.run()`

## Additions (app features)

### Module announce

**Module:** `announce.py` · **Helper:** `helpers.modules_for`

Collects chain + audio-transmission rows with `where` equal to the primary peer name and sends:

```text
Frame(type="needs-modules", data={"modules": [row, …]})
```

Loops: `wait_ready` → send → poll until peer drops → repeat. So a reconnect re-announces.

### Session sync

**Module:** `session_sync.py`

`hub.modules.after("create-session", …)` fires after a successful local create. Best-effort (does not delay the module ack):

```text
Frame(type="sync-new-session", data={"session_id", "user_id": hub.name})
```

Skipped if no peer, no `hub.name`, or peer not ready.

### Pipeline return (text inject)

**Module:** `pipeline_return.py` · **Helper:** `helpers.next_local_after_peer`

Handles peer `sync-pipeline-data`. Validates `session_id`, `user_id`, `msg_type == "text"`, and that the session exists locally. Then writes:

```python
Message(type="text", session_id=…, payload=text.encode("utf-8"))
```

to the listen dest of the first `where: local` chain module **after** a segment placed on the primary peer (via `hub.send_data`). No `user_id` on that local Message.

### Control tunnels

**Module:** `tunnel.py`

For each module row with:

```yaml
tunnel:
  - to: <remote module>
    on: <peer name>
    type: <exact or prefix*>
```

installs:

| Direction | Registration | Behavior |
| --- | --- | --- |
| Outbound | `hub.modules.on(pattern, …, name=local)` | Module `send` → `peer.send(Frame)` → map reply to `Message` |
| Inbound | `hub.peers.on(pattern, …)` (once per pattern) | Frame → `hub.modules[target].send(Message)` if this local module declared `on: <that peer>` |

Frame envelope (JSON; optional binary as `payload_b64`):

```python
data = {
  "module": "<target module>",
  "from": "<source module>",
  "data": msg.data or {},
  "payload_b64": "…",       # optional
  "session_id": "…",        # optional
  "user_id": "…",           # optional
}
```

Both hubs must declare a reciprocal tunnel for both directions. Prefix types need hub prefix matching ([Peers](../framework/hub/peers.md) / [Modules](../framework/hub/modules.md)).

**Module usage** (after `ready()`):

```python
reply = await self.send(Message(type="webrtc-tunnel-offer", data={"sdp": "…"}))
```

The remote module registers `on("webrtc-tunnel-offer", …)` (or each concrete type) and returns a dict / `Message` / `False` as usual.

## Package layout

| File | Role |
| --- | --- |
| `app.py` | Logging, install hooks, `hub.run()` |
| `announce.py` | `needs-modules` loop |
| `session_sync.py` | `create-session` → `sync-new-session` |
| `pipeline_return.py` | `sync-pipeline-data` → inject |
| `tunnel.py` | Config tunnels |
| `helpers.py` | `modules_for`, `next_local_after_peer` |
| `__main__.py` | `python -m client` |

## Related

- [Server app](server.md)
- [Hub peers](../framework/hub/peers.md)
- [Hub optional data helpers](../framework/hub/index.md#optional-data-helpers) (`send_data`)
