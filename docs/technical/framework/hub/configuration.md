# Configuration

The hub reads one YAML file. `parse_config` checks names, sorts the chain, and rejects a chain-mode file that does not name exactly one session-creator id.

`modules` may be a list, which is the chain, or a mapping:

```yaml
hub:
  modules:
    listen: unix:///run/fosia/hub.control.sock
  chain: true

modules:
  chain:
    - id: 1
      name: audio-in
      where: local
      buffer: false
    - id: 2
      name: vad
      where: local
      buffer: false
    - id: 3
      name: wakeword
      session-creator: true
      where: local
      buffer: false
    - id: 4
      name: stt
      where: local
      buffer: true
  audio-transmission:
    - name: webrtc-client
      priority: 1
      buffer: false
```

A bare list is accepted too. After parsing, `modules` is always the chain list and `audio-transmission` is always a list (empty if omitted).

## Fields the control plane uses

| Field | Where | Meaning |
| --- | --- | --- |
| `hub.modules.listen` | hub | Control socket. Default `unix:///run/fosia/hub.control.sock` |
| `hub.chain` | hub | `true` selects chain mode. Omitted or `false` means every module uses sessions |
| `id` | chain row | Sort key. Also compared with the session-creator id |
| `name` | any module row | Unique across the chain and `audio-transmission`. A duplicate is a startup error |
| `session-creator` | chain row | In chain mode, exactly one distinct `id` may set this. Several rows may share that id |
| `where` | chain row | Placement label. A missing value counts as `local` |
| `buffer` | any module row | Copied onto the register ack. The module keeps frames for replay only when this is true |
| `priority` | audio-transmission row | Try order. Lowest number first. Missing priorities sort last |

`buffer` comes from this file, not from the module's own `module.yaml`. Hot audio hops usually set `buffer: false` so old PCM is not replayed into a newly connected neighbor.

Sample configs also carry `when` and a chain-row `priority`. The running control plane does not branch on those.

## Chain mode

`hub.chain: true` requires `session-creator: true` on modules that share a single `id`. Zero creator ids, or two different ones, fail at startup. A creator row must have an `id`.

```text
id 1  audio-in     source, fan-out     no listen socket
id 2  vad          fan-out             listens, sends with no session_id
id 3  wakeword     session-creator     default creator; listens, may create-session
id 4  stt          session module      listens, reuses inbound session ids
```

Only modules whose `id` equals the creator id may send `create-session`. Modules with a lower id receive `add-dests` and do not take part in session announce. Modules at or after that id receive `add-sessions`.

Audio-transmission rows are not chain rows. They have no `id`, they are not the source, they are not session creators, and they do not fan out.

With `hub.chain: false`, fan-out is off. Any ready module may create a session, including the source, except an audio-transmission row. The source still does not listen.

## Next hop

The default next hop for chain module N is the listen address of N+1, and only if that process is connected and listening. A row that is not up means `dest` is `None` for the previous hop. The config lists the order; live sockets fill the addresses.

If N and N+1 have different `where` values, N does not use N+1. It uses the first audio-transmission method that is connected and listening, in `priority` order. If none is listening, that hop is `None`.

```text
device hub                         server hub
audio-in (where: local)
    │
    ▼
vad (where: local)
    │  where changes
    ▼
webrtc-client  ── media, not the hub ──►  server pipeline
    │
    ▼
later local modules are skipped for this hop
```

Those transmission modules register and listen like any other module. The hub still does not carry the audio. It only names the listen address the previous hop should connect to.

## Peers block

A peer mesh is enabled when `hub.peers.listen` is set or any `peers` row has `connect`. See [Peers](peers.md). Without that block, `hub.peers` is `None` and the `websockets` extra is not required.
