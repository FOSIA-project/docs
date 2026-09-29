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


The hub never opens a data socket. It stores the listen address each module reports, then tells the previous hop to connect there.

## Chain position

After the config is loaded, chain rows are sorted by `id` (rows with no `id` sort last). The first name in that order is the **source**.

The source does not open a data listen socket, and the hub never stores a dest for it. Every other module is asked to listen during register. Its listen address becomes the dest the previous hop connects to.

With `hub.chain: true`, position relative to the session creator also decides how a module sends:

- Modules whose `id` is below the session-creator id **fan out**. They receive a list of next-hop addresses and emit frames with no `session_id`.
- The session creator and every later module use **sessions**. Later modules receive `add-sessions` entries `{session_id: {dest: next hop}}`. The creator learns its next hop from the `create-session` ack `{session_id, dest}`.

With `hub.chain: false` (the default), every module uses sessions. Any ready module may create one, except an audio-transmission row. See [Modules and sessions](modules.md).

## Related pages

- [Configuration](configuration.md) — YAML shape, chain mode, and cross-host hops
- [Modules and sessions](modules.md) — register, ready, and how a session id is routed
- [Peers](peers.md) — hub-to-hub mesh
- [Module SDK](../module-sdk/index.md) — the module side of the same protocol

