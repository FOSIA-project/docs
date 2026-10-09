# Hub

The hub is the **control plane** for a FOSIA pipeline. Modules register with it, and it tells each module where the next hop is. Audio and text never pass through the hub process.

Data moves on separate sockets between neighboring modules, using ipc-lib. The hub only exchanges control messages: register, listen, sessions, and routing updates.

```text
                      control (Unix socket)
         +-----------------------+-----------------------+
         |                       |                       |
         v                       v                       v
   +-----+------+   data   +-----+------+   data   +-----+------+
   |  audio-in  | -------> |    vad     | -------> |  wakeword  |
   |   source   |          |  fan-out   |          |  default   |
   |            |          |            |          |  creator   |
   +-----+------+          +-----+------+          +-----+------+
         |                       |                       |
         +-----------------------+-----------------------+
                   each process talks to the hub
                    the hub never sees the PCM
```

By default, wake word is the session creator. VAD sits between the source and that creator: it listens and fans out, and it does not open a session. Modules in that gap might attach flow events to the frames they forward. The hub does not read those fields.

One hub process owns one local pipeline. An optional [peer mesh](peers.md) links two hubs (for example a device hub and a server hub). That mesh is also control-only, and it does not use ipc-lib.

## What the process does

`Hub` loads a YAML file, starts the module bus, and starts the peer mesh when peers are configured.

```python
hub = Hub("config.yaml")
await hub.run()
```

From the command line:

```bash
python -m hub
python -m hub examples/hub-server.yaml
```

The default config path is `config.yaml` in the working directory.


| Piece    | Role                                                                                 |
| -------- | ------------------------------------------------------------------------------------ |
| `Bus`    | Accepts module connections, keeps the name map, routes session commands              |
| `Module` | One control connection: the only read loop, and outbound sends that wait for a reply |
| `Mesh`   | Optional hub-to-hub WebSocket links. Absent when the config has no peers             |
| `Peer`   | One remote hub. Same ready-gated send pattern as a module                            |


`hub.modules` is the bus. `hub.sessions` is the bus session table. `hub.peers` is the mesh, or `None`.

The bus listens on `hub.modules.listen`. If that is omitted, the address is `unix:///run/fosia/hub.control.sock`.

## Two kinds of traffic


|                  | Control                                     | Data                                                                    |
| ---------------- | ------------------------------------------- | ----------------------------------------------------------------------- |
| Who carries it   | Hub, over the control socket                | Modules, directly to the next hop                                       |
| What it contains | Register, listen, session ids, dest updates | PCM and text frames                                                     |
| Envelope         | ipc-lib `Message`, no `session_id`          | ipc-lib `Message` with `session_id` when the frame belongs to a session |


By default the hub only stores listen addresses modules report and tells the previous hop to connect there. It does not sit on the media path. `serve_data` / `send_data` are the exception when an app deliberately uses the hub as a data endpoint.

## Chain position

After the config is loaded, chain rows are sorted by `id` (rows with no `id` sort last). The first name in that order is the **source**.

The source does not open a data listen socket, and the hub never stores a dest for it. Every other module is asked to listen during register. Its listen address becomes the dest the previous hop connects to.

With `hub.chain: true`, position relative to the session creator also decides how a module sends:

- When a module id is flagged `session-creator`, modules whose `id` is below that id **fan out**. They receive a list of next-hop addresses and emit frames with no `session_id`. The creator and every later module use **sessions**. Later modules receive `add-sessions` entries `{session_id: {dest: next hop}}`. The creator learns its next hop from the `create-session` ack `{session_id, dest}`.
- When no module is flagged, the hub is the creator: no fan-out, every chain module gets `add-sessions` from the first, and module `create-session` is nacked.

With `hub.chain: false` (the default), every module uses sessions. Any ready module may create one, except an audio-transmission row. See [Modules and sessions](modules.md).

## Config helpers

| Accessor | Meaning |
| -------- | ------- |
| `hub.name` | `hub.name` from YAML |
| `hub.module_rows` | Chain rows, then `audio-transmission` rows (as dicts) |
| `hub.module_names` | Set of configured module names from those rows |
| `hub.peers.primary` | First `peers:` row `name`, or `None` if peers are off |
| `hub.peers.configured` | Ordered list of configured peer names |

## Optional data helpers

By default the hub only stores listen addresses and tells modules where to connect; PCM/text still moves module-to-module. When an app needs the hub process itself to be a data endpoint (for example a return sink that last-chain modules write into, or injecting text into a module’s listen socket), use these ipc-lib helpers.

### `await hub.serve_data(uri, handler) -> asyncio.Task`

Starts an ipc-lib listen on `uri` (typically a `unix://` path). For each accepted connection, the hub runs a receive loop and calls `async handler(msg)` once per inbound `Message`. The handler’s return value is ignored — this is not a request/reply channel; reply over control/peers if you need one.

Returns the accept-loop `Task`. Keep a reference and cancel it when the sink should stop, or let `hub.stop()` cancel it. Multiple callers may open several listeners; each is tracked until the task ends or stop runs.

Typical use: point a session’s last-module `dest` (or a synthetic return URI) at this listen address so modules write data into the hub app, which then forwards over the peer mesh or elsewhere.

```python
async def on_message(msg: Message) -> None:
    # e.g. forward text over peers.notify(...)
    ...

task = await hub.serve_data("unix:///run/fosia/hub.return.sock", on_message)
# later: task.cancel()  — or rely on hub.stop()
```

### `await hub.send_data(dest, message) -> bool`

Connects to an ipc-lib data listen at `dest` and sends one `Message` (same envelope modules use: `type`, `session_id`, `payload`, …). Returns `True` on success, `False` on connect/send failure (closed or unreachable dest).

Outbound connections are cached by `dest` and reused. A closed or failed connection is dropped from the cache; the next call opens a new one. There is no reply wait — fire-and-forget like a module writing to the next hop.

Typical use: inject a frame into a ready module’s `module.dest` (for example reinject peer-returned text into the next local module after a peer segment).

```python
ok = await hub.send_data(
    module.dest,
    Message(type="text", session_id=s, user_id=hub.name, payload=b"..."),
)
```

Both helpers are torn down in `hub.stop()`: data tasks are cancelled, listen servers closed, and cached outbound conns closed. They do not participate in session routing or `add-sessions`; the app chooses the URI/`dest` and wires it (session dest override, config return path, etc.).

## Related pages

- [Configuration](configuration.md) — YAML shape, chain mode, and cross-host hops
- [Modules and sessions](modules.md) — register, ready, and how a session id is routed
- [Peers](peers.md) — hub-to-hub mesh
- [Module SDK](../module-sdk/index.md) — the module side of the same protocol

