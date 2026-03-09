# Authoritative Server Rules (Phase 3)

## Scope
- This phase documents where server-side truth is enforced.
- No behavioral changes were made.
- `Inferred:` marks cross-file conclusions.

## 1) Movement authority

### Server accepts requests but owns final position
- Movement request enters at `Controller.handleMove(...)` (reads requested `toX/toY`) (`src/server/Controller.java:440-463`).
- Authoritative state mutation happens in `PlayerService.playerMove(...)` (`src/services/PlayerService.java:142-198`):
  - sets `player.location.x/y`
  - cancels charging/hold effects before move
  - applies map-specific constraints
  - triggers broadcast via `MapService.sendPlayerMove(...)`.

### Movement constraints enforced server-side
- Dead characters cannot move (`src/server/Controller.java:441-444`).
- Effect-lock blocks move (`src/server/Controller.java:445`).
- In Black Ball maps, long-distance move packets are ignored if distance > 500 (`src/server/Controller.java:454-457`).
- Additional map boundary protection in maps `85..91` can force return to home map (`src/services/PlayerService.java:156-175`).
- Fly-state MP drain is computed server-side (`src/services/PlayerService.java:186-194`).

## 2) Combat authority

### Skill gating and legality checks
- All skill use goes through `SkillService.useSkill(...)` (`src/services/SkillService.java:40-109`).
- Server checks include:
  - same-clan friendly restrictions in specific contexts (`src/services/SkillService.java:41-48`)
  - revive grace windows (`src/services/SkillService.java:49-51`)
  - effect lockouts (`src/services/SkillService.java:70-73`, `80-87`)
  - mana and cooldown checks (`src/services/SkillService.java:76`, `86`, `999-1033`).

### Damage and death are server-owned
- Player damage resolution: `Player.injured(...)` (`src/player/Player.java:905-1076`).
  - applies evasion/defense caps, shield behavior, map-specific adjustments, final HP subtraction.
- Player death handling: `Player.setDie(...)` (`src/player/Player.java:324-379`).
  - resets states, drops money item, sends death packets.
- Mob damage/death: `Mob.injured(...)` and `setDie()` (`src/mob/Mob.java:96-166`, `85-88`).

### Attack dispatch path
- `CMD_ATTACK_MOB (54)` parsed in controller then delegated to `Service.attackMob(...)` (`src/server/Controller.java:502-508`, `src/services/Service.java:787-806`).
- `CMD_ATTACK_PLAYER (-60)` delegated to `Service.attackPlayer(...)` then `SkillService.useSkill(...)` (`src/server/Controller.java:272`, `src/services/Service.java:1103-1111`).

## 3) Inventory authority

### Inventory mutations occur server-side only
- Equip/unequip is handled by `InventoryService` (`itemBagToBody`, `itemBodyToBag`) (`src/services/player/InventoryService.java:376-413`).
- Item movement packet (`CMD_USE_ITEM -40`) is parsed in `UseItem.getItem(...)` and translated to inventory service calls (`src/services/func/UseItem.java:73-109`).
- Item action packet (`CMD_DO_ITEM -43`) uses server-confirmed flows (use/throw/accept) (`src/services/func/UseItem.java:134-227`).

### Currency/capacity checks
- `addItemBag(...)` enforces currency limits, special-item rules, bag/box expansion limits (`src/services/player/InventoryService.java:709-771`).
- Throwing protected items is denied server-side (`src/services/player/InventoryService.java:127-137`, `src/services/func/UseItem.java:203-205`).

## 4) Drop and pickup authority

### Drop generation
- Mob death calls `mobReward(...)` and serializes drops in death packet (`src/mob/Mob.java:452-503`).
- Drop list built by `getItemMobReward(...)` with map/event conditions (`src/mob/Mob.java:505+`).
- Server map item spawn broadcast uses command `68` (`src/services/Service.java:1182-1192`).

### Pickup validation
- `ItemMapService.pickItem(...)` rate-limits pickup attempts by timestamp (`src/services/map/ItemMapService.java:21-27`).
- Actual ownership/capacity/special checks happen in `Zone.pickItem(...)` (`src/map/Zone.java:249-316`):
  - owner check
  - cannot-pick item type check
  - bag capacity and special requirement checks
  - task progression hooks.
- Successful pickup notifies picker and other players via `-20` / `-19` flows (`src/map/Zone.java:287-295`, `src/services/Service.java:992-999`).

## 5) Cooldown and resource authority
- Skill cooldown eligibility: `canUseSkillWithCooldown(...)` (`src/services/SkillService.java:1030-1033`).
- Skill mana eligibility: `canUseSkillWithMana(...)` (`src/services/SkillService.java:999-1028`).
- After-skill side effects update MP and last-use timestamps server-side (`src/services/SkillService.java:1035-1061`, `1063-1083`).
- Cooldown sync packets sent by server (`sendTimeSkill`) (`src/services/Service.java:1132-1146`).

## 6) Map sync authority

### Map transitions
- Change-map logic and access checks are centralized in `ChangeMapService.changeMap(...)` (`src/services/map/ChangeMapService.java:316-427`).
- Access gating by progression/task/map is in `checkMapCanJoin(...)` (`src/services/map/ChangeMapService.java:840+`).

### Map join synchronization
- On change-map, server sends clear map + full map snapshot (`zone.mapInfo`) (`src/services/map/ChangeMapService.java:377-382`, `src/map/Zone.java:510-541`).
- Client `CMD_FINISH_LOAD_MAP (-39)` triggers final sync of entities/effects in `finishLoadMap(...)` (`src/server/Controller.java:266`, `src/services/map/ChangeMapService.java:496-549`).

## 7) Disconnect/reconnect authority hooks
- Session removal and player cleanup happen in `Client.remove(...)` (`src/server/Client.java:64-122`).
- Map exit, trade cancel, clan online remove, and player DB save are server-driven (`src/server/Client.java:95-122`).

## Inferred conclusions
- `Inferred:` client packets are intents, not state commits; server computes final legal outcomes for all core systems.
- `Inferred:` anti-cheat posture is soft but present (distance checks, ownership checks, rate limits, cooldown/mana checks).
- `Inferred:` preserving compatibility requires preserving not only packet IDs but also these guard conditions and timing thresholds.
