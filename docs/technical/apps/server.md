# Server app

Package: `server` (Hub process in the cloud / on the host that listens for peers). Entrypoint: `server.app:app` / `python -m server`.

## Usage

From `apps/server` (with the project venv and editable `hub[mesh]`):

```bash
.venv/bin/python -m server
.venv/bin/python -m server /path/to/config.yaml
```

Default config path is `config.yaml` in the working directory.

### What must be up

- Local modules that register on `hub.modules.listen` (for example `webrtc-server`, `stt-whisper`)
- Peer mesh listen: `hub.peers.listen` and an allowlisted `peers:` row for the client
- Return sink UDS (started by the app) so last-hop session dests can write text back

### Tests

```bash
.venv/bin/python -m pytest -q
```

## Configuration

| Area | Server expectation |
| --- | --- |
| `hub.name` | Required for the mesh hello |
| `hub.peers.listen` | WebSocket listen URI for inbound peers |
| `peers` | Allowlist (for example `name: client`) |
| `modules.chain` | Modules this host actually runs (`where: local`) |
| `tunnel` on a module row | Reciprocal control tunnel back to the client module |

Example shape (abbreviated):

```yaml
hub:
  name: server
  modules:
    listen: unix:///run/fosia/server.control.sock
  chain: true
  peers:
    listen: ws://0.0.0.0:7443/peer

peers:
  - name: client

modules:
  chain:
    - id: 1
      name: webrtc-server
      where: local
      tunnel:
        - to: webrtc-client
          on: client
          type: webrtc-tunnel-*
    - id: 2
      name: stt-whisper
      where: local
```

The return data listen URI is derived from the control listen (see helpers below). It is not a separate YAML key.

## Startup wiring

1. Set `hub.return_dest` from `return_listen_uri(hub.config)`
2. `install_modules_handler(hub)` — `needs-modules`
3. `install_session_sync_handler(hub)` — `sync-new-session`
4. `install_tunnels(hub)` — config `tunnel:` rows
5. Background `run_return_sink(hub)` — `serve_data` on the return URI
6. `await hub.run()` (cancel the return task on stop)

## Additions (app features)

### Accept module announcements

**Module:** `modules.py`

Handles peer `needs-modules`. Payload `modules` must be a list of rows with `name`. Every name must exist in `hub.module_names` or the frame is nacked with `missing`.

On ack, stores `hub.remote_modules[peer.name] = [row, …]` for later session ACL.

### Session sync (open + ACL)

**Module:** `session_sync.py` · **Helper:** `helpers.needed_modules_for_peer`

Handles `sync-new-session` with `session_id` and `user_id`.

| Case | Result |
| --- | --- |
| Missing fields | nack |
| Session exists, same `user_id` | ack (idempotent) |
| Session exists, different `user_id` | nack `session exists` |
| No modules announced for that peer | nack `no modules announced` |
| Otherwise | `hub.open_session(…)` |

`open_session` call:

```python
await hub.open_session(
    session_id,
    name=peer.name,
    user_id=user_id,
    modules=needed,                 # announced names, server config order
    sink_dest=hub.return_dest,      # last hop → return UDS
)
```

That sets `allowed_modules` so only those names get `add-sessions`. Per-hop dests chain through live module listens; the **last** name in `needed` gets `sink_dest` (the return socket). See [Modules and sessions](../framework/hub/modules.md).

### Return path (text → peer)

**Module:** `return_path.py` · **Helper:** `helpers.return_listen_uri`

`run_return_sink` calls `hub.serve_data(uri, on_msg)`. Inbound ipc-lib Messages:

- Must have `session_id` known to `hub.sessions`
- Only `type == "text"` is forwarded (UTF-8 payload)
- Looks up owning peer from `session["name"]` and `user_id` from the session
- `peer.notify(Frame(type="sync-pipeline-data", data={session_id, user_id, msg_type, text}))`

Fire-and-forget on the peer side (`notify`, not `send`).

**URI derivation** from `hub.modules.listen`:

| Control listen | Return listen |
| --- | --- |
| `…/foo.control.sock` | `…/foo.return.sock` |
| other `unix://…` | append `.return` |
| fallback | `unix:///run/fosia/server.return.sock` |

### Control tunnels

**Module:** `tunnel.py` — same behavior as the [client](client.md#control-tunnels). Reciprocal YAML is required:

```yaml
tunnel:
  - to: webrtc-client
    on: client
    type: webrtc-tunnel-*
```

Outbound: local module → peer → remote module. Inbound: peer frame → local module `send`, reply mapped back.

## Package layout

| File | Role |
| --- | --- |
| `app.py` | Logging, install hooks, return sink task, `hub.run()` |
| `modules.py` | `needs-modules` |
| `session_sync.py` | `sync-new-session` → `open_session` |
| `return_path.py` | Return UDS → `sync-pipeline-data` |
| `tunnel.py` | Config tunnels |
| `helpers.py` | `return_listen_uri`, `needed_modules_for_peer` |
| `__main__.py` | `python -m server` |

## Related

- [Client app](client.md)
- [Hub `open_session` / `allowed_modules`](../framework/hub/modules.md)
- [Hub `serve_data` / `send_data`](../framework/hub/index.md#optional-data-helpers)
