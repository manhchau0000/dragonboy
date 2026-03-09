# Protocol Reverse Engineering (Phase 2)

## Scope
- This is a behavior-preserving protocol map from server code.
- It focuses on packet framing, dispatch, opcode channels, directions, and core payloads.
- `Inferred:` marks conclusions that combine multiple code paths.

## Transport framing
- A packet is read as: `command byte` + `length` + `payload` (`src/network/MessageSendCollect.java:17-40`).
- Before key handshake (`session.sentKey() == false`), length is `unsigned short` (`src/network/MessageSendCollect.java:27-29`).
- After key handshake, command/length/payload bytes are XOR-decoded with rolling key state (`src/network/MessageSendCollect.java:20-39`, `44-50`).
- Outbound write mirrors this (XOR encode after handshake) (`src/network/MessageSendCollect.java:53-84`, `95-101`).

### Special 3-byte outbound length mode
- For commands `-32, -66, -74, 11, -67, -87, 66`, server writes 3 encoded size bytes (`src/network/MessageSendCollect.java:64-70`).
- Other commands use 2-byte length (encoded if key is active) (`src/network/MessageSendCollect.java:71-78`).
- `Inferred:` a compatible client must implement this special length rule exactly for those command IDs.

## Key handshake and encryption
1. Client sends command `-27` to trigger key exchange handling (`src/network/Collector.java:36-38`, `src/server/Controller.java:186-189`).
2. Server replies with command `-27` containing key stream seed data (`src/network/KeyHandler.java:9-20`).
3. `MyKeyHandler` additionally sends image/resource version notifications (`src/network/MyKeyHandler.java:15-19`).
4. `session.setSentKey(true)` enables XOR transport mode (`src/network/KeyHandler.java:20`, `src/network/MessageSendCollect.java:20-39`).

## Dispatch pipeline
- Socket accept -> `Network.run()` -> session clone (`src/network/Network.java:107-120`).
- Per-session receive loop: `Collector.run()` (`src/network/Collector.java:32-42`).
- Non-handshake packets call `Controller.onMessage(...)` (`src/network/Collector.java:39`, `src/server/Controller.java:149`).
- Controller split:
  - Pre-login/pre-map/system: `handleSessionCommands(...)` (`src/server/Controller.java:176-199`).
  - In-game: `handlePlayerCommands(...)` (`src/server/Controller.java:201-288`).

## Opcode channels and meaning

### High-level channels
- `-29` (`CMD_NOT_LOGIN`): pre-login actions (`src/server/Controller.java:111`, `544-553`).
- `-28` (`CMD_NOT_MAP`): pre-map/update/init actions (`src/server/Controller.java:110`, `556-569`).
- `-30` (`CMD_SUB_COMMAND`): misc subcommands (`src/server/Controller.java:112`, `610-623`).

### Session-stage actions and payloads
- C->S `-29` + action `0` (`ACTION_LOGIN`): `UTF username`, `UTF password` (`src/server/Controller.java:548`).
- C->S `-29` + action `2` (`ACTION_CLIENT_INFO`):
  - `byte clientType`, `byte zoomLevel`, `boolean isGprs`, `int width`, `int height`, `boolean isQwerty`, `boolean isTouch`, `UTF platform`
  - parsed in `Service.setClientType(...)` (`src/services/Service.java:1514-1530`).
- S->C `-29` + action `2`: server list/link info (`src/network/SessionService.java:455-463`).

- C->S `-28` + action `2`: create character request (`src/server/Controller.java:561`, `629-669`).
- C->S `-28` + action `6`: request map template update (`src/server/Controller.java:562`).
- S->C `-28` + action `6`: map/npc/mob templates payload (`src/network/SessionService.java:94-125`).
- C->S `-28` + action `7`: request skill template update (`src/server/Controller.java:563`).
- S->C `-28` + action `7`: class/skill tree payload (`src/network/SessionService.java:127-201`).
- C->S `-28` + action `8`: request item update (`src/server/Controller.java:564`).
- C->S `-28` + action `10`: request map tile data for map id (`src/server/Controller.java:565`).
- S->C `-28` + action `10`: tile map bytes (`src/network/SessionService.java:342-357`).
- C->S `-28` + action `13`: init player info on map (`src/server/Controller.java:566-568`, `575-608`).

- C->S `-30` + sub `16`: increase point (`byte type`, `short point`) (`src/server/Controller.java:615-620`).
- C->S `-30` + sub `64`: submenu action (`int`, `short`) (`src/server/Controller.java:622`).

