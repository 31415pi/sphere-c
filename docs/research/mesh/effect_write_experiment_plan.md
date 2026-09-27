# Sphere C effect-write experiment plan

## Evidence boundary

No vendor WRITE has been sent by this project. Product UI behavior and
the APK codec below are **derived/static evidence**; the current values
and earlier query behavior are **live read evidence**. The proposed
write is a plan, not a live result.

## DP 102 values

The saved Sphere C mini-app schema defines DP 102 `colour_mode` as
`flame` (`00`) followed by `colour_1` through `colour_31` (`01`..`1f`).
The panel visibly offers Flame and these numbered values:

| Value | Name | Panel label |
|---:|---|---|
| `00` | `flame` | Flame |
| `01` | `colour_1` | Cotton Candy |
| `02` | `colour_2` | Aurora |
| `03` | `colour_3` | Fireworks |
| `04` | `colour_4` | Fountain |
| `06` | `colour_6` | Spiral |
| `07` | `colour_7` | Color Wheel |
| `08` | `colour_8` | Red Flash |
| `09` | `colour_9` | Confetti |
| `0a` | `colour_10` | Butterfly |
| `0b` | `colour_11` | Fireball |
| `0c` | `colour_12` | Gradient |
| `0d` | `colour_13` | Carousel |
| `0f` | `colour_15` | Magic Ball |
| `10` | `colour_16` | Lilac |
| `11` | `colour_17` | Sunset |
| `12` | `colour_18` | Vibrant |
| `14` | `colour_20` | Red Light |
| `15` | `colour_21` | Blue sky |
| `18` | `colour_24` | Harvest |
| `19` | `colour_25` | Willow Wisp |
| `1a` | `colour_26` | Zombie |
| `1c` | `colour_28` | Phantom Purple |
| `1d` | `colour_29` | Christmas |
| `1e` | `colour_30` | Candy Cane |
| `1f` | `colour_31` | Snowfall |

Schema values hidden from the panel are `05` Firefly, `0e` Floating,
`13` Autumn, `16` New Year, `17` Christmas, and `1b` Snowfall.
New Year and the first Christmas entry are marked `_ban` in the
mini-app. Being in the schema does not prove hidden entries work on the
current firmware.

## Write ordering and derived bytes

The individual-device UI path turns DP 1 on if needed, then calls the
typed setter for DP 2=`colour`, then the setter for DP 102=the selected
effect. These are sequential writes, not proven to be one access PDU.
The group path publishes DPs 2 and 102 together. A combined write is
codec-derived and representable, but there is no captured Sphere write
to prove that the device path batches them.

For the acknowledged vendor opcode `c9 d0 07`, the APK-derived body
for DP 2=`colour` is `01 02 04 01`; DP 102=`NN` is `01 66 04 NN`.
Together they would be `01 02 04 01 66 04 NN`. The corresponding
unacknowledged opcode is `ca d0 07`. These encodings have not been
validated by a live write.

## Acknowledgment and correlation expectations

The APK action expects response opcode `cd d0 07` to an acknowledged
write. This is derived from static action/dispatcher code; no local
Sphere write establishes whether the response is empty, contains the
written DP records, or is followed by a status snapshot. A READ did
live-return an empty DATA message (`cd d0 07 01`) followed by a
segmented five-record snapshot, but that read behavior must not be
assumed for WRITE.

Plain `c9`/`cd` parameters contain no request ID or TID. The APK waiter
allows five seconds and matches response opcode plus a masked node
source (`source & 0xfff8`), without checking command type or returned
DP IDs. A careful experiment should serialize vendor transactions per
node and correlate by unicast source, destination, security context,
expected `cd d0 07` opcode, command type (`01` or `05`), and bounded
receive window. Treat a matching DATA as likely related, not uniquely
bound to the request; unsolicited publication cannot be excluded.
Record empty acknowledgments and later DATA separately. Do not interpret
Mesh segment acknowledgments as the vendor-model acknowledgment.

## Safest reversible probe with current live state

**Live baseline:** DP 102 was read as `07` (`colour_7`, Color Wheel).
DP 2 was omitted by focused and combined reads, so its current value is
unknown. No DP 102 write has been tested.

The lowest-change acknowledged probe is a same-value DP 102 write
(`01 66 04 07`, with opcode `c9 d0 07`) followed by a serialized
read-only query and visual observation. It avoids guessing DP 2 and
should leave the selected effect unchanged if firmware accepts the
derived codec. It tests write acceptance/restoration of the known value,
but does not test changing the visible effect.

If the intended test must visibly change the effect, DP 2 must first be
set to `colour` to mirror the individual-device app path, then DP 102
can be set to a visible value such as `06` (Spiral), and finally
restored to live-baseline `07` (Color Wheel). However, because DP 2 is
currently unknown and unreadable in observed replies, that sequence
cannot guarantee restoration of the prior mode. Record the mode change
as a reversibility limitation; do not claim the whole pre-test state
was restored. Stop on timeout, unexpected status, or unexpected lamp
behavior, and do not retry an acknowledged write blindly.

## Sources

- [`host_device_animation_audit.md`](host_device_animation_audit.md):
  product schema, picker labels, per-device and group write paths.
- [`vendor_dp_response_codec.md`](vendor_dp_response_codec.md):
  write/read opcodes, DATA grammar, response waiter and correlation.
- [`vendor_dp_live_query.md`](vendor_dp_live_query.md):
  authenticated read result, DP 102=`07`, and DP 2 omission.
- [`vendor_dp_inventory.md`](vendor_dp_inventory.md):
  evidence grades and live-versus-derived inventory.
