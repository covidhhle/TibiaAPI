# Record: ImpactTracking CSV Export

## Overview

Record writes a real-time CSV of all `ImpactTracking` events (damage dealt, damage received, healing) alongside the `.oxr` recording. The CSV is intended for external analytics — specifically as a replacement for the OCR-based log capture in [tibia_parser](https://github.com/covidhhle/tibia_parser), providing exact values directly from the protocol.

---

## Output Format

```csv
timestamp_ms,event_type,amount,element,source,target
1743426125000,healing_received,150,,,
1743426125312,damage_dealt,312,Fire,,
1743426125841,damage_taken,85,Physical,Demon,
```

| Column | Type | Notes |
|---|---|---|
| `timestamp_ms` | Unix ms UTC | `DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()` — use `pd.to_datetime(df['timestamp_ms'], unit='ms', utc=True)` in pandas |
| `event_type` | string | `healing_received`, `damage_dealt`, `damage_taken` — matches `CombatEvent.event_type` in tibia_parser |
| `amount` | int | Damage or heal value |
| `element` | string | `Physical`, `Fire`, `Earth`, `Energy`, `Ice`, `Holy`, `Death`, `Healing`, `Drown`, `LifeDrain`, `Undefined` — empty for `healing_received` |
| `source` | string | Attacker name for `damage_taken`, empty otherwise |
| `target` | string | Reserved for future use (see [record-damage-dealt-target.md](record-damage-dealt-target.md)) |

---

## File Lifecycle

While a session is active the file has a `.current.csv` suffix so consumers can distinguish live from completed sessions:

```
Recordings/
  31_3_2026__14_22_05.oxr               ← recording
  31_3_2026__14_22_05.impact.current.csv ← live (being written)
```

On shutdown the `.current.csv` file is renamed:

```
Recordings/
  31_3_2026__14_22_05.oxr
  31_3_2026__14_22_05.impact.csv         ← completed
```

The writer uses `AutoFlush = true`, so rows are flushed to disk immediately and can be tailed in real time.

---

## Usage

```
Record [options]

  -i=<folder>, --impactcsv=<folder>
      Folder to write the impact CSV into.
      Defaults to the Recordings directory next to the binary.
      The filename is always auto-generated from the session start time.
```

Example:

```bash
./Record --impactcsv=/home/user/tibia_parser/data
```

Produces `/home/user/tibia_parser/data/31_3_2026__14_22_05.impact.csv`.

---

## Implementation Notes

### Why not use IsServerPacketParsingEnabled

Enabling full server packet parsing caused noise — `ParseServerMessage` logs `Error`-level output for any unrecognised or malformed packet, which is Record's default log level. There is no per-packet-type parsing filter in the library.

Instead, `ScanImpactTracking` runs a silent local parse pass directly on the raw decrypted bytes received in `OnReceivedServerMessage`. It iterates all packets using `ServerPacket.CreateInstance` + `ParseFromNetworkMessage` inside a single `try/catch`, logging nothing on failure. This correctly handles `ImpactTracking` packets that arrive bundled after other packet types in the same server message — the original raw-byte approach only checked the first opcode in the buffer (`buf[7]`), which is why `DamageReceived` and `Heal` were missed.

### Missing target for damage_dealt

The `ImpactTracking` DamageDealt packet (type 1) contains only `amount` and `element` — no target name. The creature name is available in a separate `Message` packet (mode `DamageDealed`) that arrives in the same server message batch. Pairing the two on `amount` is possible but ambiguous for AoE attacks. See [record-damage-dealt-target.md](record-damage-dealt-target.md).
