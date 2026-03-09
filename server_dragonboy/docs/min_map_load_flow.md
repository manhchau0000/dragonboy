# Minimal Map Load Flow

## Scope
- This is the minimum compatible flow to enter a map after successful login.
- It covers template/resource requests, map snapshot, and finish-load barrier.

## Exact Packet Sequence

Precondition:
- Client already passed login flow and has XOR active.

1. Optional template/resource sync requests (recommended minimum set)

1.1 `C->S` `-28` action `6` (`ACTION_UPDATE_MAP`)
- Payload: `byte 6`.
- Expected response: `S->C -28(action=6)` with map/npc/mob template arrays.
- References: `src/server/Controller.java:562`, `src/network/SessionService.java:94-125`.

1.2 `C->S` `-28` action `7` (`ACTION_UPDATE_SKILL`)
- Payload: `byte 7`.
- Expected response: `S->C -28(action=7)` with class/skill templates.
- References: `src/server/Controller.java:563`, `src/network/SessionService.java:127-201`.

1.3 `C->S` `-28` action `8` (`ACTION_UPDATE_ITEM`)
- Payload: `byte 8`.
- Expected responses: multiple `S->C -28(action=8)` blocks:
  - mode `0` item options
  - mode `100` arr head2frame
  - mode `1` initial item template block
  - mode `2` remaining item template block
- References: `src/server/Controller.java:564`, `src/item/ItemData.java:12-17`, `19-112`.

1.4 `C->S` `-28` action `10` (`ACTION_MAP_TEMP`) with map id
- Payload fields:
  - `byte 10`
  - `unsigned byte mapId`
- Expected response: `S->C -28(action=10)` + raw tile-map bytes for that `mapId`.
- References: `src/server/Controller.java:565`, `src/network/SessionService.java:342-357`.

2. In-map init request
- `C->S` `-28` action `13` (`ACTION_INIT_INFO`)
- Payload: `byte 13`.
- Reference: `src/server/Controller.java:566-568`.

3. Server sends player/map init outputs
- Includes:
  - player/core sync packets (`Service.player`, stats/flags/tasks/etc.)
  - `S->C -24` map snapshot via `zone.mapInfo(...)`
- References: `src/server/Controller.java:575-608`, `src/map/Zone.java:510-541`.

4. Finish-load barrier
- After local map snapshot is applied, send `C->S -39`.
- Payload: none.
- Reference: `src/server/Controller.java:266`.

5. Post-load sync responses
- Server runs `ChangeMapService.finishLoadMap(...)`, sending me/others/effect finalization packets.
- Reference: `src/services/map/ChangeMapService.java:496-549`.

## Payload Fields (Minimum Required)

- `-28(action=10)` request: `byte action=10`, `unsigned byte mapId`.
- `-24` map snapshot parser must handle at least:
  - map/zone metadata
  - local spawn `x/y`
  - waypoints
  - mobs
  - npcs
  - items
  - bg/effect blocks
- References: `src/map/Zone.java:510-620`.

## Expected Server Responses

Required for successful map entry:
1. `-24` map snapshot.
2. Post `-39`, additional entity/effect synchronization from `finishLoadMap`.

Supporting responses (during template sync):
- `-28(action=6/7/8/10)` template blocks.

## Local Client State Transitions

1. `AUTH_SUCCESS_PENDING_INIT` -> `TEMPLATE_SYNC` (while processing action `6/7/8/10` responses).
2. `TEMPLATE_SYNC` -> `INIT_INFO_SENT` (after `C->S -28(action=13)`).
3. `INIT_INFO_SENT` -> `MAP_SNAPSHOT_READY` (after parsing `S->C -24`).
4. `MAP_SNAPSHOT_READY` -> `FINISH_LOAD_SENT` (after `C->S -39`).
5. `FINISH_LOAD_SENT` -> `IN_WORLD_ACTIVE` (after finish-load sync messages settle).

## Failure Points

1. Skipping `-28(action=13)` prevents `initPlayerInfo` and map snapshot path (`src/server/Controller.java:566-568`, `575-592`).
2. Incorrect `-24` parser causes map desync or crash (`src/map/Zone.java:510-620`).
3. Sending gameplay packets before `-39` can race map/effect sync (`src/services/map/ChangeMapService.java:496-549`).
4. Missing/incorrect map temp (`action=10`) can break local collision/render setup.
5. Not handling `clearMap(-22)` in surrounding init/change-map flows may leave stale entities (`src/services/Service.java:342-349`, `src/services/map/ChangeMapService.java:377`).

## Assumptions vs Confirmed Behavior

Confirmed:
- `-28` actions `6/7/8/10/13` are wired exactly in controller.
- `action=8` response is chunked into multiple `-28` payload variants.
- `-39` triggers `finishLoadMap(...)`.

Assumptions:
- Request order among `6/7/8/10` is flexible as long as required data is ready before gameplay.
- A minimum compatibility client may ignore non-critical init packets, but must parse `-24` and complete `-39` barrier.
