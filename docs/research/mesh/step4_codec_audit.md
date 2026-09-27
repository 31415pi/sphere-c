# Step 4 lighting codec audit

## Scope and source identity

Step 4 needs acknowledged Get/Set/Status support for Light
Lightness, Light CTL, Light CTL Temperature, Light HSL, and Light
xyL.

The installed package is `btmesh 0.10.3`. Its
`direct_url.json` pins commit
`a8f7bbbebaeba124d7d08cb16122055074f2ef46`, as does the root
`pyproject.toml`. The installed source is under
`.venv/lib/python3.14/site-packages/btmesh/`.

`old/vendor/ha-bluetooth-mesh/` is a tracked source snapshot rather
than an independent Git worktree. Its lighting portion of
`src/btmesh/access.py` matches the installed package. Its only
current differences in that file are unrelated configuration
publication and Network Transmit codecs. Its tests are therefore a
useful description of the installed lighting codec.

## Sphere composition boundary

The observed Sphere C composition and the Olight binding allowlist
contain these relevant server IDs:

| Model | Server ID | Observed |
|---|---:|:---:|
| Light Lightness | `0x1300` | yes |
| Light CTL | `0x1303` | yes |
| Light CTL Temperature | `0x1306` | yes |
| Light HSL | `0x1307` | yes |
| Light xyL | `0x130C` | no |

This is reflected in `src/sphere_mesh/provision.py`: its binding
allowlist is `0x1000`, `0x1300`, `0x1303`, `0x1306`, and `0x1307`.
Step 4 should route each request to the element that owns its server,
not assume every model is on the primary address.

There is one existing constant bug. Both copies of `btmesh.access`
define `MODEL_LIGHT_CTL_TEMPERATURE_SERVER = 0x1304`. `0x1304` is
the Light CTL Setup Server; the Light CTL Temperature Server is
`0x1306`. No HSL or xyL model constants exist.

The absence of `0x130C` means xyL can be implemented as a standard
codec and conditional capability, but it must not be advertised or
sent to this observed Sphere composition. HSL is the standard color
path actually reported by this device.

## Existing access codecs

The installed and vendored `btmesh.access` copies provide the same
lighting API below. Multi-octet opcodes are emitted big-endian;
multi-octet parameters are little-endian. `_expect_params()` checks
both the opcode and exact allowed parameter lengths.

### Light Lightness

| Message | Opcode | Existing symbol |
|---|---:|---|
| Get | `0x824B` | `light_lightness_get()` |
| Set | `0x824C` | `light_lightness_set(..., ack=True)` |
| Set Unacknowledged | `0x824D` | same, `ack=False` |
| Status | `0x824E` | `parse_light_lightness_status()` |

The Set encoder emits `lightness:u16le, tid:u8`; it supports only
the legal minimal form without transition time and delay. It checks
both values' ranges.

`LightLightnessStatus` contains `present_lightness`, optional
`target_lightness`, and optional `remaining_time`. The parser accepts
two parameter bytes for a settled status or five for a transition.

### Light CTL

| Message | Opcode | Existing symbol |
|---|---:|---|
| Get | `0x825D` | `light_ctl_get()` |
| Set | `0x825E` | `light_ctl_set(..., ack=True)` |
| Set Unacknowledged | `0x825F` | same, `ack=False` |
| Status | `0x8260` | `parse_light_ctl_status()` |

The Set encoder emits `lightness:u16le, temperature:u16le,
delta_uv:i16le, tid:u8`. It supports no explicit transition. It
validates lightness and TID, delta UV as signed 16-bit, and
temperature as 800 through 20,000 K.

`LightCtlStatus` contains `present_lightness`,
`present_temperature`, optional `target_lightness`, optional
`target_temperature`, and optional `remaining_time`. The parser
accepts four or nine parameter bytes.

### Light CTL Temperature

| Message | Opcode | Existing symbol |
|---|---:|---|
| Get | `0x8261` | `light_ctl_temperature_get()` |
| Set | `0x8264` | `light_ctl_temperature_set(..., ack=True)` |
| Set Unacknowledged | `0x8265` | same, `ack=False` |
| Status | `0x8266` | `parse_light_ctl_temperature_status()` |

The Set encoder emits `temperature:u16le, delta_uv:i16le, tid:u8`
and applies the same range checks as aggregate CTL Set.

