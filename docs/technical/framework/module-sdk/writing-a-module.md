# Writing a module

A new module is a `Module` subclass plus a `module.yaml` whose `name` is a row in the hub config. The hub's chain order decides the role. The subclass only implements the hooks for that role.

Three roles cover a normal voice pipeline.

| Role | Hub position | What you send | What you receive |
| --- | --- | --- | --- |
| Source | First chain name | `fanout_data` when chain mode puts this hop before the session creator. Otherwise `put_data` on a session this hop creates | Nothing. No listen socket, so `on_data` does not run |
| Session creator | The row with `session-creator: true` when `hub.chain` is true (if none, the hub is the creator and modules cannot open sessions). Any ready module except audio-transmission when chain mode is off | `create_session()`, then `put_data` on that handle | Upstream frames. `on_data`'s `session` is `None` when the previous hop fans out |
| Downstream | At or after the creator, not the one opening the id | `session.put_data` to forward, or nothing if this hop consumes the frame | `on_data(msg, session)` with this hop's next dest |

Fan-out hops that sit between the source and a module creator (chain mode, `id` lower than the creator) listen, and they emit with `fanout_data`. They cannot call `create_session()`. Only the creator id is allowed to open a session. If no module sets `session-creator`, the hub is the creator: there is no fan-out, and every module `create_session()` is rejected.

## Source

The source has no data listen socket. When it sits below the session creator, the hub sends `add-dests` and frames go out with `fanout_data`. If that dest list is empty, the SDK drops the frame and logs it.

```python
class AudioIn(Module):
    async def on_run(self):
        await self.ready()
        while True:
            chunk = read_microphone()          # bytes, one PCM frame
            await self.fanout_data(chunk)
```

A chain-mode source whose `id` is below the session creator uses `fanout_data` only. That hop is not allowed to open a session. When the source itself is the session creator, or when `hub.chain` is false, the source calls `create_session()` and sends with `put_data`.

## Between the source and the creator

VAD is the usual hop in that gap. It listens, and it still fans out. `on_data` gets `session=None` because audio-in sent frames with no `session_id`. Forward with `fanout_data`. This hop cannot call `create_session()`.

Modules in this gap might set flow events on the frames they fan out (`flow_id` and `data["event"]`). The hub does not use those fields.

```python
class Vad(Module):
    async def on_data(self, msg, session):
        if not self._speech:
            return
        await self.fanout_data(msg.payload)
```

## Session creator

When a hub row sets `session-creator: true`, that id is the creator (sample configs often use wake word). If no row sets it, the hub is the creator and modules cannot call `create_session()`.

`create_session()` does not return until every other ready session module has acked the new id. The returned `Session` is **this** module's next hop.

```python
class Wakeword(Module):
    async def on_data(self, msg, session):
        if not spotted(msg.payload):
            return
        if self._session is None:
            self._session = await self.create_session()
        if self._session is None:
            return
        await self._session.put_data(msg.payload)

    async def end_spot(self):
        if self._session is not None:
            await self._session.finish()
            self._session = None
```

`on_data`'s `session` argument is `None` here, because the previous hop fanned out frames with no `session_id`. Forward with the handle from `create_session()`, not with that argument.

A second `create_session()` allocates a new id. That is a new stream (for example TTS playback), not a way to keep the current one.

## Downstream

Everyone after wake word reuses the inbound id. The `session` argument is this module's next hop for that id, which is a different object from wake word's handle even though `session_id` matches.

```python
class Stt(Module):
    async def on_data(self, msg, session):
        if session is None:
            return
        text = transcribe(msg.payload)
        if text:
            await session.put_data(text)
```

If this module is the sink, `session.dest` is `None` and `put_data` does nothing. Consume the frame and return.

A frame whose `session_id` was never announced is dropped. `on_data` is not called. That is why `create_session()` waits for the other modules before it returns.

## Payload shape

`fanout_data` and `Session.put_data` accept:

| Input | Frame on the wire |
| --- | --- |
| `bytes` | `type="audio"` |
| `str` | `type="text"` (UTF-8 payload) |
| `Message` | forwarded; `session_id` is filled in when the call is sessioned and the message has none |
| sync or async iterable of those | one frame per item |

`on_data` receives the ipc-lib `Message`. Audio is `msg.payload`. The session id is `msg.session_id`, and also `session.session_id` when the handle is present.

## Lifecycle sketch

```python
class Stt(Module):
    async def on_init(self):
        self.model = load_model()
        return True

    async def on_run(self):
        await self.ready()
        await asyncio.Event().wait()       # keep the task alive; data arrives in on_data

    async def on_data(self, msg, session):
        text = self.model.push(msg.payload)
        if text and session is not None:
            await session.put_data(text)

    async def on_stop(self):
        self.model.close()
        return True
```

`run()` starts `on_run` as a background task and waits on the control loop. `ready()` awaits `on_ready()` inside that task before the module is marked ready. The `await asyncio.Event().wait()` above keeps the background task alive after `ready()` returns. Work that must finish before the hub is told this module is ready belongs in `on_ready`.

`on_init` runs again after a hub reconnect, because `run()` treats each connection as a new init. Put one-time process setup there only if repeating it is safe, or guard it yourself.

`send()` returns `None` at once when the local ready flag is unset. `ready()` sets that flag after `on_ready()` and before the hub acks `ready`. `create_session()` and `finish()` send immediately; the hub nacks `create-session` unless this module is already `ready`. Register and the `ready` message use that same immediate path.
