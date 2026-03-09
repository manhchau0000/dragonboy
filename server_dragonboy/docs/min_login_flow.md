# Minimal Login Flow

## Scope
- This is the minimum compatible login flow for an existing server account.
- It includes transport handshake because login packets depend on XOR state.
- References are exact server code paths.

## Exact Packet Sequence (Client -> Server / Server -> Client)

1. `C->S` `-27` (handshake trigger)
- Payload: none.
- Reference: `src/network/Collector.java:36-38`.

2. `S->C` `-27` (session key payload)
- Payload fields:
  - `byte keyLen`
  - `byte key[0]`
  - `byte encodedKey[i]` for `i >= 1` where `encodedKey[i] = key[i] ^ key[i-1]`
- References: `src/network/KeyHandler.java:10-17`.

3. Client enables XOR transport for all later packets
- Timing requirement: immediately after reading server key packet.
- References: `src/network/KeyHandler.java:18-20`, `src/network/MessageSendCollect.java:20-39`.

4. `C->S` `-29` with action `2` (`ACTION_CLIENT_INFO`)
- Payload fields (exact read order):
  - `byte clientType`
  - `byte zoomLevel`
  - `boolean isGprs`
  - `int width`
  - `int height`
  - `boolean isQwerty`
  - `boolean isTouch`
  - `UTF platform`
- References: `src/server/Controller.java:544-550`, `src/services/Service.java:1514-1526`.

5. `S->C` `-29` action `2` (link/ip list)
- Payload fields:
  - `byte 2`
  - `UTF linkPayload`
  - `byte 1`
- Reference: `src/network/SessionService.java:455-463`.

6. `C->S` `-29` with action `0` (`ACTION_LOGIN`)
- Payload fields:
  - `UTF username`
  - `UTF password`
- Reference: `src/server/Controller.java:548`.

7. Success path responses (existing character)
- `S->C` `-77` (small version), then `-93` (bg-item version) during login attach.
- `S->C` `-28` action `4` (version game payload).
- `S->C` `-31` (bg item data).
- Followed by init burst from `Controller.sendInfo(...)`.
- References: `src/network/MySession.java:117-138`, `src/network/SessionService.java:303-315`, `239-251`, `40-61`, `271-287`, `src/server/Controller.java:675-703`.

8. New-character branch (if account has no player)
- `S->C` command `2` (switch to create-char).
- Then client must send `C->S` `-28` action `2` with:
  - `UTF name`
  - `byte gender`
  - `byte hair`
- References: `src/player/PlayerDAO.java:1051-1055`, `src/services/Service.java:1444-1451`, `src/server/Controller.java:629-669`.

## Expected Server Responses

Success (existing character):
- Version/data packets listed above.
- Client becomes bound to player session and can proceed to map init flow.
- References: `src/network/MySession.java:132-138`, `src/server/Controller.java:675-703`.

Failure / blocked cases:
- `-26` notify dialogs (ban, maintenance, full server, etc.).
- `122` wait-to-login with `short seconds`.
- `-102` login-fail byte.
- Possible disconnect on invalid parsing in `messageNotLogin`.
- References: `src/network/MySession.java:91-107`, `src/player/PlayerDAO.java:1032-1049`, `1071-1073`, `src/services/Service.java:1480-1486`, `2116-2123`, `src/server/Controller.java:551-553`.

## Local Client State Transitions

1. `SOCKET_CONNECTED` -> `KEY_WAIT` (after TCP connect).
2. `KEY_WAIT` -> `XOR_ACTIVE` (after parsing `S->C -27`).
3. `XOR_ACTIVE` -> `CLIENT_INFO_SENT` (after `C->S -29(action=2)`).
4. `CLIENT_INFO_SENT` -> `LOGIN_SENT` (after `C->S -29(action=0)`).
5. `LOGIN_SENT` -> one of:
- `AUTH_SUCCESS_PENDING_INIT` (success burst received), or
- `CREATE_CHAR_REQUIRED` (received cmd `2`), or
- `AUTH_REJECTED` (failure packet/dialog).

## Failure Points

1. XOR activation done too early/late causes framing desync (`src/network/MessageSendCollect.java:20-39`).
2. Signed-byte command parsing bugs for negative opcodes (`src/server/Controller.java:47-120`).
3. Wrong client-info field order breaks session version parsing (`src/services/Service.java:1516-1526`).
4. Re-login timing window can reject valid credentials temporarily (`src/player/PlayerDAO.java:1034-1049`).
5. Existing online session may be kicked on duplicate login (`src/player/PlayerDAO.java:1041-1045`, `1056-1059`).

## Assumptions vs Confirmed Behavior

Confirmed:
- Handshake trigger/response command is `-27` and flips transport to XOR mode.
- Login channel is `-29`, with actions `2` (client-info) and `0` (auth).
- New-character path is server command `2` + `-28(action=2)` create-char payload.

Assumptions:
- Client should send handshake trigger immediately after connect (server also has a controller case for `-27`, but collector-level handling is primary).
- Init packet burst order after successful login may interleave with other server emissions, but listed packets are required by the login attach path.