`LightCtlTemperatureStatus` contains `present_temperature`,
`present_delta_uv`, optional `target_temperature`, optional
`target_delta_uv`, and optional `remaining_time`. Delta UV is decoded
as signed 16-bit. The parser accepts four or nine parameter bytes.

The source also has Temperature Range Get/Status at `0x8262` and
`0x8263`. Its status type has `status_code`, `range_min`, and
`range_max`. This is useful for a UI range but is not required for
the core Get/Set/Status operation.

## Missing HSL and xyL codecs

Neither source copy contains any HSL or xyL opcode, model constant,
encoder, status type, parser, export, or controller method.

The minimal Step 4 HSL surface is:

| Message | Opcode | Parameters after opcode |
|---|---:|---|
| Light HSL Get | `0x826D` | none |
| Light HSL Set | `0x8276` | `L:u16le, H:u16le, S:u16le, TID:u8` |
| Set Unacknowledged | `0x8277` | same |
| Light HSL Status | `0x8278` | `L:u16le, H:u16le, S:u16le[, RT:u8]` |

HSL Set may legally append transition time and delay, making its
parameter length seven or nine. Following the existing codecs, the
initial encoder can deliberately emit only the seven-byte form.
Status must accept six or seven parameter bytes. Its result needs
`lightness`, `hue`, `saturation`, and optional `remaining_time`.
All three state values use the full unsigned 16-bit mesh domain.

The minimal xyL surface is:

| Message | Opcode | Parameters after opcode |
|---|---:|---|
| Light xyL Get | `0x8283` | none |
| Light xyL Set | `0x8284` | `L:u16le, x:u16le, y:u16le, TID:u8` |
| Set Unacknowledged | `0x8285` | same |
| Light xyL Status | `0x8286` | `L:u16le, x:u16le, y:u16le[, RT:u8]` |

xyL has the same seven-or-nine-byte Set and six-or-seven-byte Status
shapes as HSL. Its result needs `lightness`, `x`, `y`, and optional
`remaining_time`. Add `MODEL_LIGHT_HSL_SERVER = 0x1307` and
`MODEL_LIGHT_XYL_SERVER = 0x130C` alongside the corrected CTL
Temperature constant.

Dedicated Hue/Saturation, Target, Default, and Range operations are
outside the stated Get/Set/Status requirement. They are not needed to
read back an acknowledged whole-color Set.

## Existing test coverage

`old/vendor/ha-bluetooth-mesh/tests/test_access.py` covers:

- Lightness Set byte layout, acknowledged/unacknowledged selection,
  and settled/transitional Status fields;
- CTL Set layout, signed delta UV, acknowledged/unacknowledged
  selection, and range validation;
- CTL Temperature Set layout, signed delta UV, unacknowledged form,
  and range validation;
- short and full CTL and CTL Temperature Status parsing;
- all three CTL Get byte encodings; and
- CTL Temperature Range success, non-success, and bad length.

`old/vendor/ha-bluetooth-mesh/tests/test_controller.py` adds
request/response coverage for Lightness Get/Set, CTL Get/Set, CTL
Temperature Get/Set, range reads, clamping, group unacknowledged
Lightness, and choosing a target value during a transition.

The root tests do not cover these codecs. `tests/test_access.py`
tests only the local Network Transmit codec, while
`tests/test_control.py` tests only Generic OnOff. The current root
`BasicController` exposes only read/on/off/toggle.

## Tests and integration still needed

For the codec layer, add exact-byte tests for all HSL and xyL Gets,
acknowledged Sets, unacknowledged Sets, and both allowed Status
lengths. Also test every unsigned 16-bit boundary, TID rejection,
wrong opcode, and malformed lengths. Add direct opcode/model-constant
assertions, including the `0x1306` correction.

Existing coverage would benefit from a direct Lightness Get byte
test and malformed-length tests for Lightness, CTL, and CTL
Temperature statuses. The controller tests currently cover the
Lightness Get indirectly.

For the Sphere controller, add capability-to-element resolution from
persisted Composition Data; acknowledged request/Status round trips;
TID allocation before every Set; timeout errors; and returned-state
tests that preserve `present`, optional transition target, and
remaining time. HSL and xyL readback must come from Status rather
than echoing the requested values.

Finally, current composition persistence stores elements and model
IDs, but `control_model` selects only OnOff or Lightness. Step 4 needs
per-model element addresses and explicit unavailable results. On this
Sphere, HSL should resolve and xyL should report unsupported unless a
future composition actually includes `0x130C`.
