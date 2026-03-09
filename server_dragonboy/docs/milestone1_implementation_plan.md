# Milestone 1 Implementation Plan

## Scope
Implement only the minimum compatibility path:
- transport
- handshake
- login
- template/init
- map snapshot parse
- movement loop

Do not implement full combat, inventory, NPC systems, or full UI/UX.

## Phase A: Transport + Handshake

### Classes to implement
- `NetSocketTransport`
- `PacketReaderLoop`
- `PacketWriterLoop`
- `PacketCodec`
- `XorCodecState`
- `HandshakeController`
- `PacketTraceLogger`

### Packet IDs to support
- `-27` handshake trigger/response
- transport support for special-length cmds: `-32,-66,-74,11,-67,-87,66`

### Success criteria
1. Client connects and sends `-27`.
2. Client parses `S->C -27` key payload.
3. XOR mode activates and subsequent packets decode correctly.

### Debug output required
- connect/disconnect logs
- packet traces for `-27`
- XOR mode switch log with key length
- read/write key index progression snapshots (coarse-grained)

## Phase B: Login

### Classes to implement
- `LoginFlowController`
- `AuthModel`
- `ClientInfoPacketBuilder`
- `LoginPacketBuilder`

### Packet IDs to support
- Outgoing: `-29(action=2)`, `-29(action=0)`
- Incoming: `-29(action=2)`, `-26`, `122`, `-102`, `2`, `-77`, `-93`, `-28(action=4)`, `-31`

### Success criteria
1. Valid account reaches `AUTH_SUCCESS_PENDING_INIT`.
2. Invalid account reaches `AUTH_REJECTED` with visible reason.
3. New account reaches `CREATE_CHAR_REQUIRED`.

### Debug output required
- auth state transitions
- login request/response packet trace lines
- reason-tagged auth failure logs

## Phase C: Template/Init Flow

### Classes to implement
- `InitSyncController`
- `TemplateSyncTracker`
- `ItemTemplateParser`
- `MapTemplateParser`
- `SkillTemplateParser`

### Packet IDs to support
- Outgoing: `-28(action=6)`, `-28(action=7)`, `-28(action=8)`, `-28(action=10,mapId)`, `-28(action=13)`
- Incoming: `-28(action=6/7/8/10)`

### Success criteria
1. Client can complete minimal template sync and send `action=13`.
2. Item update chunk modes (`0/100/1/2`) are parsed without crash.

### Debug output required
- per-action request/response logs
- template block counters and completion summary
- unknown sub-mode warnings (non-fatal)

## Phase D: Map Snapshot Parse

### Classes to implement
- `MapSnapshotParser`
- `MapSnapshotModel`
- `EntityRegistry`
- `MapLoadController`

### Packet IDs to support
- Incoming: `-24` map snapshot, `-22` clear map
- Outgoing: `-39` finish-load

### Success criteria
1. `-24` is parsed into map + entity registries.
2. Client reaches `MAP_SNAPSHOT_READY`, sends `-39`, then reaches `IN_WORLD_ACTIVE`.

### Debug output required
- map id/zone/local spawn logs
- entity counts (mobs/npcs/items)
- clear-map reset logs

## Phase E: Movement Loop

### Classes to implement
- `MovementSyncController`
- `LocalPlayerController`
- `RemotePlayerUpdater`

### Packet IDs to support
- Outgoing: `-7` movement
- Incoming: `-7` remote movement, `46` correction/reset

### Success criteria
1. Local move requests are sent only in `IN_WORLD_ACTIVE`.
2. Remote movement updates are applied.
3. Corrections are applied without desync.
4. Movement-triggered map loading transitions to `MAP_LOADING` and back through Phase D path.

### Debug output required
- outgoing move traces (x,y,b)
- incoming remote move traces (id,x,y)
- correction application logs
- blocked-move logs (state invalid/dead/effect-lock inferred from server behavior)

## Cross-Phase Quality Gates

1. State-machine enforcement is enabled from Phase A onward.
2. Packet codec tests run with captured packet fixtures.
3. No rendering class imports protocol codec classes directly.
4. No protocol class references libGDX rendering APIs.
