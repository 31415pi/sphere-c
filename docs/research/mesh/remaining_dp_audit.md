# Remaining Sphere C datapoint audit

## Scope and evidence

This audit covers DPs 7, 33, 37, 38, 101, 104, 105, 106, and 107.
DP 6 is mentioned only to distinguish its opaque scene format from
these DPs. Music DP 8 is excluded.

The product-specific source is the saved mini-app bundle
[`eo8LLi.js`](../../../old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js).
Its schema declares these names, types, ranges, and access modes.
The bundle's actual UI code has setter calls for DP 101 and DP 106,
but not for the remaining entries. The APK codec behavior below is
the generic Thingclips/Tuya record encoder and decoder documented in
[`vendor_dp_response_codec.md`](vendor_dp_response_codec.md); no
DP-specific firmware semantics are established by that codec alone.
The live snapshot in
[`vendor_dp_inventory.md`](vendor_dp_inventory.md) confirms DP 7,
DP 101, and DP 104 records and current values, but does not confirm
their write behavior.

“Schema capability” is kept separate from an actual mini-app call
site: a declared `rw` mode does not prove the Sphere firmware accepts
a guessed write.

## Per-DP findings

| DP | Product schema and generic DATA shape | Actual Sphere UI behavior | Probe recommendation |
|---:|---|---|---|
| 7 `countdown` | Value, `0..43200` seconds, step 1. Generic Tuya value record is `07 02` followed by a four-byte big-endian integer. A live read returned `0`. | Schema is `rw`, but no panel setter for DP 7 was found. The visible countdown panel reads and writes DP 101 instead. | A same-value write of `0` is the only reasonable candidate for a cautious probe: it represents no active countdown and should be a no-op. Still unverified; read before and after, and do not try nonzero durations because they can turn the lamp off later. |
| 33 `power_memory` | Raw, max 255 bytes. Generic shape: `21 00 LEN DATA...`; schema permits an opaque byte string up to 255 bytes. | `rw` schema entry only. No setter, getter, or panel reference found. | Block. Payload grammar and power-restoration behavior are unknown; even an empty or same-looking payload may alter persistent startup behavior. |
| 37 `rtc_timer` | Raw, max 128 bytes. Generic shape: `25 00 LEN DATA...`. | `rw` schema entry only. No setter or panel reference found. | Block. Unknown timer command format and potential scheduled actions; no safe same-value can be constructed. |
| 38 `timer_report` | Raw, max 128 bytes. Generic shape: `26 00 LEN DATA...`. | `rw` schema entry only. No panel call site. The name suggests a report, but schema access says writable and does not settle direction or encoding. | Block. Opaque payload and unclear direction; schema name is insufficient to justify any write. |
| 101 `countdown_2` | Value, `0..43200` seconds, step 1. Generic shape: `65 02` plus four-byte big-endian seconds. Live read returned `0`. | Confirmed panel behavior: the countdown picker spans 00–719 minutes and 00–59 seconds, computes total seconds, and calls `countdown_2.set(seconds)`. It displays the current countdown and schedules a local display update. | The same-value write `0` is a cautious candidate to test the setter path, with before/after readback. A positive duration is reversible only by writing `0`, but has an actual delayed power-off effect; keep it blocked unless a controlled short countdown and cancellation are explicitly intended. |
| 104 `sleep_switch` | Bool. Generic shape: `68 01 00` or `68 01 01`. Live read returned false. | `rw` schema entry only; no UI read or setter found. Display name is `0min`, which does not establish what enabling it does. | Block. Setting true may cause sleep or timed shutdown; even same-value false can have undocumented firmware effects. |
| 105 `game_flag` | Bool. Generic shape: `69 01 00` or `69 01 01`. | `rw` schema entry only; no call site found. Translated label means “whack-a-mole,” but physical behavior is unknown. | Block. Unknown mode/action effects; no evidence that a same-value write is inert. |
| 106 `factory_mode_reset` | Bool, write-only in the schema (`wr`). Generic shape: `6a 01 01` for the value used by the panel. | Confirmed destructive UI action: after a confirmation dialog, the individual-device path calls `factory_mode_reset.set(true)` and requests a DP refresh. Group path publishes `{106: true}`. The translated label says restore factory lighting modes. | Block all probes. This is explicitly an action, not a state setting; same-value is not meaningful and may reset customized lighting presets. It is not Mesh Node Reset, but still has persistent user-visible consequences. |
| 107 `water_mode` | Bool. Generic shape: `6b 01 00` or `6b 01 01`. | `rw` schema entry only; no call site found. The translated label “water mode” does not explain the effect. | Block. Unknown behavior, no established neutral value or reversible control. |

The generic APK DATA decoder consumes raw records as a one-byte length
followed by that many bytes, bool as one byte, and value as four
big-endian bytes. Its writer uses the corresponding schema type. This
describes transport syntax only; it does not establish whether any of
the above payloads are accepted by Sphere firmware. For DPs 33, 37,
and 38 in particular, type-correct bytes are not semantically valid
payloads unless their inner format is known.

## DP 6 boundary

DP 6 `scene_data` is a separate generic scene structure. Generic APK
code has scene compression support, but the saved Sphere panel does
not use DP 6 for its built-in effect picker or the interactive
countdown. This audit found no reason to infer the raw formats of DPs
33, 37, or 38 from DP 6's scene codec.

## Summary

Low-risk candidates are limited to writing the already-read neutral
value zero for DP 7 and DP 101, then confirming readback. Both are
countdowns, so nonzero values create a delayed off action. Leave DPs
33, 37, 38, 104, 105, 106, and 107 blocked pending product-defined
payload semantics or a controlled observation of the actual UI
operation. DP 106 should remain blocked even for a same-value probe
because it is a one-shot reset action.

## Live DP 101 result

On 2026-09-27, the locally provisioned Sphere reported DP 101=`0`.
The controller then sent the acknowledged payload:

```text
c9 d0 07 01 65 02 00000000
```

The Sphere returned the exact DATA echo:

```text
cd d0 07 01 65 02 00000000
```

A separate read again returned DP 101=`0`, and Generic OnOff remained
on. Automatic write retries were disabled. This validates the DP 101
wire encoding, acknowledgment, neutral write, and readback. It does not
validate a positive countdown or the eventual power-off action. DP 7
was not written.
