# Module SDK

The module SDK is how a pipeline stage talks to the [hub](../hub/index.md). A module subclasses `Module`, registers on the control socket, and either listens for data or pushes it to the next hop.

The SDK does not copy audio through the hub. Control messages go to the hub. PCM and text go directly to the neighbor the hub named.

```text
module.yaml + config.yaml
        │
        ▼
   Module.run()
        │
        ├── on_init()          load models, once per connection
        ├── register           control socket to the hub
        ├── listen             only if the hub says this module is not the source
        ├── on_run()           background task; default awaits ready()
        │     └── on_ready()   awaited inside ready(), before the ready message
        └── on_data(msg, session)
                inbound frames, after ready()
```

Public exports are `Module`, `Session`, and `ANSWERED`.

## Two files

`module.yaml` is the manifest. `config.yaml` is runtime config. Both paths default to those names in the working directory.

```yaml
# module.yaml
name: wakeword
version: 1.0
target:
  - local
capabilities:
  - wakeword.detect
transport:
  listen: unix:///tmp/fosia/wakeword.data.sock
```

```yaml
# config.yaml
hub:
  listen: unix:///run/fosia/hub.control.sock
```

| File | Fields the SDK reads |
| --- | --- |
| `module.yaml` | `name`, `version`, `capabilities`, `target`, `transport.listen` |
| `config.yaml` | `hub.listen` |

If `hub.listen` is omitted, the SDK dials `unix:///run/fosia/hub.control.sock`, which is the hub default.

`transport.listen` is the data address this process opens when the hub sends `listen`. A value without `://` is treated as a socket name and becomes `unix:///run/fosia/<name>`. The source omits `transport.listen`. The hub never asks it to listen.

The hub decides `buffer`, whether to listen, and whether this hop fans out. Those flags come back on the register ack. Setting them only in `module.yaml` does not change routing.

`name` is the key the hub routes on. Use the same name as this process's row in the hub config. A name that is not in the chain still registers; it is not the source, and in chain mode it cannot create sessions.

## Process shape

```python
class Wakeword(Module):
    async def on_data(self, msg, session):
        ...

asyncio.run(Wakeword("wakeword.yaml", "config.yaml").run())
```

`run()` connects, starts `on_run` as a background task, and waits on the control read loop. If the hub socket drops it clears local session state and connects again. `stop()` closes data sockets, drops sessions, disconnects, and calls `on_stop()`.

The control connection retries on its own, from 0.2 seconds up to 5 seconds, until the hub socket accepts it.

## Hooks

| Hook | When it runs | Default |
| --- | --- | --- |
| `on_init` | Once before register, each time the control connection comes up | returns `True` |
| `on_run` | Background task after register, beside the control read loop | awaits `ready()` |
| `on_ready` | Awaited by `ready()`, before the `ready` message is sent | returns `True` |
| `on_data(msg, session)` | One inbound data frame, after the local ready flag is set | logs the frame |
| `on_stop` | After sockets are closed | returns `True` |

`run()` does not await `on_run`. It schedules that hook in the background so the control loop can keep handling `listen` and session updates. `ready()` awaits `on_ready()` on that same task, then sends `ready` to the hub.

If you override `on_run`, call `await self.ready()` before you expect `on_data`. Inbound frames sit on the data socket until the local ready flag is set. `ready()` waits until a required listen socket is up, then awaits `on_ready()`.

`on(type, handler)` replaces the handler for one control type the hub sends to this module. The handler is `async (msg)`. Return a dict to ack, `False` to nack, a `Message` to send as the reply, or `ANSWERED` if you already wrote the reply.

The built-in control types are `listen`, `add-dests`, `add-sessions`, `delete-data`, and `remove-session`. Leave those in place unless you are extending the handshake.

## Sending to a hub handler

After `ready()`, `send()` delivers a control message to the hub. The hub runs the handler registered with `hub.modules.on` for that type and writes the reply. `send()` waits up to 5 seconds and returns that reply.

```python
from ipc_lib import Message

reply = await self.send(Message(type="ping", data={"n": 1}))
```

| `send()` returns | Meaning |
| --- | --- |
| `Message` with `type="ack"` | Handler returned a dict. The dict is `reply.data` |
| `Message` with `type="nack"` | Handler returned `False`, raised, or no handler is registered (`data["error"]` is `unhandled`) |
| Other `Message` | Handler returned a `Message`. The hub fills `reply_to` when the handler left it empty |
| `None` | Timed out, the control socket is down, or `ready()` has not finished |

The matching hub handler is `async (module, msg)`. A handler registered for this module's name wins over the handler for every module. See [Modules and sessions](../hub/modules.md#overrides-and-handlers).

## Where to go next

- [Writing a module](writing-a-module.md) — source, session creator, and downstream hops
- [Data and sessions](data-and-sessions.md) — `fanout_data`, `create_session`, `put_data`, `finish`
- [Hub modules and sessions](../hub/modules.md) — what the control plane does with those calls
