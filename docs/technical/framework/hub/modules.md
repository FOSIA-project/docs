# Modules and sessions

Each module process opens one control connection. The bus accepts it, wraps it in a `Module`, and runs a single read loop for that socket. Replies (`reply_to` set) complete an outbound wait. Everything else is a request and is dispatched by type.

```text
connected ──register──► registered ──listen──► listening ──ready──► ready
                              │
                              └── source skips listen ──► ready
```

Public `module.send(msg)` is allowed only in `ready`. It waits up to 5 seconds for an ack or reply and returns `None` on timeout. The register handshake uses an internal send that does not check `ready`, because the module is not ready yet.

## Register

On `register` the hub reads `name` and rejects a second live connection with the same name. The new connection is nacked and dropped.

`pid` is taken from the Unix socket (`SO_PEERCRED`). A `pid` in the payload is removed. The rest of the payload is kept on `module.info` (name, version, capabilities, target).

The hub then:

1. Acks immediately with `{ok, name, buffer, listen, fanout}`. `listen` is true for every non-source. `fanout` is true in chain mode for ids below the session creator. `buffer` is the value from hub config.
2. Syncs existing session ids to this module, before it is listening.
3. If it must listen, sends `listen` and waits for that ack (5 seconds). The address in the ack is stored as `module.dest`.
4. Publishes dests again so the previous hop can see the new address.

That order is what makes a late join safe. The new module learns session ids first, then opens its data socket, then everyone else is updated so the previous hop's dest is no longer `None`.

`ready` before a required listen is nacked. A second `ready` from a module that is already ready is acked again.

## What a session is

A session is a **routing id**, not a socket and not a user conversation. The hub allocates it with `create-session` and stores:

```text
session_id → { name, dests, active, finished }
```


| Field      | Meaning                                                                 |
| ---------- | ----------------------------------------------------------------------- |
| `name`     | Module that created it                                                  |
| `dests`    | Optional per-module next-hop overrides                                  |
| `active`   | Last module that sent `active-session` (first inbound data on that hop) |
| `finished` | Modules that sent `finish-session`                                      |


The same id is used on every hop. Each module is told only its own next dest for that id.

By default, wake word is the session creator. VAD sits between the source and that creator, so it receives `add-dests` and never sees the session map. Another row can be the creator when it is the one flagged `session-creator: true`.

```text
add-dests to audio-in:     [ <vad listen> ]
add-dests to vad:          [ <wakeword listen> ]
create-session ack to wakeword:  { session_id: S, dest: <stt listen> }
add-sessions to stt:       { S: { dest: <llm listen> } }
```

Fan-out modules are not in the session map. They receive `add-dests`: a de-duplicated list of the default next hop plus any session overrides.

## create-session

The caller must be `ready`. In chain mode it must also share the session-creator id. Otherwise the request is nacked. Audio-transmission modules are nacked in either mode.

The hub allocates an id, records the creator, then announces that id to **every other ready session module** and waits for each ack (5 seconds). Fan-out modules are skipped. Only then does the creator receive `{session_id, dest}`.

The creator is not in the announce. Its next hop is on its own ack, not via `add-sessions`.

If any announce fails or times out, the hub removes the session and nacks the creator. That wait is why the first sessioned frame is not dropped as an unknown id: downstream modules already have the id before the creator is allowed to send.

If nobody else is ready, announce is a no-op and the creator is acked immediately. `dest` may still be `None`.

## Finish, remove, disconnect

`finish-session` adds the sender to `finished`. The session stays. Downstream can still use the id, and `put_data` still works until the hub removes it.

`remove_session(session_id)` deletes the row and sends `remove-session` to the remaining session modules.

`delete_data(session_id)` tells those modules to drop buffered frames and close the data connection for that id. The id itself stays.

Disconnect does not remove sessions, including the creator's. The hub forgets the module row, then republishes dests so live hops see `dest: null` where that process was the next socket. Leftover ids after a module exits are expected. They go away on `remove_session` or when the hub stops.

## Overrides and handlers

`set_session_dest(session_id, module_name, dest)` pins that module's next hop for one session. `dest` may be `None`, which makes that hop a sink. Fan-out modules get a fresh `add-dests`. Session modules get a one-entry `add-sessions`.

A ready module sends a control message with `send()`. The hub looks up `msg.type` and runs that handler. The handler's return value is the reply `send()` is waiting for.

```python
async def on_ping(module, msg):
    return {"echo": (msg.data or {}).get("n"), "name": module.name}

hub.modules.on("ping", on_ping)                        # every module
hub.modules.on("ping", on_ping, name="wakeword")       # this name wins
```

```python
# inside the module, after ready()
reply = await self.send(Message(type="ping", data={"n": 1}))
```

```text
module.send(Message type=ping)
        │
        ▼
hub.modules.on("ping")  →  handler(module, msg)
        │
        ▼
ack / nack / reply      →  the Message send() returns
```

A handler is `async (module, msg)`. Return a dict to ack with that data, `False` to nack, a `Message` to send as the reply, or `ANSWERED` if the handler already wrote the reply. An unknown type is nacked with `unhandled`. A handler that raises is nacked with the exception text. `send()` returns `None` on timeout (5 seconds) or when the module is not `ready`.

Built-in types are `register`, `ready`, `create-session`, `active-session`, and `finish-session`. Registering `on` for one of those types replaces that built-in for the modules it covers. The module side of `send()` is in [Sending to a hub handler](../module-sdk/index.md#sending-to-a-hub-handler).