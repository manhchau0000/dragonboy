# Minimum Client Spec (libGDX)

Goal: define the minimum client behavior needed to connect to this server and enter the game world safely, without changing server behavior.

References below point to exact server source methods.

## 1) Required packet order

Minimum happy-path order (existing account with character):
1. TCP connect; wait for collector readiness (`src/server/DragonBoy.java:95-107`, `src/network/Collector.java:32-42`).
2. C->S `-27` (handshake trigger) (`src/network/Collector.java:36-38`).
3. S->C `-27` (key payload), then enable XOR for all later packets (`src/network/KeyHandler.java:9-20`, `src/network/MessageSendCollect.java:20-39`).
4. C->S `-29` action `2` (client info) (`src/server/Controller.java:544-550`, `src/services/Service.java:1514-1531`).
5. S->C `-29` action `2` (link/ip payload) (`src/network/SessionService.java:455-463`).
6. C->S `-29` action `0` (login with user/pass) (`src/server/Controller.java:548`).
7. S->C initial data/version packets and account/player bootstrap (`src/network/MySession.java:117-138`, `src/server/Controller.java:675-703`).
8. C->S `-28` actions `6`, `7`, `8`, `10` as needed for templates/resources (`src/server/Controller.java:561-566`, `src/network/SessionService.java:94-201`, `342-357`).
9. C->S `-28` action `13` (init player info) (`src/server/Controller.java:566-568`, `575-608`).
10. S->C `-24` map info snapshot (`src/map/Zone.java:510-541`).
11. C->S `-39` finish-load (`src/server/Controller.java:266`).
12. S->C post-load entity/effect sync (`src/services/map/ChangeMapService.java:496-549`).

Conditional branch (new account, no character):
- After login, server sends create-char switch (`cmd 2`), then client must send `-28` action `2` create-char payload (`src/player/PlayerDAO.java:1051-1055`, `src/services/Service.java:1444-1451`, `src/server/Controller.java:629-669`).

## 2) Handshake/XOR activation timing

Required timing:
1. Send handshake trigger `-27` in plaintext.
2. Receive server `-27` key packet in plaintext.
3. Reconstruct key bytes from length + first byte + xor chain (`src/network/KeyHandler.java:13-17`).
4. Enable XOR for next outbound packet and all future packets.

Why timing matters:
- Server flips `session.sentKey=true` immediately after sending key packet (`src/network/KeyHandler.java:18-20`), and from then on expects encoded inbound bytes (`src/network/MessageSendCollect.java:20-39`).

Transport rules to mirror exactly:
- Rolling XOR indices for read/write (`curR`, `curW`) (`src/network/MessageSendCollect.java:13-15`, `44-50`, `95-101`).
- Special 3-byte length mode for commands `-32,-66,-74,11,-67,-87,66` (`src/network/MessageSendCollect.java:64-70`).

## 3) Login flow

C->S login packets:
- `-29` + action `2`: `byte typeClient, byte zoomLevel, boolean isGprs, int width, int height, boolean isQwerty, boolean isTouch, UTF platform` (`src/services/Service.java:1516-1525`).
- `-29` + action `0`: `UTF username, UTF password` (`src/server/Controller.java:548`).

Server decisions:
- Auth and anti-login checks in `MySession.login` + `PlayerDAO.login` (`src/network/MySession.java:85-116`, `src/player/PlayerDAO.java:1006-1087`).
- Can reject for wait windows/maintenance/capacity; client must handle notify/fail responses (`src/network/MySession.java:95-107`, `src/player/PlayerDAO.java:1034-1049`, `1071-1073`, `src/services/Service.java:2116-2123`).

## 4) Resource/init flow

Minimum support:
- Version/data/bootstrap packets:
  - `-77` small version (`src/network/SessionService.java:303-315`)
  - `-93` bg-item version (`src/network/SessionService.java:239-251`)
  - `-28` action `4` version-game payload (`src/network/SessionService.java:40-61`)
  - `-31` bg-item data (`src/network/SessionService.java:271-287`)
- Optional/conditional update pulls from client:
  - map templates (`-28` action `6`) (`src/network/SessionService.java:94-125`)
  - skill templates (`-28` action `7`) (`src/network/SessionService.java:127-201`)
  - map temp (`-28` action `10`) (`src/network/SessionService.java:342-357`)
  - resource stream (`-74` type `1/2` requests, server returns `0/1/2/3`) (`src/server/Controller.java:310-317`, `src/network/SessionService.java:385-452`).

