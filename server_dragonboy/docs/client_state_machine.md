# Client State Machine

## Scope
Concrete state model for minimal compatible libGDX client.

Ground truth:
- `docs/minimum_client_spec.md`
- `docs/min_login_flow.md`
- `docs/min_map_load_flow.md`
- `docs/min_movement_flow.md`

## States
- `DISCONNECTED`
- `CONNECTED`
- `KEY_WAIT`
- `XOR_ACTIVE`
- `CLIENT_INFO_SENT`
- `LOGIN_SENT`
- `AUTH_SUCCESS_PENDING_INIT`
- `TEMPLATE_SYNC`
- `INIT_INFO_SENT`
- `MAP_SNAPSHOT_READY`
- `FINISH_LOAD_SENT`
- `IN_WORLD_ACTIVE`
- `MAP_LOADING`
- `AUTH_REJECTED`
- `CREATE_CHAR_REQUIRED`

## State Definitions

### `DISCONNECTED`
- Entry conditions:
  - app start before connect
  - explicit disconnect
  - transport/protocol fatal error
- Allowed outgoing packets:
  - none
- Expected incoming packets:
  - none
- Error transitions:
  - stay `DISCONNECTED`

### `CONNECTED`
- Entry conditions:
  - TCP connected event
- Allowed outgoing packets:
  - `-27` handshake trigger
- Expected incoming packets:
  - none required yet
- Error transitions:
  - transport error -> `DISCONNECTED`

### `KEY_WAIT`
- Entry conditions:
  - handshake trigger `-27` sent
- Allowed outgoing packets:
  - none (recommended)
- Expected incoming packets:
  - `-27` key payload
- Error transitions:
  - timeout/decode error -> `DISCONNECTED`

### `XOR_ACTIVE`
- Entry conditions:
  - valid `S->C -27` parsed, XOR enabled
- Allowed outgoing packets:
  - `-29(action=2)` client-info
- Expected incoming packets:
  - optional version/resource notifications
- Error transitions:
  - XOR decode failure -> `DISCONNECTED`

### `CLIENT_INFO_SENT`
- Entry conditions:
  - `C->S -29(action=2)` sent
- Allowed outgoing packets:
  - `-29(action=0)` login
- Expected incoming packets:
  - `S->C -29(action=2)` link payload
- Error transitions:
  - malformed response/timeout -> `DISCONNECTED`

### `LOGIN_SENT`
- Entry conditions:
  - `C->S -29(action=0)` sent
- Allowed outgoing packets:
  - none until auth result
- Expected incoming packets:
  - success init burst (`-77`, `-93`, `-28(action=4)`, `-31`, others)
  - or failure packets (`-26`, `122`, `-102`)
  - or create-char switch (`cmd 2`)
- Error transitions:
  - failure packet -> `AUTH_REJECTED`
  - timeout/disconnect -> `DISCONNECTED`

### `AUTH_SUCCESS_PENDING_INIT`
- Entry conditions:
  - login accepted and init burst begins
- Allowed outgoing packets:
  - `-28(action=6)`
  - `-28(action=7)`
  - `-28(action=8)`
  - `-28(action=10,mapId)`
- Expected incoming packets:
  - template blocks (`-28` variants)
- Error transitions:
  - decode/protocol mismatch -> `DISCONNECTED`

### `TEMPLATE_SYNC`
- Entry conditions:
  - at least one template request sent
- Allowed outgoing packets:
  - remaining `-28(action=6/7/8/10)`
  - `-28(action=13)` when minimum data ready
- Expected incoming packets:
  - `-28(action=6/7/8/10)` responses
- Error transitions:
  - sync timeout -> `DISCONNECTED` or retry in place

### `INIT_INFO_SENT`
- Entry conditions:
  - `C->S -28(action=13)` sent
- Allowed outgoing packets:
  - none gameplay-critical
- Expected incoming packets:
  - `-24` map snapshot
- Error transitions:
  - missing snapshot timeout -> `DISCONNECTED`

### `MAP_SNAPSHOT_READY`
- Entry conditions:
  - valid `-24` parsed and registry populated
- Allowed outgoing packets:
  - `-39` finish-load
- Expected incoming packets:
  - optional additional init packets
- Error transitions:
  - parser inconsistency -> `DISCONNECTED`

### `FINISH_LOAD_SENT`
- Entry conditions:
  - `C->S -39` sent
- Allowed outgoing packets:
  - none movement yet (recommended until post-load settles)
- Expected incoming packets:
  - post-load entity/effect sync
- Error transitions:
  - timeout/disconnect -> `DISCONNECTED`

### `IN_WORLD_ACTIVE`
- Entry conditions:
  - finish-load responses processed
- Allowed outgoing packets:
  - movement `-7`
  - minimal interaction packets only if implemented
- Expected incoming packets:
  - movement `-7` (others)
  - corrections `46`
  - map-clear/map-change packets
- Error transitions:
  - map transition trigger -> `MAP_LOADING`
  - protocol/transport failure -> `DISCONNECTED`

### `MAP_LOADING`
- Entry conditions:
  - map transition detected (e.g. clear-map/map snapshot sequence)
- Allowed outgoing packets:
  - `-28(action=10,mapId)` if tile data needed
  - `-39` after snapshot apply
- Expected incoming packets:
  - `-24` new map snapshot
  - post-load sync
- Error transitions:
  - timeout/decode error -> `DISCONNECTED`

### `AUTH_REJECTED`
- Entry conditions:
  - login failure packet/dialog observed
- Allowed outgoing packets:
  - optional retry path: resend client-info/login after user action
- Expected incoming packets:
  - notification packets only
- Error transitions:
  - reconnect requested -> `CONNECTED` (new session)

### `CREATE_CHAR_REQUIRED`
- Entry conditions:
  - server command `2` received
- Allowed outgoing packets:
  - `-28(action=2,name,gender,hair)`
- Expected incoming packets:
  - post-create login success burst, or reject notification
- Error transitions:
  - create-char validation reject -> remain `CREATE_CHAR_REQUIRED`
  - disconnect -> `DISCONNECTED`

## Global Error Rules

1. Any frame decode exception -> `DISCONNECTED`.
2. Any illegal outgoing packet for current state -> blocked locally and logged.
3. Any XOR mismatch suspicion (impossible length/command stream) -> `DISCONNECTED`.
