# Record: Resolving DamageDealt Target Name

## Problem

The `ImpactTracking` packet for `DamageDealt` (type 1) does not include the target creature name — its payload is only `amount` (uint32) + `element` (byte). The creature name is only available in the `Message` packet with mode `DamageDealed`, as the text log line:

```
A grim reaper loses 945 hitpoints due to your critical attack.
```

Neither packet alone is sufficient for the CSV:

| Packet | Has target | Has element |
|---|---|---|
| `ImpactTracking` (DamageDealt) | No | Yes |
| `Message` (DamageDealed) | Yes | No |

## Solution

Both packets arrive in the **same server message batch**. `ScanImpactTracking` already iterates every packet in the message, so a single pass can collect both types and join them on `amount` before writing the CSV row.

### Sketch

```csharp
// Collect both packet types during the scan loop
var impactPackets = new List<ImpactTracking>();   // DamageDealt entries
var messagePackets = new List<Message>();          // DamageDealed entries

// After the loop, pair them:
foreach (var impact in impactPackets)
{
    var match = messagePackets.FirstOrDefault(m => m.FirstValue == impact.Amount);
    var target = match != null ? ParseTargetFromText(match.Text) : string.Empty;
    // write CSV row with impact.Amount, impact.Element, target
}
```

`ParseTargetFromText` extracts the creature name from the log line — the text always starts with the creature name before `" loses "`.

## AoE Caveat

Matching on `amount` alone is ambiguous when multiple creatures take the **same damage value** in the same batch (e.g. AoE spells hitting several enemies for identical amounts). In that case there is no reliable way to pair packets without additional context.

Options to consider:
- Accept the ambiguity and assign the first unmatched `Message` entry — fine for single-target combat, wrong for AoE
- Leave `target` empty for any batch where multiple `DamageDealt` impacts share the same amount
- Track creature positions: the `Message` packet includes a `Position` field; correlate with known creature positions if available
