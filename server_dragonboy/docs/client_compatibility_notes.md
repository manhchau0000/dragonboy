# Client Compatibility Notes (Phase 5)

## Goal
- Summarize exact behaviors a new libGDX client must replicate to stay compatible with this server.
- This is protocol/flow compatibility guidance only (no server refactor).

## 1) Implement transport exactly
- Message frame: `cmd byte + length + payload` (`src/network/MessageSendCollect.java:17-40`).
- Pre-key mode: 2-byte unsigned length (`src/network/MessageSendCollect.java:27-29`).
- Post-key mode: XOR decode/encode `cmd`, `length`, and payload bytes with rolling key index (`src/network/MessageSendCollect.java:20-39`, `53-84`).
- Must support special outbound 3-byte length handling for commands `-32, -66, -74, 11, -67, -87, 66` (`src/network/MessageSendCollect.java:64-70`).

## 2) Follow handshake/session ordering
- Be ready to send/receive `-27` key flow (`src/network/Collector.java:36-38`, `src/network/KeyHandler.java:9-20`).
- Send `-29` action `2` client info before login action `0` (required for version/client metadata path) (`src/server/Controller.java:544-550`, `src/services/Service.java:1514-1531`).
- Client-info payload format must match server reads exactly:
  - `byte typeClient, byte zoomLevel, boolean, int, int, boolean, boolean, UTF platform` (`src/services/Service.java:1516-1525`).

## 3) Respect channel semantics (`-29`, `-28`, `-30`)
- `-29`: pre-login only; includes login and client-info actions (`src/server/Controller.java:544-553`).
- `-28`: pre-map updates and init info actions (`src/server/Controller.java:556-569`).
- `-30`: subcommands such as stat-point and menu (`src/server/Controller.java:610-623`).

## 4) Resource/data update compatibility
- Support full update exchanges:
  - map templates (`-28` action `6`) (`src/network/SessionService.java:94-125`)
  - skill templates (`-28` action `7`) (`src/network/SessionService.java:127-201`)
  - map tile data (`-28` action `10`) (`src/network/SessionService.java:342-357`)
  - data pack (`-87`) (`src/network/SessionService.java:63-92`)
  - resource stream (`-74` types 0/1/2/3) (`src/network/SessionService.java:385-452`).

## 5) Reproduce runtime packet order for map join
- After login attach, process init packets from `Controller.sendInfo(...)` (`src/server/Controller.java:675-703`).
- Send `-28` action `13` to trigger `initPlayerInfo(...)` (`src/server/Controller.java:566-568`, `575-608`).
- Parse `-24` map snapshot correctly (waypoints/mobs/npcs/items/bg/effects) (`src/map/Zone.java:510-620`).
- Send `-39` finish-load when map assets/entities are ready (`src/server/Controller.java:266`).

## 6) Implement gameplay request packet shapes exactly
- Move request `-7`: `byte b, short toX, short toY` (`src/server/Controller.java:450-453`).
- Skill request `-45`:
  - always starts with `byte status`
  - for `status==20`, include `byte skillId, short dx, short dy, byte dir, short x, short y`
  (`src/services/SkillService.java:59-67`).
- Attack mob `54`: `byte mobId`; if `mobId==-1`, append `int masterId` (`src/server/Controller.java:503-507`).
- Attack player `-60`: `int targetId` (`src/server/Controller.java:272`).
- Pick item `-20`: `short itemMapId` (`src/server/Controller.java:273-275`).

## 7) Parse authoritative server outputs (do not re-simulate)
- Server move broadcast: `-7` (`src/services/map/MapService.java:420-427`).
- Item spawn/pick/disappear: `68`, `-19`, `-21` (`src/services/Service.java:1182-1192`, `992-999`, `src/services/map/ItemMapService.java:35-41`).
- Death/combat/map effect packets come from server logic; client should display, not decide outcomes.

## 8) Match state assumptions that affect compatibility
- One account maps to one active player entry in this flow (`src/player/PlayerDAO.java:1050-1061`).
- Re-login may kick older live session (`src/player/PlayerDAO.java:1041-1045`, `1056-1059`).
- Server enforces rules such as cooldown/mana/movement limits; client should not assume local authority (`src/services/SkillService.java:999-1033`, `src/services/PlayerService.java:142-198`).

## 9) Compatibility checklist for a new libGDX client
- Implement signed-byte opcode handling and channel routing.
- Implement key-handshake timing and post-key XOR stream behavior.
- Implement special 3-byte length commands.
- Implement exact field order/types for login/client-info/move/skill/attack/pick packets.
- Implement map join sequence ending with `-39` finish-load.
- Treat server as sole authority for position, combat, inventory, and drops.

## Inferred conclusions
- `Inferred:` biggest breakage risks are binary framing details and packet order, not rendering.
- `Inferred:` a compatibility client should prioritize protocol correctness first, then visual parity.
