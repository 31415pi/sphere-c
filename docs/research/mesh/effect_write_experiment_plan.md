# Sphere C effect-write experiment

## Evidence boundary

This document began as the experiment plan. The DP 102-only sequence
was completed on 2026-09-27; its result is recorded below. Product UI
behavior and untested DP 2 encoding remain **derived/static evidence**.

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

## Live result

The experiment used the locally provisioned primary element at
`0x0118`. Initial state was powered on, and a read returned DP 102=`07`
(Color Wheel). DP 2 remained absent. No DP 2 write was sent.

The following acknowledged access payloads were sent once each, with
automatic retries disabled:

```text
# same-value probe
c9 d0 07 01 66 04 07

# select Aurora
c9 d0 07 01 66 04 02

# restore Color Wheel
c9 d0 07 01 66 04 07
```

Each write received a DATA echo containing the written record:

```text
cd d0 07 01 66 04 07
cd d0 07 01 66 04 02
cd d0 07 01 66 04 07
```

Separate READ transactions confirmed DP 102 values `07`, then `02`,
then restored `07`. The initial and final Generic OnOff state was on.
No person was present to observe the lamp, so this proves write
acceptance, state change, readback, and restoration but does not prove
that the visible animation changed to the expected appearance.

This first live result validates the derived plain DP 102 WRITE grammar
and shows that a successful write response echoes DP 102 directly. It
did not yet validate DP 2=`colour`, `effect activate`, hidden effects,
or power-cycle persistence.

## Live activation and visible effect

A second experiment on 2026-09-27 tested the app's sequential
individual-device path. Before activation, the Sphere was on and showed
the previously configured static red output. Standard model reads were
Lightness 1956, Temperature 800 K, and HSL lightness 3637, hue 1820,
saturation 65535. DP 102 was still Color Wheel (`07`).

The controller sent DP 2=`colour` and then DP 102=`colour_7` as two
separate acknowledged writes:

```text
c9 d0 07 01 02 04 01
c9 d0 07 01 66 04 07
```

The Sphere echoed both records exactly:

```text
cd d0 07 01 02 04 01
cd d0 07 01 66 04 07
```

The Sphere remained static red. The mini-app treats Color Wheel as an
interactive color-selection mode, so this is consistent with retaining
the last chosen static color rather than starting an animation.

With DP 2 now confirmed as `colour`, the controller selected Aurora:

```text
c9 d0 07 01 66 04 02
cd d0 07 01 66 04 02
```

The user visually confirmed that the Sphere began animating. A separate
READ returned DP 102=`02`, and Generic OnOff remained on. Repeated
standard Lightness, Temperature, and HSL reads did not change, showing
that those SIG states do not report the instantaneous animation output.

This validates the sequential DP 2/DP 102 activation path and the
visible execution of at least one built-in effect. The Sphere was left
running Aurora. Hidden effects, combined multi-DP activation, and
power-cycle persistence remain untested.

After adding human-readable effect aliases, a final same-value live
test sent `effect select aurora`. The service resolved it to
`colour_2`, received an exact DP 102=`02` acknowledgment, and read the
same value back in a separate query. This validates the alias path
without changing the running effect.

## Sources

- [`host_device_animation_audit.md`](host_device_animation_audit.md):
  product schema, picker labels, per-device and group write paths.
- [`vendor_dp_response_codec.md`](vendor_dp_response_codec.md):
  write/read opcodes, DATA grammar, response waiter and correlation.
- [`vendor_dp_live_query.md`](vendor_dp_live_query.md):
  authenticated read result, DP 102=`07`, and DP 2 omission.
- [`vendor_dp_inventory.md`](vendor_dp_inventory.md):
  evidence grades and live-versus-derived inventory.
