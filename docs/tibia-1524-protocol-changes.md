# Tibia 15.24 Protocol Changes

This document describes the protocol changes made to support Tibia 15.24 and the fixes applied to the proxy.

---

## 1. Packet Framing

### Old format
```
[size: 2 bytes] [sequence_number: 4 bytes] [XTEA encrypted payload]
```
- `size` = number of payload bytes following the 2-byte header
- Total packet length on wire = `size + 2`
- After XTEA decrypt: `[inner_size: 2 bytes] [payload]`
- `PayloadDataPosition` = 8 (skipping 2-byte chunk header + 4-byte seq + 2-byte inner size)

### New format (15.24)
```
[chunk_count: 2 bytes] [sequence_number: 4 bytes] [XTEA encrypted data: chunk_count * 8 bytes]
```
- `chunk_count` = number of 8-byte XTEA blocks
- Total packet length on wire = `chunk_count * 8 + 6`
- After XTEA decrypt: `[padding_count: 1 byte] [payload] [padding: padding_count bytes]`
- `PayloadDataPosition` = 7 (skipping 2-byte chunk header + 4-byte seq + 1-byte padding count)

### Size calculation changes
| Location | Old | New |
|---|---|---|
| `BeginReceiveServerCallback` | `ToUInt16(buf, 0) + 2` | `ToUInt16(buf, 0) * 8 + 6` |
| `BeginReceiveClientCallback` | `ToUInt16(buf, 0) + 2` | `ToUInt16(buf, 0) * 8 + 6` |
| All `BeginReceive` calls | read `2` bytes initially | read `6` bytes initially |

---

## 2. NetworkMessage: PrepareToParse

Reads the decrypted payload for parsing.

### Old behaviour
- Read 2-byte inner size at position 6
- `Size = inner_size + PayloadDataPosition`
- `Position = 6`, then `ReadUInt16()` advanced it to 8

### New behaviour
- Read 1-byte padding count from `buffer[6]`
- `Position = PayloadDataPosition` (7)
- `Size = Size - padding_count`

### Compression fix
When the packet has the `CompressedFlag` set in the sequence number, the payload is zlib-deflated. After inflating, `Size` was not updated to reflect the smaller decompressed size, leaving garbage bytes at the end of the buffer. Fix: set `Size = Position` immediately after writing the decompressed data.

```csharp
Write(outBuffer, 0, (uint)zStream.next_out_index);
Size = Position;  // <-- added: trim Size to actual decompressed data
paddingCount = 0;
```

---

## 3. NetworkMessage: PrepareToSend

Re-encrypts the payload for sending.

### Old behaviour
- Wrote 2-byte inner size at position 6
- Called `Xtea.Encrypt`, which padded to a multiple of 8
- Wrote `(_size - 2)` as the 2-byte outer size at position 0

### New behaviour
- Computes `padding_count = (8 - ((Size - 6) % 8)) % 8`
- Writes `padding_count` as 1 byte at position 6
- Calls `Xtea.Encrypt` (pads to multiple of 8 automatically)
- If XTEA key is `null` (pre-login), manually adds `padding_count` to `_size` to keep chunk count correct
- Writes `(_size - 6) / 8` as the 2-byte chunk count at position 0

---

## 4. Login Packet: AssetsHash

The login packet field after `ClientVersion` changed type.

| Field | Old | New |
|---|---|---|
| After `ClientVersion` | `DatRevision` (uint16) | `AssetsHash` (string — SHA256 of assets.json) |

Updated in `ClientPackets/Login.cs`: property, `ParseFromNetworkMessage`, and `AppendToNetworkMessage`.

---

## 5. RSA Start Index (Connection.cs)

The start of the RSA-encrypted block in the login packet is no longer at a fixed offset because `Version` and `AssetsHash` are variable-length strings.

### Old
```csharp
var rsaStartIndex = _client.VersionNumber >= 124010030 ? 31 : 18;
```

### New
```csharp
_clientInMessage.Seek(16, SeekOrigin.Begin); // skip fixed-size fields
var clientVersion = _clientInMessage.ReadString();
var assetsHash = _clientInMessage.ReadString();
_clientInMessage.Seek(1, SeekOrigin.Current); // skip ClientPreviewState
var rsaStartIndex = (int)_clientInMessage.Position;
```

---

## 6. ImpactTracking Logging

Real-time damage/heal logging added to `BeginReceiveServerCallback`. After XTEA decryption, if the first payload byte is `0xCC` (`ServerPacketType.ImpactTracking`), the packet is parsed and logged:

```
[ImpactTracking] DamageDealt: 312 (Fire)
[ImpactTracking] DamageReceived: 85 (Physical) from Demon
[ImpactTracking] Heal: 150 HP
```

Packet structure (`ImpactAnalyzer` sub-types):

| Type byte | Name | Fields |
|---|---|---|
| 0 | Heal | `amount` (uint32) |
| 1 | DamageDealt | `amount` (uint32), `element` (byte) |
| 2 | DamageReceived | `amount` (uint32), `element` (byte), `target` (string) |

If an unknown sub-type is encountered, the raw decrypted bytes are logged for debugging.

---

## 7. Target Framework

| Project | Old | New |
|---|---|---|
| `Apps/Record` | `netcoreapp3.1` | `net8.0` |
| `Apps/LogReader` | `net6.0` | `net8.0` |
