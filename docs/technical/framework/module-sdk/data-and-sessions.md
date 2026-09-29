# Data and sessions

Control messages and data frames are different sockets. A session id is how those data frames stay on one route. It is not a connection, and it is not the conversation object used above the pipeline.

Fan-out hops do not hold a `Session`. In the usual chain that is audio-in and VAD: both sit before wake word, and both emit with no `session_id`. From the session creator onward, each module holds its own `Session` for the same id. `session.dest` is **this** module's next hop. By default that creator is wake word.

Modules between the source and the creator might set flow events on those frames (`flow_id` and `data["event"]`).

```text
audio-in              vad                    wakeword (WWD)              stt
fanout_data(pcm)      on_data(session=None)  create_session()            on_data(..., session)
no session_id         fanout_data(pcm)       dest = stt's listen addr    dest = the hop after stt
                      no session_id          put_data(pcm)               same session_id
```

## Fan-out

Chain mode gives every module below the session-creator id an `add-dests` list. `fanout_data` connects to each address and sends the frame with no `session_id`. Connections are cached and replaced when the list changes.

```python
await self.fanout_data(pcm_chunk)
```

With an empty list the call logs and returns. The list is empty when the next module is not connected, when it has registered but has not finished `listen`, or when a `where` change has no transmission module listening. A later `add-dests` fills the list; frames that already passed are gone. Fan-out does not use `buffer`.

## create_session

```python
session = await self.create_session()
```

The call sends `create-session` and waits for the hub ack. The ack includes `session_id` and this module's `dest`. The hub does not ack until every other ready session module has stored the id. If the hub nacks (not ready, not the session creator, announce timeout), the method returns `None`.

The SDK then inserts a local `Session`. Downstream modules receive the same id through `add-sessions`, each with its own dest.

Call this when a stream starts. Downstream hops should keep the inbound id. Calling it again opens a second route.

## put_data and finish

```python
await session.put_data(pcm_chunk)    # or a str, a Message, or an iterable
await session.finish()
```

`put_data` connects to `session.dest` and sends one or more data frames tagged with `session_id`. It returns without sending when:

- the session was removed
- `dest` is `None` (this hop is the sink, or the next module is not listening yet)

Frames dropped because `dest` is `None` are not stored. A later `add-sessions` can fill that dest in; audio that already arrived is gone.

When the hub config set `buffer: true` for this module, each frame passed to `put_data` while a dest is set is kept. If a later `add-sessions` changes that dest, the kept frames are sent to the new address. Hot audio hops leave `buffer: false`, so a dest change does not replay old PCM.

`finish()` sends `finish-session`. The hub records this module in `finished` and leaves the id in place. Downstream can still receive and send on it. `finish()` does not close the data socket and does not drop the local `Session`. A second `finish()` on the same handle is a no-op that returns `True`.

The hub deletes an id only on `remove-session` (or when the hub process stops). The SDK then drops the local `Session`, closes its data connection, and further `put_data` calls log and return.

`delete-data` clears stored frames and closes the data connection for that id, and keeps the `Session`.

## Inbound data

The data listen loop starts while the `listen` command is handled, before the ack is written. Frames reach `on_data` only after the local ready flag is set.

| Inbound frame | `session` argument |
| --- | --- |
| No `session_id` (fan-out from upstream) | `None` |
| Known `session_id` | This module's `Session` for that id |
| Unknown `session_id` | Frame dropped. `on_data` is not called |

The first frame on a known session also sends `active-session` in the background. That updates the hub's `active` field. Do not wait on it.

## Reconnect

If the control socket closes, `run()` cancels the current `on_run` task, drops every local session and fan-out connection, and registers again. Sessions that still exist in the hub are pushed back down through `add-sessions` / `add-dests` during that register. A data listen socket that is already open stays open; the hub's next `listen` reuses it. `stop()` is what closes those sockets.

Outbound control sends (`create_session`, `finish`, and anything through `send()`) wait up to 5 seconds. A timeout returns `None` from the low-level send, and `create_session()` turns that into `None`.
