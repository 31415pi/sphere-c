# Sphere C vendor datapoints

This is the living, reader-oriented summary of the Sphere C's
Tuya/Olight datapoints. Update it whenever a live read, controlled
write, APK trace, or product-schema discovery changes the evidence.

Last updated: 2026-09-27.

The detailed evidence ledger is
[the vendor-DP inventory](../research/mesh/vendor_dp_inventory.md).
That document records sources and confidence separately for every DP.

## Evidence boundary

The complete 17-DP product schema comes from the saved Sphere C
mini-app. Executable code reversed from the Olight APK establishes the
wire codec and routing behavior. Authenticated queries against the
locally owned Sphere establish the smaller live-read set.

DP 102 selection has been written and restored on the locally owned
Sphere. Other write encodings below remain derived from
product-specific UI call sites and the APK encoder rather than
live-confirmed behavior.

Use these evidence labels when updating this page:

- **Live:** observed in authenticated traffic from our Sphere C.
- **Product:** present in the Sphere C mini-app schema or UI code.
- **APK:** recovered from executable Olight/Thingclips code.
- **Generic:** supported by the SDK but not proven for this product.

## Vendor transport

The primary server is vendor model `0x07d0:0x0004`.

| Operation | Access opcode |
|---|---|
| Acknowledged write | `c9 d0 07` |
| Unacknowledged write | `ca d0 07` |
| Read | `cc d0 07` |
| Data/status | `cd d0 07` |

An ordinary datapoint record is:

```text
dp_id:u8 || dp_type:u8 || encoded_value
```

| Type | Byte | Encoding |
|---|---:|---|
| Raw | `00` | `length:u8 || bytes` |
| Boolean | `01` | one byte |
| Value | `02` | four-byte big-endian integer |
| String | `03` | `length:u8 || UTF-8 bytes` |
| Enum | `04` | one byte |
| Bitmap | `05` | length 1, 2, or 4 plus bytes |

READ parameters contain a command type, count, and requested IDs.
DATA parameters contain a command type followed by records to the end;
there is no response count or transaction ID.

## Product schema

“Access” below is the product-schema declaration. It does not guarantee
that a DP is transported as an ordinary vendor record. Power,
brightness, color temperature, and static color are normally mapped to
standard Bluetooth Mesh models by the APK.

| DP | Name | Access | Meaning and status |
|---:|---|---|---|
| 1 | `switch_led` | R/W | Main power. Product UI writes it; normally SIG Generic OnOff. |
| 2 | `work_mode` | R/W | White, colour, scene, or music. Product UI writes it; live reads omitted it. |
| 3 | `bright_value` | R/W | Brightness `10..1000`; normally SIG Light Lightness. |
| 5 | `colour_data` | R/W | Static HSV color; normally converted to SIG Light HSL. |
| 6 | `scene_data` | R/W | Generic compressed Tuya scene. Product use is unproven. |
| 7 | `countdown` | R/W | Countdown to off, `0..43200` seconds. Live-read as zero. |
| 8 | `music_data` | Write | Phone-microphone rhythm stream; compressed to four raw bytes. |
| 33 | `power_memory` | R/W | Power-restoration behavior; opaque raw payload. |
| 37 | `rtc_timer` | R/W | Local timer configuration; opaque raw payload. |
| 38 | `timer_report` | R/W | Local timer status; opaque raw payload. |
| 101 | `countdown_2` | R/W | Countdown used by the panel. Live-read as zero. |
| 102 | `colour_mode` | R/W | Built-in firmware effect. Live-read as Color Wheel. |
| 103 | `battery_percentage` | Read | Battery `0..100%`. Live-read as 100%. |
| 104 | `sleep_switch` | R/W | Sleep-mode enable. Live-read as false. |
| 105 | `game_flag` | R/W | “Whack-a-mole” game flag; behavior untested. |
| 106 | `factory_mode_reset` | Write | Restore factory lighting modes; not Mesh Node Reset. |
| 107 | `water_mode` | R/W | Water-mode flag; behavior untested. |

DP 4, color temperature, appears in generic Tuya material but is not
in the saved Sphere C schema. The Sphere exposes standard Mesh CTL
Temperature instead.

## Live-readable records

Reads requesting DP 2, DP 102, or both produced the same result. The
Sphere first sent an empty DATA acknowledgment and then a segmented
five-record snapshot:

| DP | Type | Live value | Meaning |
|---:|---|---:|---|
| 7 | value | 0 | No active countdown |
| 101 | value | 0 | No active panel countdown |
| 102 | enum | 7 | `colour_7`, Color Wheel |
| 103 | value | 100 | Battery 100% |
| 104 | bool | false | Sleep mode disabled |

This confirms that these five are readable vendor records. It also
suggests that the READ ID list triggers a general status refresh rather
than strictly filtering the reply. DP 2's omission does not prove that
it is write-only.

