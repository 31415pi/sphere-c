# Sphere C device-side animation / vendor-command audit

## Scope and conclusion

This is a read-only audit of the locally saved Olight APK, its
decompilation, the downloaded Sphere C mini-app, Bluetooth captures,
research notes, and current controller code. No radio command was sent.

The strongest result is product-specific, not merely a generic Tuya
inference: the saved Sphere C mini-app contains the device's DP schema
and its effect-picker logic. The panel selects a built-in animation by
writing `work_mode = "colour"` (DP 2) and `colour_mode` (DP 102).
`colour_mode` is an enum containing `flame` and `colour_1` through
`colour_31`. The enum labels include Cotton Candy, Aurora, Fireworks,
Fountain, Spiral, Color Wheel, Red Flash, Gradient, Carousel, and many
others.

The same schema has generic Tuya lighting DPs `scene_data` (DP 6) and
`music_data` (DP 8). The Sphere panel does not use DP 6 for its effect
picker. It does use DP 8 for microphone-driven music rhythm after
setting `work_mode = "music"`.

The APK independently establishes how these DPs travel over Bluetooth
Mesh: Tuya vendor server model `0x07d0:0x0004`, acknowledged WRITE
`c9 d0 07`, unacknowledged WRITE `ca d0 07`, READ `cc d0 07`, and
DATA/response `cd d0 07`. The exact logical effect selection is proven.
The concrete access bytes shown below are a derivation from the APK
encoder and mini-app schema, not an authenticated Sphere capture.

No artifact proves per-LED color control. The five secondary elements
are OnOff-only and their physical meaning remains unknown.

## Evidence grades

The terms in this note mean:

- **direct**: present in a Sphere C-specific local artifact or an
  authenticated local composition;
- **derived**: mechanically follows from a direct schema plus the APK's
  encoder, but has not been observed in a decrypted Sphere transaction;
- **generic**: implemented by the bundled Thingclips/Tuya SDK but not
  shown to be exercised by this Sphere panel or firmware;
- **unsupported**: no local evidence establishes the claim.

## Artifact provenance

The principal artifacts are:

| Artifact | SHA-256 |
|---|---|
| `old/apk/olight-3.2.5.apk` | `1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9` |
| Sphere mini-app `b1y/eo8LLi.js` | `024c0b833914dfd52102f4d96568ddda2be3223cf5a98b5b2c402eb18f1d8bf5` |
| Sphere English I18N JSON | `bd890661734c1bf28a3286875404213910730b63312041eab92fc9bd777a9da9` |

The mini-app is under:

```text
old/phone_external/files/Godzilla/MiniApp/
  9E6DD95017EDCABEA0673DC3A9AFAC12/
```

Its I18N filename contains product ID `d0dpga0h`, and its
`product_name` value is `Sphere C`. The same PID is recognized in the
APK's `DevicePidManage` class. The schema and panel implementation are
in the minified `Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js`.
That bundle is one physical line, so references to it below are line 1.

The APK was inspected in the existing local smali tree at
`/tmp/olight-dex.7xC3jV`. Temporary paths are not repository evidence;
the APK hash above is the reproducible input.

## Sphere C-specific DP schema

The mini-app embeds this relevant schema directly:

| ID | Code | Type/access | Meaning in this panel |
|---:|---|---|---|
| 1 | `switch_led` | bool, read/write | power |
| 2 | `work_mode` | enum, read/write | white/colour/scene/music |
| 3 | `bright_value` | value, read/write | brightness |
| 5 | `colour_data` | string, read/write | HSV color |
| 6 | `scene_data` | string, read/write | generic SoC scene DP |
| 8 | `music_data` | string, write-only | app music rhythm |
| 102 | `colour_mode` | enum, read/write | built-in Sphere effects |
| 103 | `battery_percentage` | value, read-only | battery |

This corrects an ambiguity in older research. The public-profile note
in `docs/analysis/compatibility.md:69-81` called DP 102 a scene DP.
The actual saved panel calls it `colour_mode`; operationally, it is the
built-in effect selector. DP 6 still exists separately as
`scene_data`.

The `work_mode` enum order is exact:

