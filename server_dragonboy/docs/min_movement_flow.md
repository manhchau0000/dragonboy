# Minimal Movement Flow

## Scope
- This is the minimum movement flow once client is in `IN_WORLD_ACTIVE` state.
- Server is authoritative for final movement legality and map-transition side effects.

## Exact Packet Sequence

Preconditions:
- Client completed map load and sent `-39`.
- Player is not dead and not effect-locked.

1. `C->S` movement request `-7`
- Payload fields:
  - `byte b`
  - `short toX`
  - `short toY`
- Reference: `src/server/Controller.java:450-453`.

2. Server-side validation path
- `Controller.handleMove(...)` checks:
  - dead -> triggers death packet flow and returns
  - effect skill lock -> return
  - BlackBall maps: distance > 500 -> return
- References: `src/server/Controller.java:441-457`.

3. Authoritative movement apply
- `PlayerService.playerMove(...)` mutates `player.location.x/y`, applies map-specific rules and movement side effects.
- Reference: `src/services/PlayerService.java:142-198`.

4. Server broadcast (others)
- `S->C -7` to other players in map:
  - `int playerId`
  - `short x`
  - `short y`
- Reference: `src/services/map/MapService.java:420-427`.

## Expected Server Responses

Normal move:
- Other clients receive `-7` broadcast.
- Mover does not receive a dedicated `-7` echo from this path.

Special outcomes:
- If move invalid in certain contexts, request is ignored (no success ack).
- If map constraints trigger correction/change:
  - local reset can be sent via cmd `46` (`resetPoint`).
  - forced map change path may emit `-22` clear map + `-24` map snapshot and require new `-39`.
- References: `src/services/Service.java:327-336`, `src/services/PlayerService.java:156-175`, `src/services/map/ChangeMapService.java:377-382`, `496-549`.

Dead-move attempt:
- Server sends death flow packets (`charDie`) instead of moving.
- Reference: `src/server/Controller.java:441-444`, `src/services/Service.java:760-785`.

## Local Client State Transitions

1. `IN_WORLD_ACTIVE` -> `MOVE_REQUEST_SENT` (on local input).
2. Optional local prediction: `MOVE_REQUEST_SENT` -> `LOCAL_PREDICTED`.
3. If no correction/map change: return to `IN_WORLD_ACTIVE`.
4. If reset packet (`46`) received: `LOCAL_PREDICTED` -> `POSITION_CORRECTED` -> `IN_WORLD_ACTIVE`.
5. If map change triggered by movement: `IN_WORLD_ACTIVE` -> `MAP_LOADING` and follow map-load flow again.

## Failure Points

1. Sending movement while dead/effect-locked causes no move acceptance (`src/server/Controller.java:441-445`).
2. Large teleport deltas in BlackBall maps are dropped (`src/server/Controller.java:454-457`).
3. Not handling correction packet `46` leads to client/server position divergence (`src/services/Service.java:327-336`).
4. Not handling movement-triggered map transition leads to soft lock/desync.
5. Misreading `byte b` can break feature behavior tied to fly-task checks (`src/server/Controller.java:458-460`).

## Assumptions vs Confirmed Behavior

Confirmed:
- Request opcode/payload is `-7` with `(byte, short, short)`.
- Server enforces legality and applies authoritative location.
- Broadcast to others uses `-7(int id, short x, short y)`.
- In BlackBall maps, >500 distance is rejected.

Assumptions:
- Local prediction is optional for compatibility (not required by server).
- `b==1` likely represents a flight-related movement mode (only confirmed usage is achievement check).