## Best-understood writes

### Built-in effect

The Sphere UI selects an effect by setting:

```text
DP 2   = colour
DP 102 = flame or colour_1..colour_31
```

The derived combined acknowledged message is:

```text
c9 d0 07 01 02 04 01 66 04 NN
```

`NN=00` is Flame. `01..1f` select `colour_1..colour_31`.
The product's visible picker omits some values that remain in the
schema, so schema membership alone does not prove firmware support.

The local service now exposes a named effect catalog and a guarded
selection command. `sphere-meshctl effect get` reads the vendor
snapshot. `effect select aurora` (also `effect set aurora`) writes
DP 102 alone; `effect activate aurora` first writes DP 2=`colour`
and then DP 102, matching the individual-device panel order. Only
visible picker values are writable. The 26 visible effects have unique
slugs; canonical schema names such as `colour_2` also work. Catalog
and current-effect responses contain `slug` and `canonical` metadata.
Hidden schema values have no slug. Each write expects DATA and is
sent once, without automatic retries. An empty DATA acknowledgment
counts as a transport acknowledgment, not proof of a visible change.
DP 102 writes are live-validated. A same-value write of Color Wheel
(`07`) was acknowledged, Aurora (`02`) was selected and read back, and
Color Wheel was restored and read back. A later live activation was
visually confirmed. The friendly `aurora` name was then independently
live-tested and read back as canonical `colour_2`.

| Value | Slug | Product label |
|---:|---|---|
| 0 | `flame` | Flame |
| 1 | `cotton-candy` | Cotton Candy |
| 2 | `aurora` | Aurora |
| 3 | `fireworks` | Fireworks |
| 4 | `fountain` | Fountain |
| 6 | `spiral` | Spiral |
| 7 | `color-wheel` | Color Wheel |
| 8 | `red-flash` | Red Flash |
| 9 | `confetti` | Confetti |
| 10 | `butterfly` | Butterfly |
| 11 | `fireball` | Fireball |
| 12 | `gradient` | Gradient |
| 13 | `carousel` | Carousel |
| 15 | `magic-ball` | Magic Ball |
| 16 | `lilac` | Lilac |
| 17 | `sunset` | Sunset |
| 18 | `vibrant` | Vibrant |
| 20 | `red-light` | Red Light |
| 21 | `blue-sky` | Blue sky |
| 24 | `harvest` | Harvest |
| 25 | `willow-wisp` | Willow Wisp |
| 26 | `zombie` | Zombie |
| 28 | `phantom-purple` | Phantom Purple |
| 29 | `christmas` | Christmas |
| 30 | `candy-cane` | Candy Cane |
| 31 | `snowfall` | Snowfall |

Only Aurora has been visually checked so far. The names and values for
the other entries come from the saved product picker, and their host
encoding is covered by the offline test suite.

### Music mode

The UI sets DP 2 to `music` and continuously streams DP 8. The APK
compresses each microphone-derived update to four raw bytes:

```text
01 08 00 04 XX XX XX XX
```

The packed value carries mode, hue, saturation, and value. The product
UI fixes saturation at 1000.

### Other product write paths

- DP 101 is set to a countdown duration in seconds. The local service
  exposes guarded get, set, and cancel operations for `0..43200`
  seconds. A live same-value zero write was acknowledged and read back;
  positive values may turn the lamp off later and remain untested.
- DP 106 is set to true by “Restore Factory Colors.” It restores
  lighting presets, not mesh ownership or provisioning state.
- DP 1, DP 3, and DP 5 are written through the ordinary UI but are
  normally routed through standard Bluetooth Mesh models.

DPs 6, 7, 33, 37, 38, 104, 105, and 107 are declared writable by the
schema, but their product write behavior is not yet established well
enough to expose as control operations.

Music DP 8 is deliberately deferred. Its microphone-driven app mode
works poorly enough that it is not an initial control priority.

## Second vendor model

Model `0x07d0:0x0005` is Tuya's vendor client; `0x0004` is the server
used for all DP operations described above. The Olight APK binds only
`0x0004` and contains no `0x0005`-specific Sphere action or safe query.
The local controller acts as the external client, so it should not bind
or address the Sphere's embedded client as another DP server.

## Known exclusions

- DP 102 selects a firmware effect; it is not an LED index.
- No known DP exposes a pixel buffer, LED index, or segment map.
- DP 6 is generic scene data, not the Sphere effect picker.
- DP ID 0 is used by SDK code as an all-DP query convention; it is not
  a product-schema DP.
- The `86 02 <group>` vendor command is group association, not a DP.

## Next updates

Update this page after each controlled experiment. In particular,
record the exact request, acknowledged response, visible device change,
restoration result, and whether a power cycle preserves the value.
The first proposed write experiment should save the current effect,
select one visible effect through DPs 2 and 102, read back status, and
restore the original effect.