| Wire enum | Schema value |
|---:|---|
| `0x00` | `white` |
| `0x01` | `colour` |
| `0x02` | `scene` |
| `0x03` | `music` |

This order appears in both the Sphere schema and the SDK parser. The
parser's integer-to-name mapping is in
`bdpdqbp.smali:1246-1290`.

## Built-in effect catalog

DP 102 is ordered as `flame`, then `colour_1` through `colour_31`.
Tuya enum values are encoded by their zero-based schema index, so the
wire enum is `0x00` for `flame` and the numeric suffix for every
`colour_N`. The local I18N file supplies these labels:

| Enum | Value | Label |
|---:|---|---|
| `0x00` | `flame` | Flame |
| `0x01` | `colour_1` | Cotton Candy |
| `0x02` | `colour_2` | Aurora |
| `0x03` | `colour_3` | Fireworks |
| `0x04` | `colour_4` | Fountain |
| `0x05` | `colour_5` | Firefly |
| `0x06` | `colour_6` | Spiral |
| `0x07` | `colour_7` | Color Wheel |
| `0x08` | `colour_8` | Red Flash |
| `0x09` | `colour_9` | Confetti |
| `0x0a` | `colour_10` | Butterfly |
| `0x0b` | `colour_11` | Fireball |
| `0x0c` | `colour_12` | Gradient |
| `0x0d` | `colour_13` | Carousel |
| `0x0e` | `colour_14` | Floating |
| `0x0f` | `colour_15` | Magic Ball |
| `0x10` | `colour_16` | Lilac |
| `0x11` | `colour_17` | Sunset |
| `0x12` | `colour_18` | Vibrant |
| `0x13` | `colour_19` | Autumn |
| `0x14` | `colour_20` | Red Light |
| `0x15` | `colour_21` | Blue sky |
| `0x16` | `colour_22` | New Year (marked `_ban`) |
| `0x17` | `colour_23` | Christmas (marked `_ban`) |
| `0x18` | `colour_24` | Harvest |
| `0x19` | `colour_25` | Willow Wisp |
| `0x1a` | `colour_26` | Zombie |
| `0x1b` | `colour_27` | Snowfall |
| `0x1c` | `colour_28` | Phantom Purple |
| `0x1d` | `colour_29` | Christmas |
| `0x1e` | `colour_30` | Candy Cane |
| `0x1f` | `colour_31` | Snowfall |

The panel's visible picker is a subset. It includes flame and effect
numbers 1-4, 6-13, 15-18, 20-21, 24-26, and 28-31. It omits 5, 14,
19, 22, 23, and 27 even though those values remain in the schema.
Schema membership does not prove that every hidden value is enabled in
this firmware version.

## What the Sphere panel actually sends

The effect click handler is direct product evidence. For a device it:

1. turns on DP 1 if necessary;
2. sets DP 2 to `colour`; and
3. sets DP 102 to the selected `flame`/`colour_N` value.

For a group it publishes `{2: "colour", 102: effect}` together. The
device path invokes the two typed setters in sequence. Therefore code
should not assume that mode and effect always occupy one access PDU.

The music button sets DP 2 to `music`, starts the phone microphone
audio-to-RGB callback, and continuously writes DP 8. Its
`MusicTransformer` formats:

```text
mode:1 hex digit
hue:4 hex digits
saturation:4 hex digits
value:4 hex digits
brightness:4 hex digits
temperature:4 hex digits
```

The mini-app fixes saturation at 1000 for these updates. On the Mesh
path the SDK compresses `music_data` into a four-byte raw value. The
compression consumes the one-digit mode and the first three 16-bit
fields (hue, saturation, value); the APK code at
`bbqqdbq.smali:842-1095` packs their high bits into byte zero and their
low bytes into bytes one through three. Brightness and temperature are
not independently present in that four-byte compressed result.

The bundle contains `scene_data` only in the embedded schema; there is
no panel call site for it. It must not be presented as the mechanism for
the Sphere preset picker.

## Tuya vendor transport and DP grammar

### Model and opcodes

The SDK's `LightModeSetAction` identifies company `0x07d0`, packed
model ID `0x07d00004`, request opcode byte `0xc9`, and expected response
byte `0xcd` (`pdpbqqp.smali:7-13,58-86`). The related classes establish:

