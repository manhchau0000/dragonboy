# Minimal libGDX Client Architecture

## Scope
This architecture is derived from:
- `docs/minimum_client_spec.md`
- `docs/min_login_flow.md`
- `docs/min_map_load_flow.md`
- `docs/min_movement_flow.md`

Goal: minimal compatible client that can connect, authenticate, initialize map data, enter world, and sync movement.

## Design Constraints
1. Keep protocol, state, and rendering separated.
2. Do not implement full gameplay logic client-side.
3. Server remains authoritative for movement/combat/inventory/map transitions.

## Suggested Package Layout

- `client.app`
- `client.state`
- `client.net.transport`
- `client.net.codec`
- `client.net.packet`
- `client.flow`
- `client.world`
- `client.screen`
- `client.debug`

## Module/Class Design

| Module | Core classes | Responsibility | Depends on |
|---|---|---|---|
| Socket transport | `NetSocketTransport` | Open/close TCP, read/write raw bytes, reconnect hooks | none |
| Packet reader | `PacketReaderLoop` | Pull bytes from transport, decode frame, emit `InboundPacket` | `NetSocketTransport`, `PacketCodec` |
| Packet writer | `PacketWriterLoop` | Accept `OutboundPacket`, encode frame, send bytes | `NetSocketTransport`, `PacketCodec` |
| XOR codec state | `XorCodecState` | Hold key bytes, read/write indices, mode flag (`plaintext/xor`) | none |
| Handshake flow | `HandshakeController` | Send `-27`, parse key response `-27`, activate XOR | `ClientStateMachine`, `PacketWriterLoop`, `XorCodecState` |
| Login flow | `LoginFlowController` | Send client-info `-29(action=2)` and login `-29(action=0)`, detect success/failure/create-char | `ClientStateMachine`, `PacketRouter` |
| Init/template flow | `InitSyncController` | Send `-28` actions `6/7/8/10/13`, track template completion | `ClientStateMachine`, `PacketWriterLoop` |
| Map snapshot parser | `MapSnapshotParser` | Parse `-24` payload into immutable map snapshot DTO | `PacketReader` primitives |
| Entity registry | `EntityRegistry` | Hold current map entities (players/mobs/npcs/items), reset on `-22`/map load | `MapSnapshotParser` |
| Movement sync | `MovementSyncController` | Send `-7`, apply corrections (`46`), process remote move `-7` | `EntityRegistry`, `ClientStateMachine` |
| Screen/state management | `ScreenRouter`, `ConnectionScreen`, `LoginScreen`, `LoadingScreen`, `WorldScreen` | UI routing driven by client state machine events | `ClientStateMachine` |
| Debug logging | `PacketTraceLogger`, `StateTraceLogger` | Structured logs for packet I/O and state transitions | all modules |

## Protocol/State/Rendering Separation

### Protocol layer (`client.net.*`)
- Handles signed-byte command IDs, frame lengths, XOR mode, packet serialization.
- Has no libGDX rendering or gameplay decisions.

### State/flow layer (`client.state`, `client.flow`)
- Owns lifecycle states and transitions (connect, auth, init, in-world).
- Decides which packets may be sent in each state.
- Consumes decoded packets and emits domain events.

### Rendering layer (`client.screen`, `client.world`)
- Renders based on read-only snapshots from state/registry.
- Does not directly parse bytes or mutate protocol state.

## Concrete Class Responsibilities

### `NetSocketTransport`
- `connect(host, port)`
- `disconnect(reason)`
- `readFully(...)`
- `write(...)`
- Emits transport events (`connected`, `disconnected`, `io_error`).

### `PacketCodec`
- `decodeFrame(InputStream in, XorCodecState xorState): InboundPacket`
- `encodeFrame(OutboundPacket pkt, OutputStream out, XorCodecState xorState)`
- Enforces 2-byte vs special 3-byte length rules.

### `PacketRouter`
- Routes by command and current `ClientState`.
- Forwards to:
  - `HandshakeController`
  - `LoginFlowController`
  - `InitSyncController`
  - `MapLoadController`
  - `MovementSyncController`

### `ClientStateMachine`
- Holds current state from `docs/client_state_machine.md`.
- Validates allowed outgoing packets by state.
- Records transition reason and last packet context.

### `WorldSession`
- Minimal runtime session object:
  - connection info
  - codec mode + key state
  - auth flags
  - map/zone id
  - local player id/position
  - template sync flags

## Event-Driven Flow Wiring

1. `AppBootstrap` starts `NetSocketTransport`.
2. `PacketReaderLoop` decodes packets -> `PacketRouter`.
3. Flow controllers update `ClientStateMachine` + `WorldSession`.
4. `ScreenRouter` listens to state changes and switches screens.
5. `PacketWriterLoop` is fed by flow controllers only (never directly by screens).

## Safety Rules

1. Never send gameplay packets before `FINISH_LOAD_SENT` -> `IN_WORLD_ACTIVE`.
2. On `-22` clear-map, hard-reset `EntityRegistry` immediately.
3. On decode error/XOR desync, force transition to `DISCONNECTED`.
4. Treat server packets as authoritative corrections.

## Minimal Runtime Threads/Loops

- `PacketReaderLoop` (single thread)
- `PacketWriterLoop` (single thread)
- libGDX render thread
- optional background logger flush thread

No gameplay simulation thread is required for milestone 1.
