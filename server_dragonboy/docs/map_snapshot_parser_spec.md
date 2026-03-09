# Map Snapshot Parser Spec (`S->C -24`)

## Scope
This spec defines the exact parse order for the server map snapshot packet (`cmd = -24`) as written by server code.

Primary references:
- `src/map/Zone.java`:
  - `Zone.mapInfo(Player)`
  - `Zone.writeWayPoints(Message)`
  - `Zone.writeMobs(Message)`
  - `Zone.writeNpcs(Player, Message)`
  - `Zone.writeItems(Player, Message)`
  - `Zone.writeBackground(Message)`
  - `Zone.writeEffect(Message)`
- `src/services/map/NpcManager.java`:
  - `NpcManager.getNpcsByMapPlayer(Player)`

Opcode reference:
- `CMD_MAP_INFO = -24` in `src/map/Zone.java:48`.

## Packet Section Boundaries (in write order)

1. Header section (fixed primitives)
2. Waypoints section (counted list)
3. Mobs section (counted list)
4. Reserved separator byte
5. NPC section (counted list)
6. Item section (counted list)
7. Background block (raw bytes)
8. Effect block (raw bytes)
9. Tail section (fixed primitives)

Source order: `Zone.mapInfo` in `src/map/Zone.java:510-535`.

## Exact Primitive Field Order

## 1) Header section
Written by `Zone.mapInfo` (`src/map/Zone.java:513-521`):
1. `byte mapId`
2. `byte planetId`
3. `byte tileId`
4. `byte bgId`
5. `byte mapType`
6. `UTF mapName`
7. `byte zoneId`
8. `short localSpawnX`
9. `short localSpawnY`

Notes:
- `localSpawnX/localSpawnY` are the local player spawn/entry fields for this snapshot.

## 2) Waypoints section
Written by `Zone.writeWayPoints` (`src/map/Zone.java:543-555`):
1. `byte wayPointCount`
2. Repeat `wayPointCount` times:
- `short minX`
- `short minY`
- `short maxX`
- `short maxY`
- `boolean isEnter`
- `boolean isOffline`
- `UTF name`

Fallback behavior:
- On writer exception, server writes `byte 0` count (`src/map/Zone.java:556-558`).

## 3) Mobs section
Written by `Zone.writeMobs` (`src/map/Zone.java:561-585`):
1. `byte mobCount`
2. Repeat `mobCount` times:
- `boolean flag1` (currently always `false`)
- `boolean flag2` (currently always `false`)
- `boolean flag3` (currently always `false`)
- `boolean flag4` (currently always `false`)
- `boolean flag5` (currently always `false`)
- `byte mobTempId`
- `byte unknownByte0` (currently always `0`)
- `int currentHp`
- `byte mobLevel`
- `int maxHp`
- `short x`
- `short y`
- `byte status`
- `byte lvMob`
- `boolean specialMobFlag`

Where `specialMobFlag = (tempId == GAU_TUONG_CUOP) || (tempId in [VOI_CHIN_NGA..PIANO])` (`src/map/Zone.java:584`).

Mob inclusion filter before writing:
- Dead big-boss mobs are excluded except `MOB_HIRUDEGARN_PART` (`src/map/Zone.java:563-567`, constant at `src/map/Zone.java:50`).

Fallback behavior:
- On writer exception, server writes `byte 0` count (`src/map/Zone.java:586-588`).

## 4) Reserved separator byte
Written directly in `Zone.mapInfo` (`src/map/Zone.java:525`):
- `byte reservedAfterMobs = 0`

## 5) NPC section
Written by `Zone.writeNpcs` (`src/map/Zone.java:591-601`):
1. `byte npcCount`
2. Repeat `npcCount` times:
- `byte status`
- `short cx`
- `short cy`
- `byte tempId`
- `short avatar`

NPC list is pre-filtered via `NpcManager.getNpcsByMapPlayer` (`src/map/Zone.java:593`, `src/services/map/NpcManager.java:33-48`) with conditions such as task/power and egg visibility.

Fallback behavior:
- On writer exception, server writes `byte 0` count (`src/map/Zone.java:602-604`).

## 6) Item section
Written by `Zone.writeItems` (`src/map/Zone.java:607-617`):
1. `byte itemCount`
2. Repeat `itemCount` times:
- `short itemMapId`
- `short itemTemplateId`
- `short x`
- `short y`
- `int ownerPlayerId`

Item list is pre-filtered via `Zone.getItemMapsForPlayer` (`src/map/Zone.java:609`, `221-230`) by task and owner conditions.

Fallback behavior:
- On writer exception, server writes `byte 0` count (`src/map/Zone.java:618-620`).

## 7) Background block
Written by `Zone.writeBackground` (`src/map/Zone.java:623-629`):
- Server writes raw bytes from file: `data/map/item_bg_map_data/<mapId>`.
- No explicit length is written in `mapInfo`; length must be derived from embedded block format.

Fallback behavior:
- On writer exception, server writes `short 0` (`src/map/Zone.java:628`).

## 8) Effect block
Written by `Zone.writeEffect` (`src/map/Zone.java:632-638`):
- Server writes raw bytes from file: `data/map/eff_map/<mapId>`.
- No explicit length is written in `mapInfo`; length must be derived from embedded block format.

Fallback behavior:
- On writer exception, server writes `short 0` (`src/map/Zone.java:637`).

## 9) Tail section
Written by `Zone.mapInfo` (`src/map/Zone.java:532-534`):
1. `byte bgType`
2. `byte idSpaceShip`
3. `byte lycheeCastleFlag` where `1` if `mapId == LAU_DAI_LYCHEE`, else `0`

## Optional / Conditional Blocks and Conditions

1. Mob list content is conditional (dead big-boss filtering).
2. NPC list content is conditional by player/task/power via `NpcManager.getNpcsByMapPlayer`.
3. Item list content is conditional by player/task/owner via `Zone.getItemMapsForPlayer`.
4. Background/effect are conditional on file availability; fallback sentinel `short 0` may appear.

## Parser Guidance (Minimal Compatible)

1. Parse header, waypoints, mobs, separator, npcs, items in strict sequence.
2. Treat all count bytes as unsigned (`0..255`) in client parser.
3. For background/effect blocks, delegate to existing map asset block parser (length comes from internal block format, not outer packet).
4. Parse final 3 tail bytes after background/effect parser consumes exactly each block.

## Fields Still UNKNOWN

1. `mobs.flag1..flag5` exact semantics (`src/map/Zone.java:570-574`).
2. `mobs.unknownByte0` exact meaning (`src/map/Zone.java:576`).
3. `reservedAfterMobs` byte purpose (`src/map/Zone.java:525`).
4. Internal binary schema of `background` block bytes (`src/map/Zone.java:625-626`).
5. Internal binary schema of `effect` block bytes (`src/map/Zone.java:634-635`).
6. Whether `mapId/planetId/tileId/bgId/type/zoneId/bgType/idSpaceShip` should always be treated as unsigned on all maps (server writes `byte`; interpretation on client may vary by implementation).

## Confirmed vs Assumed

Confirmed:
- Primitive write order and section order above are exact from `Zone.mapInfo` and helpers.
- Waypoint/mob/npc/item sections are count-prefixed.
- Local player spawn fields are `short x/y` in header.

Assumed:
- Existing client block parsers for background/effect are required to delimit those raw segments correctly.
- Unknown flags/bytes do not block minimal render/movement compatibility if preserved/parsing-safe.
