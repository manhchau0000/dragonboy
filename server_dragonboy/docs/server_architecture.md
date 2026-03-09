# Server Architecture (Phase 1)

## Scope
- Goal: document the current architecture without changing behavior.
- Verification sources are code references in this repository.
- `Inferred:` marks conclusions that are not directly stated in one line of code.

## Entry points
- Process entry: `run.bat` runs `java -jar dist/NgocRongOnline.jar`.
- Java entry: `DragonBoy.main(...)` in `src/server/DragonBoy.java:47`.
- Boot init: `DragonBoy.init()` in `src/server/DragonBoy.java:60`.
- Runtime start: `DragonBoy.run()` in `src/server/DragonBoy.java:70`.

## Boot flow
1. `DragonBoy.main` calls `DragonBoy.gI().init()` then `run()` (`src/server/DragonBoy.java:47-53`).
2. `init()` configures logging, loads static data, initializes NPC factory, and clears history (`src/server/DragonBoy.java:62-67`).
3. `GameData.load()` executes DB load then map initialization (`src/server/GameData.java:122-130`).
4. `GameData.initMap()` constructs map instances, mobs, NPCs, then starts the `"Update Maps"` loop (`src/server/GameData.java:150-191`).
5. `run()` opens network socket and starts manager threads (boss/event/tournament/namec dragon) (`src/server/DragonBoy.java:72-90`).
6. `activeServerSocket()` binds accept handler and wires each session to:
   - message handler: `Controller.gI()`
   - transport codec: `MessageSendCollect`
   - key handler: `MyKeyHandler`
   - collector thread start
   (`src/server/DragonBoy.java:95-107`).

## Session boot and attach flow
1. `Network.run()` accepts sockets and clones `MySession` (`src/network/Network.java:107-120`, `src/server/DragonBoy.java:119`).
2. `Collector.run()` reads messages forever while connected (`src/network/Collector.java:32-42`).
3. Pre-login dispatch goes to `Controller.onMessage(...)` (`src/network/Collector.java:39`, `src/server/Controller.java:149`).
4. After successful login, `MySession.login(...)` attaches `Player`, inserts into `Client`, sends versions/info, then `Controller.sendInfo(...)` (`src/network/MySession.java:115-138`).
5. `Controller.sendInfo(...)` starts per-player update thread via `player.start()` (`src/server/Controller.java:675-703`, `src/player/Player.java:413-415`).

## Major modules and responsibilities

| Module | Responsibility | Primary references |
|---|---|---|
| `server.DragonBoy` | Process lifecycle, boot orchestration, socket activation, manager startup | `src/server/DragonBoy.java` |
| `server.GameData` | Static game data load, map graph creation, zone update scheduler | `src/server/GameData.java` |
| `network.Network` | NIO accept loop and session creation | `src/network/Network.java` |
| `network.Session` + `Collector/Sender` | Per-connection send/collect threads and transport handling | `src/network/Session.java`, `src/network/Collector.java` |
| `server.Controller` | Central packet dispatcher (pre-login and in-game) | `src/server/Controller.java` |
| `services.*` | Gameplay/business services (combat, map, inventory, chat, clans, events) | `src/services/*.java`, `src/services/map/*.java`, `src/services/func/*.java` |
| `map.Zone` + `map.Map` | Authoritative zone state update, map snapshot, item pickup handling | `src/map/Zone.java`, `src/map/Map.java` |
| `player.Player` | Per-entity state + update loop + damage/death logic | `src/player/Player.java` |
| `server.Client` | Online player/session registries, disconnect cleanup, periodic session timeout checks | `src/server/Client.java` |
| `player.PlayerDAO` | Account/player persistence and restore | `src/player/PlayerDAO.java` |

## Continuous loops and background threads
- Network accept loop: `Network.run()` (`src/network/Network.java:107-127`).
- World/zone loop (~1s): `"Update Maps"` calling `zone.update()` (`src/server/GameData.java:165-182`).
- Session timeout loop (~1s): `Client.run()` / `update()` (`src/server/Client.java:154-192`).
- Player loop (~1s per player): `Player.run()` -> `Player.update()` (`src/player/Player.java:404-410`, `417+`).
- Trade loop (~300ms): `TransactionService.run()` (`src/services/func/TransactionService.java:206-215`).
- Global chat loop (~1s): `ChatGlobalService.run()` (`src/services/ChatGlobalService.java:87-112`).
- Server notify loop (~1s): `ServerNotify.run()` (`src/server/ServerNotify.java:36-55`).
- Event/manager threads from boot: `DragonBoy.run()` (`src/server/DragonBoy.java:73-90`).

## Architectural boundaries
- Packet parsing and routing live in `Controller`; domain validation is delegated to services (`src/server/Controller.java:176-287`).
- World state mutation happens in zone/player/mob/inventory services, not in network classes.
- Disconnect persistence and cleanup are centralized in `Client.remove(...)` and `PlayerDAO.updatePlayer(...)` (`src/server/Client.java:64-122`).

## Inferred conclusions
- `Inferred:` this is a server-authoritative architecture: clients request actions, but movement/combat/item outcomes are computed server-side.
- `Inferred:` the code uses many singleton services intentionally to keep mutable game state globally accessible rather than dependency-injected.
- `Inferred:` threading model is coarse-grained (many long-running loops), so behavior depends on those loop cadences (mostly 300ms-1000ms windows).
