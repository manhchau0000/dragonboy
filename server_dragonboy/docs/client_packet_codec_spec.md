# Client Packet Codec Spec

## Scope
Binary packet codec contract for minimal compatible libGDX client.

Ground truth:
- `docs/minimum_client_spec.md`
- `docs/min_login_flow.md`
- `docs/min_map_load_flow.md`
- `docs/min_movement_flow.md`

## 1) Signed Byte Handling

1. Command IDs are signed 8-bit values (`-128..127`).
2. Internal packet model should store both:
- `byte commandSigned`
- `int commandUnsigned = commandSigned & 0xFF`
3. Routing keys should use signed value, since docs/server use negative IDs directly.

## 2) Plaintext vs XOR Mode Switching

### Plaintext mode
- Active from connect until server key packet is parsed.
- Read/write command + length + payload without XOR.

### XOR mode
- Activate immediately after parsing `S->C -27` key packet.
- XOR applies to:
  - command byte
  - length bytes
  - payload bytes
- Keep separate rolling indices for read and write streams.

### Required state fields
- `boolean xorActive`
- `byte[] key`
- `int readKeyIndex`
- `int writeKeyIndex`

## 3) Length Rules

### Default rule
- 2-byte big-endian length.

### Special 3-byte rule (writer/reader must support)
Commands:
- `-32`, `-66`, `-74`, `11`, `-67`, `-87`, `66`

Behavior:
- Length is encoded across 3 bytes in server transport logic.
- Client decoder must branch on command before payload read.

## 4) Packet Object Model

### Core models
- `PacketFrame`
  - `byte cmd`
  - `int payloadLength`
  - `byte[] payload`
  - `boolean xorMode`
- `PacketReader`
  - `PacketFrame readNext(InputStream in, CodecContext ctx)`
- `PacketWriter`
  - `void write(PacketFrame frame, OutputStream out, CodecContext ctx)`
- `CodecContext`
  - XOR mode + key state + counters

### Typed payload wrappers
- `PacketInput`
  - `readByte`, `readUnsignedByte`, `readShort`, `readInt`, `readLong`, `readBoolean`, `readUTF`
- `PacketOutput`
  - matching write methods

## 5) UTF/String Read-Write Rules

1. Use Java-compatible `DataInputStream.readUTF` / `DataOutputStream.writeUTF` behavior.
2. This is modified UTF-8 with 2-byte length prefix, not raw null-terminated UTF-8.
3. Never mix with custom string encoding in milestone 1.

## 6) Endianness and Primitive Rules

1. `short/int/long` use Java big-endian wire format.
2. `boolean` uses 1 byte (`0/1`) as Java stream logic.
3. Unsigned semantics needed for some fields (for example `mapId` read as unsigned byte).

## 7) Packet Trace Logging Format

## Log schema
- `timestampIso`
- `direction` (`IN`/`OUT`)
- `state` (client state machine state)
- `mode` (`PLAIN`/`XOR`)
- `cmdSigned`
- `cmdUnsigned`
- `length`
- `payloadPreviewHex` (first N bytes)
- `note`

## Recommended line format
`2026-03-09T17:30:12.456Z IN state=KEY_WAIT mode=PLAIN cmd=-27 ucmd=229 len=8 payload=03 4E 10 22 01 ... note=key_packet`

## Required debug events
1. XOR mode activation/deactivation.
2. Special 3-byte length branch usage.
3. Decode failures (including current cmd and indices).
4. Illegal state packet blocks (packet dropped by state machine).

## 8) Compatibility Guardrails

1. Reject outbound packets not allowed by current state.
2. On impossible length (`<0`, overly large), close session and reset codec context.
3. Reset codec indices on new connection only.
4. Never allow rendering layer to read/write raw packets directly.