## Core in-game opcodes (verified subset)

| Direction | Cmd | Meaning | Payload shape | Reference |
|---|---:|---|---|---|
| C->S | `-7` | Move request | `byte b`, `short toX`, `short toY` | `src/server/Controller.java:440-463` |
| S->C | `-7` | Move broadcast | `int playerId`, `short x`, `short y` | `src/services/map/MapService.java:420-427` |
| C->S | `-45` | Use skill | first `byte status`; if `status==20` then `byte skillId, short dx, short dy, byte dir, short x, short y` | `src/server/Controller.java:254-257`, `src/services/SkillService.java:40-69` |
| C->S | `54` | Attack mob | `byte mobId`; if `mobId==-1`, then `int masterId` | `src/server/Controller.java:502-508` |
| C->S | `-60` | Attack player | `int targetPlayerId` | `src/server/Controller.java:272` |
| C->S | `-20` | Pick item | `short itemMapId` | `src/server/Controller.java:273-275` |
| S->C | `-19` | Notify others item picked | `short itemMapId`, `int pickerId` | `src/services/Service.java:992-999` |
| S->C | `-21` | Item disappeared | `short itemMapId` | `src/services/map/ItemMapService.java:35-41` |
| S->C | `68` | Drop item on map | `short itemMapId, short itemTemplateId, short x, short y, int owner/flag` | `src/services/Service.java:1182-1192` |
| S->C | `-24` | Map snapshot | map meta + waypoints + mobs + npcs + items + bg/effects | `src/map/Zone.java:510-620` |
| C->S | `-39` | Finish map load | no payload used | `src/server/Controller.java:266`, `src/services/map/ChangeMapService.java:496-549` |
| C->S | `-33`/`-23` | Waypoint map change | no extra payload read | `src/server/Controller.java:250-253`, `src/services/map/ChangeMapService.java:429-474` |
| C->S | `-40` | Use item transport/equip move | `byte type`, `byte index` | `src/server/Controller.java:259-261`, `src/services/func/UseItem.java:73-109` |
| C->S | `-43` | Item action/confirm | `byte type`, `byte where`, `byte index` (+ optional) | `src/server/Controller.java:262-264`, `src/services/func/UseItem.java:134-227` |
| C->S | `-86` | Trade action | `byte action` + action-specific fields | `src/server/Controller.java:220`, `src/services/func/TransactionService.java:46-164` |
| C->S | `6`/`7` | Buy/Sell item | buy: `byte tab`, `short itemId`; sell path parses `byte action, byte type, short id` | `src/server/Controller.java:225-232`, `411-419` |

## Resource/update protocol
- C->S `-74` type `1/2`: request resource sizes / resource binaries (`src/server/Controller.java:181`, `310-317`).
- S->C `-74`:
  - type `0`: version (`sendVersionRes`) (`src/network/SessionService.java:385-395`)
  - type `1`: size list (`sendSizeRes`) (`src/network/SessionService.java:397-412`)
  - type `2`: each file content (`sendRes`) (`src/network/SessionService.java:414-443`)
  - type `3`: transfer end + version (`src/network/SessionService.java:444-449`)
- S->C `-87`: update packed data blobs (`src/network/SessionService.java:63-92`).
- S->C `-67`, `66`, `-66`, `-32`, `11`: icon/image/effect/bg/mob template transfers (`src/network/SessionService.java:211-237`, `253-327`, `370-383`).

## Session flow (protocol-level)
1. TCP connect accepted; collector starts (`src/server/DragonBoy.java:95-107`).
2. Client sends `-29` action `2` (client info) to get server link payload (`src/server/Controller.java:548-550`, `src/network/SessionService.java:455-463`).
3. Client sends `-29` action `0` (login) (`src/server/Controller.java:548`).
4. Server authenticates and either:
   - sends create-char switch command `2` (`src/services/Service.java:1444-1451`, `src/player/PlayerDAO.java:1051-1055`), or
   - loads player and sends init/version/data (`src/network/MySession.java:115-138`).
5. Client requests required update channels (`-28` actions `6/7/8/10/13`) and eventually signals `-39` finish-load (`src/server/Controller.java:561-567`, `266`).

## Inferred conclusions
- `Inferred:` opcode space is intentionally mixed signed-byte values and legacy Teamobi-compatible message IDs.
- `Inferred:` protocol has two control layers: transport encryption/framing and gameplay command channels (`-29/-28/-30` + direct cmds).
- `Inferred:` client compatibility risk is highest in key handshake timing and special 3-byte length commands.