| Operation | Access opcode bytes |
|---|---|
| acknowledged WRITE | `c9 d0 07` |
| unacknowledged WRITE | `ca d0 07` |
| READ | `cc d0 07` |
| DATA/response | `cd d0 07` |

The unacknowledged values are in `bddbpbq.smali:7-39`. READ and its
model ID are in `ppbppqq.smali:7-12,35-70`.

The first WRITE parameter is command type `0x01`. It is followed by one
or more DP records. From `pdpbqqp.smali:235-1045`, the record grammar is:

```text
dp_id:u8 || dp_type:u8 || [length:u8] || value
```

| DP type | Byte | Length/value representation |
|---|---:|---|
| raw | `0x00` | length byte + raw bytes |
| bool | `0x01` | one byte |
| value | `0x02` | four-byte integer |
| string | `0x03` | length byte + string bytes |
| enum | `0x04` | one byte |
| bitmap | `0x05` | length 1/2/4 + big-endian value |

A READ parameter body is:

```text
command_type:u8 || dp_count:u8 || dp_id...
```

The command type is `0x01`, or `0x05` when the caller's flag is set
(`ppbppqq.smali:449-611`). The meaning of the flag is not established
by a Sphere-specific transaction.

The SDK also has TID-framed vendor variants for longer payloads. A
minimal implementation should reproduce the action-selection logic,
not assume every possible raw scene is sent with plain `0xc9`.

### Derived built-in-effect bytes

Combining the direct Sphere schema with the direct APK encoder produces
these **derived**, not captured, parameters:

```text
# mode = colour
01 02 04 01

# effect = flame or colour_N
01 66 04 NN

# both in one multi-DP write
01 02 04 01 66 04 NN
```

Here `0x66` is DP 102 and `NN` is the enum byte from the catalog. With
the acknowledged opcode, a combined selection would be:

```text
c9 d0 07 01 02 04 01 66 04 NN
```

Examples include Aurora (`NN=02`), Color Wheel (`NN=07`), and Candy
Cane (`NN=1e`). The app may send power and the two selection DPs as
separate writes, especially on the individual-device path.

The corresponding derived music-mode body is:

```text
01 02 04 03
```

A compressed four-byte DP 8 rhythm update has this body:

```text
01 08 00 04 XX XX XX XX
```

No local capture authenticates any of those Sphere runtime bytes. They
are suitable as codec test vectors only until a read or controlled live
write confirms the behavior and response body.

## Generic scene and music translation

The generic SIG Mesh light parser explicitly recognizes `work_mode`,
`colour_data`, `scene_data`, and `music_data`
(`bdpdqbp.smali:7-29,264-518`). It forces scene and music data to raw
vendor DPs after translating their normal Tuya string forms.

For `scene_data`, `bbqqdbq.bppdpdq`:

- parses a one-byte header followed by 13-byte scene units;
- calculates the unit count as `(byte_length - 1) / 13`;
- emits a two-byte packed header and four bytes per unit; and
- reads five 16-bit fields from each unit before choosing one of two
  compact layouts.

Those facts are visible at `bbqqdbq.smali:219-470` and later in the
same method. The semantic names of every packed bit were not recovered
from local product code. Calling them colors, speed, or transition mode
without a validated vector would overstate the evidence.

For `music_data`, the four-byte packing is substantially clearer and is
described above. Both converters also have inverse/report paths in the
same helper class.

These converters are **generic SDK capability**. DP 6 is in the Sphere
schema, but the saved Sphere UI does not exercise it. There is no saved
Sphere scene payload from which to validate the DP 6 field semantics.

## Captures: what they do and do not establish

The original `old/captures/sphere-c.btsnoop` has 251 Mesh Proxy Network
PDUs but no NetKey or AppKey. It cannot prove access opcodes or DP IDs;
see `docs/analysis/capture.md:19-35`. Its repeated 24-byte requests and
variable-length response fans are consistent with vendor reads, but
that remains length-only evidence
(`docs/analysis/compatibility.md:119-143`).