## 5) Map load flow

Required behavior:
1. Send `-28` action `13` once character/session attach is complete.
2. Parse `-24` map snapshot and instantiate map-local state from packet fields:
   - map meta, waypoints, mobs, npcs, items, background/effects (`src/map/Zone.java:510-620`).
3. Keep local player position from snapshot (`src/map/Zone.java:520-521`).

Safety rule:
- Do not send movement/skill packets before map snapshot is applied.

## 6) Finish-load flow

Required behavior:
1. After local map snapshot application, send C->S `-39` (no payload).
2. Apply post-load packets from `ChangeMapService.finishLoadMap(...)`:
   - me/others sync
   - effect sync
   - map-specific post-join events
   (`src/services/map/ChangeMapService.java:496-549`).

`Inferred:` treating `-39` as a strict barrier prevents many desync issues at join time.

## 7) Minimum local client state

Keep at minimum:
- Transport state:
  - socket status
  - key bytes
  - XOR read/write rolling indices
  - encryption-enabled flag
  (`src/network/MessageSendCollect.java:13-15`, `44-50`, `95-101`).
- Session state:
  - `typeClient`, `zoomLevel`, parsed `version` compatibility
  (`src/services/Service.java:1516-1526`, `src/network/MySession.java:31-33`, `51`).
- Player core state:
  - player id
  - map id/zone
  - x/y
  - hp/mp/money displays
  (`src/map/Zone.java:519-521`, `src/services/map/MapService.java:423-427`, `src/services/PlayerService.java:116-140`).
- Runtime registries:
  - current map waypoints/mobs/npcs/items from `-24`
  - known templates/resources requested during init.

## 8) Movement sync requirements

Client send requirements:
- C->S `-7`: send `byte b, short toX, short toY` (`src/server/Controller.java:450-453`).
- Avoid extreme teleport deltas, especially in BlackBall maps (>500 ignored) (`src/server/Controller.java:454-457`).

Client receive requirements:
- Apply S->C `-7` as authoritative for remote players (`src/services/map/MapService.java:420-427`).
- Accept server corrections/resets (for example `resetPoint` cmd `46`) (`src/services/Service.java:327-336`).
- Respect map-change side effects triggered by movement (`src/services/PlayerService.java:156-175`).

## 9) Known protocol risks

1. XOR desync risk if any packet length decode is wrong (`src/network/MessageSendCollect.java:24-39`, `53-84`).
2. Special 3-byte length commands must be parsed/written exactly (`src/network/MessageSendCollect.java:64-70`).
3. Signed-byte command handling bugs (negative opcode values are common) (`src/server/Controller.java:47-120`).
4. Version-conditional payload changes (example: consign quantity type changes by client version) (`src/server/Controller.java:328`).
5. Premature gameplay packets before `-39` can cause inconsistent map/effect state (`src/services/map/ChangeMapService.java:496-549`).
6. Reconnect race: old session can be kicked on duplicate login (`src/player/PlayerDAO.java:1041-1045`, `1056-1059`).

## 10) Fields still unknown and how to verify

Unknown/partially-known fields:
1. Move packet first byte `b` meaning beyond flight-task check (`src/server/Controller.java:450-460`).
2. Several `mapInfo(-24)` booleans/flags in mob serialization (`src/map/Zone.java:570-585`).
3. Item drop packet `68` trailing int semantic (`src/services/Service.java:1190`, `1205`).
4. `sendVersionGame` long threshold array purpose (`src/network/SessionService.java:50-56`).
5. `SessionService.sendDataImageVersion(...)` is empty in current server (`src/network/SessionService.java:203-209`), so expected client behavior for `-111` may vary by legacy clients.

Verification procedure (safe, no server changes):
1. Add binary packet logging in client before and after XOR transform (cmd, length, first N payload bytes).
2. For each unknown field, generate one controlled action at a time and diff packet captures.
3. Correlate captured packets with exact parser/writer methods above.
4. Replay known-good packet sequences against a local account to confirm deterministic server response.
5. Mark each field as `Confirmed` only after at least one write-path and one read-path match.
