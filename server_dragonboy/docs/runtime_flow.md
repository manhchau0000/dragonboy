# Runtime Flow (Phase 4)

## Scope
- This describes live sequence behavior observed in code.
- `Inferred:` marks sequence assumptions across multiple methods.

## A) Login sequence
1. Socket accepted in `Network.run()` and `MySession` created (`src/network/Network.java:107-120`, `src/server/DragonBoy.java:119`).
2. Session is configured with controller + codec + key handler and collector thread starts (`src/server/DragonBoy.java:103-107`).
3. `Collector.run()` reads packets and forwards to `Controller.onMessage(...)` (`src/network/Collector.java:35-40`, `src/server/Controller.java:149`).
4. Client sends `-29` action `2` (client info) then server responds with link/IP payload (`src/server/Controller.java:548-550`, `src/services/Service.java:1514-1531`, `src/network/SessionService.java:455-463`).
5. Client sends `-29` action `0` (username/password) (`src/server/Controller.java:548`).
6. `MySession.login(...)` performs gate checks (maintenance, capacity, anti-login) then `PlayerDAO.login(...)` (`src/network/MySession.java:85-116`).
7. If valid and character exists, server binds player session and sends initial version/data/info (`src/network/MySession.java:117-138`).

## B) Character creation / selection sequence
1. During `PlayerDAO.login(...)`, if no player row exists for account, server sends:
   - version/data packets
   - `switchToCreateChar` packet (`cmd 2`)
   (`src/player/PlayerDAO.java:1051-1055`, `src/services/Service.java:1444-1451`).
2. Client sends `-28` action `2` with `name`, `gender`, `hair` (`src/server/Controller.java:561`, `629-637`).
3. Server validates name policy and uniqueness, then persists player (`src/server/Controller.java:638-661`).
4. On success, server immediately re-calls `session.login(...)` with cached credentials (`src/server/Controller.java:666-668`).

`Inferred:` there is no multi-slot character picker path in this server; login loads one player record per account (`src/player/PlayerDAO.java:1050-1061`).

## C) Map join sequence
1. After login attach, `Controller.sendInfo(...)` sends global/user init packets, then starts player thread (`src/server/Controller.java:675-703`).
2. Client requests/receives map/skill/item/template updates via `-28` actions (`src/server/Controller.java:561-566`, `src/network/SessionService.java:94-201`, `342-357`).
3. Client sends `-28` action `13` to initialize in-map player state (`src/server/Controller.java:566-568`, `575-608`).
4. `initPlayerInfo(...)` sends full player snapshot, flags, tasks, and `zone.mapInfo(...)` (`src/server/Controller.java:575-592`, `src/map/Zone.java:510-541`).
5. Client sends `-39` finish load; server calls `ChangeMapService.finishLoadMap(...)` to sync other entities/effects (`src/server/Controller.java:266`, `src/services/map/ChangeMapService.java:496-549`).

## D) Movement sequence
1. Client sends `CMD_MOVE (-7)` with `b, toX, toY` (`src/server/Controller.java:440-452`).
2. Controller rejects illegal states (dead/effect-lock/too-far in BlackBall) (`src/server/Controller.java:441-457`).
3. `PlayerService.playerMove(...)` updates authoritative position and movement side-effects (`src/services/PlayerService.java:142-198`).
4. Server broadcasts movement to others with `MapService.sendPlayerMove(...)` (`src/services/map/MapService.java:420-427`).

## E) Combat sequence
1. Client can send:
   - `-45` use skill
   - `54` attack mob
   - `-60` attack player
   (`src/server/Controller.java:254-257`, `271-272`, `502-508`).
2. All flows converge into `SkillService.useSkill(...)` either directly or via `Service.attackMob/attackPlayer` (`src/services/Service.java:787-806`, `1103-1111`, `src/services/SkillService.java:40-109`).
3. Server checks legality (target, state, mana, cooldown, restrictions), applies damage via `Player.injured(...)` / `Mob.injured(...)`, and emits result packets (`src/services/SkillService.java`, `src/player/Player.java:905+`, `src/mob/Mob.java:96+`).

## F) Disconnect sequence
1. Collector loop exits on socket/processing failure (`src/network/Collector.java:43-52`).
2. Accept handler calls `sessionDisconnect(...)` (`src/network/Collector.java:46-47`, `src/server/DragonBoy.java:110-118`).
3. If player exists, `Client.kickSession(...)` runs (`src/server/DragonBoy.java:111-115`, `src/server/Client.java:124-129`).
4. Cleanup path removes player from registries, exits map, cancels trade, updates account logout timestamp, and saves player state (`src/server/Client.java:64-122`).

## G) Reconnect sequence
1. New login repeats the normal login path.
2. Duplicate online session handling: if account already in game, old session is kicked (`src/player/PlayerDAO.java:1041-1045`, `1056-1059`).
3. Anti-fast-relogin waiting rules can delay re-entry (`src/player/PlayerDAO.java:1034-1049`).
4. Loaded player location may be normalized to safe maps if previous map is restricted/offline event map (`src/player/PlayerDAO.java:1171-1200`).

## Inferred conclusions
- `Inferred:` runtime is packet-driven plus periodic world loops; correctness depends on both message order and loop timing.
- `Inferred:` `finishLoadMap (-39)` is a critical compatibility checkpoint because many visual/effect sync packets are sent there.
- `Inferred:` reconnect behavior is intentionally single-session-per-account with forced handoff.