The local self-provision/configuration captures were also checked with
the locally saved keys without printing key material. The complete
application-key traffic in the full configuration capture consists of
Generic OnOff Get/Status. The other decoded traffic is configuration
traffic under the DeviceKey. No saved capture contains a decrypted
Sphere preset, scene, music, or per-secondary-element runtime command.

The only exact vendor write already documented from APK control flow is
the conditional group-association command:

```text
c9 d0 07 86 02 <group-address>
```

Its response is `cd d0 07 86 01 00`. This is group configuration, not
an effect command; see
`docs/research/mesh/post_provision_commit_diff.md:170-187`.

## Per-LED and secondary-element findings

Authenticated composition reports six elements. The primary hosts the
standard lighting models plus vendor models `0x07d0:0x0004` and
`0x07d0:0x0005`; the other five expose only Generic OnOff Server
`0x1000` (`docs/research/mesh/step4_sphere_model_map.md:51-95`).

Consequences:

- DP 102 is addressed to the primary vendor server and selects a
  firmware effect; it is not an LED index.
- A secondary element can represent only Boolean state through its
  advertised SIG model. It has no standard Lightness, RGB/HSL, or
  per-pixel model.
- The five secondaries might be channels, zones, layers, or an SDK
  template. No captured command or visible observation maps them.
- No local schema has an LED index, pixel buffer, segment map, or raw
  paint DP for this Sphere product.

The dedicated audit reaches the same conservative conclusion in
`docs/research/mesh/individual_led_audit.md:1-34,155-189`.

Therefore, built-in firmware effects are proven; individually
addressable LEDs are unsupported by current evidence.

## Current host-code gap

`src/sphere_mesh/lighting.py:1-312` implements acknowledged standard
Lightness, CTL Temperature, HSL, and xyL operations. It has no Tuya
vendor-DP codec, no DP 102 effect API, no DP 8 music sender, and no DP 6
scene codec. `src/sphere_mesh/capture.py:149-168` can label an unknown
vendor opcode and print parameters, but it does not decode Tuya DP
records.

The existing standard HSL API can stream host-generated color changes,
as discussed in `host_animation_feasibility.md`. That is separate from
asking the lamp firmware to run a DP 102 built-in effect.

## Confidence-ranked findings

### Direct / high confidence

1. Sphere C effects are selected by DP 102 `colour_mode`, with `flame`
   plus 31 numbered enum values.
2. The panel selects an effect by setting DP 2 to `colour` and DP 102
   to the effect enum.
3. The panel's music mode sets DP 2 to `music` and streams DP 8.
4. The vendor server and opcode family are `0x07d0:0x0004` and
   `c9/ca/cc/cd d0 07`.
5. The host controller implements guarded DP 102 effect selection.
6. A later live experiment validated DP 102 selection, readback, and
   restoration; visible appearance and DP 2 activation remain untested.

### Derived / strong

1. A combined acknowledged effect activation encodes as
   `c9 d0 07 01 02 04 01 66 04 NN`.
2. A four-byte compressed music update has parameters
   `01 08 00 04 XX XX XX XX`.
3. A safe read-only DP query can use opcode `cc d0 07` with the READ
   body grammar above, provided destination, keys, binding, sequence,
   and response handling are correct.

### Generic or incomplete

1. DP 6 scene compression exists in the SDK and Sphere schema, but its
   product semantics and actual Sphere use are unvalidated.
2. The SDK's longer/TID-framed vendor paths need a focused codec audit
   before supporting arbitrary scene payloads.
3. Hidden DP 102 enum values may work, but schema presence alone is not
   proof that the panel's firmware enables them.

### Unsupported

1. Per-LED or per-pixel color addressing.
2. A physical mapping for the five secondary OnOff elements.
3. Treating DP 6 as the Sphere effect-picker command.
4. Claiming the derived bytes were observed on the wire.

## Recommended next evidence step

First add an offline decoder for the `0xcd` DATA record grammar and test
it against synthetic vectors. Then perform a read-only vendor query for
DPs 2 and 102 on the primary vendor model, recording the authenticated
response. Only after that succeeds should a controlled acknowledged
write select one already-visible effect, with the current values saved
for restoration. Keep secondary-element investigation separate; it is
not needed to validate device-side effects.
