# Step 4 Sphere C standard-model map

## Result

The captured Sphere C has three of the four candidate standard
lighting surfaces on its primary element:

- Light Lightness Server `0x1300`;
- Light CTL Server `0x1303` and Light CTL Temperature Server
  `0x1306`;
- Light HSL Server `0x1307`, Hue Server `0x130a`, and Saturation
  Server `0x130b`.

Lightness, CTL, and HSL were selected and bound by both the Olight
3.2.5 APK and the local provisioning flow. xyL is **not present**. Its
Server ID is `0x130c`, followed by Setup Server `0x130d` and Client
`0x130e`; none appears in the authenticated composition.

The supported servers are on element zero, so their destination is the
node's primary unicast address. The currently active local allocation
is `0x0118`, but code must derive this from state rather than hard-code
it. Reprovisioning has already moved the same physical Sphere through
several unicast ranges.

## Evidence and confidence

The evidence used here, in descending order of authority, is:

1. `state/sphere_state.json` and three matching composition dumps in
   `state/provision_run.log`. These are authenticated Composition Data
   page 0 results from the actual Sphere C.
2. The Olight 3.2.5 baksmali under
   `/tmp/olight-dex.7xC3jV/`. Its Thingclips SIG Mesh parser contains
   the exact Lightness, CTL, and HSL actions and conversions.
3. `old/captures/sphere-c.btsnoop` and
   `old/analysis/proxy_pdus.csv`. The old Olight-network keys are not
   available, so these give structural packet evidence, not decoded
   opcodes.
4. The local provisioning code and successful self-provision captures.
   These prove binding and acknowledged Generic OnOff access, but no
   live local Lightness, CTL, HSL, or xyL transaction has yet been
   recorded.
5. The Bluetooth Mesh Model specification and Bluetooth Assigned
   Numbers for normative field meanings and opcodes.

The composition is device-specific evidence. Numeric meanings and
message layouts below are standard-model semantics. Actual emitter
gamut and the Sphere's configured CTL/HSL ranges still require
read-only range queries or controlled live tests.

## Authoritative composition

The node reports CID `0x07d0`, PID `0x0200`, VID `0x3530`, six
elements, and feature bits `0x0007` (Relay, Proxy, and Friend).

Element zero contains these SIG models:

| ID | Model |
|---|---|
| `0x0000` | Configuration Server |
| `0x0002` | Health Server |
| `0x0003` | Health Client |
| `0x1000` | Generic OnOff Server |
| `0x1001` | Generic OnOff Client |
| `0x1300` | Light Lightness Server |
| `0x1301` | Light Lightness Setup Server |
| `0x1302` | Light Lightness Client |
| `0x1303` | Light CTL Server |
| `0x1304` | Light CTL Setup Server |
| `0x1305` | Light CTL Client |
| `0x1306` | Light CTL Temperature Server |
| `0x1307` | Light HSL Server |
| `0x1308` | Light HSL Setup Server |
| `0x1309` | Light HSL Client |
| `0x130a` | Light HSL Hue Server |
| `0x130b` | Light HSL Saturation Server |

It also contains vendor models `0x07d0:0x0004` and
`0x07d0:0x0005`. Elements one through five contain only Generic OnOff
Server `0x1000`.

There is no Generic Level Server `0x1002` in the captured composition.
The sentence in `docs/guides/self-provisioning.md` that says the
primary element has Generic Level is therefore inaccurate. The raw
composition and repeated log dumps agree and should win.

There is likewise no Light xyL Server `0x130c`, Setup Server
`0x130d`, or Client `0x130e`. Existing prose in `README.md` and
`docs/guides/self-provisioning.md` that says the composition includes
xyL is based on a model-ID misidentification. IDs `0x130a` and
`0x130b` complete the HSL family.

The five secondary OnOff servers may represent separately addressable
internal channels, but no local artifact establishes their physical
mapping. Step 4 lighting commands should target element zero only.

## Binding state

The Olight APK scans every element but binds only this fixed SIG
allow-list:

```text
0x1000, 0x1300, 0x1303, 0x1306, 0x1307
```

It also binds vendor server `0x07d0:0x0004`. The local provisioner
mirrors this list. On this composition that produced six primary
bindings plus Generic OnOff on the five secondary elements.

Consequences for Step 4:

- Lightness `0x1300`, CTL `0x1303`, CTL Temperature `0x1306`, and
  HSL `0x1307` are expected to accept the current AppKey.
- Setup servers are not direct runtime-control targets.
- xyL cannot be enabled by adding a binding because its server model is
  absent from composition.
- The client models describe client capability in the firmware; they
  are not destinations for controller commands.

## Lightness / brightness

Use the Light Lightness Server on the primary element.

| Operation | Request | Acknowledged response |
|---|---:|---:|
| Read | Lightness Get `0x824b` | Status `0x824e` |
| Set | Lightness Set `0x824c` | Status `0x824e` |

The no-transition Set parameters are little-endian `lightness:u16,
tid:u8`. Status has `present_lightness:u16`; while transitioning it may
also have `target_lightness:u16, remaining_time:u8`.

This is Light Lightness **Actual**, not Light Lightness Linear. Its raw
range is `0x0000..0xffff`. Zero means no emitted light and is bound to
Generic OnOff off. Nonzero values represent perceptually uniform
lightness, with `0xffff` the maximum. A normalized Step 4 API can map
`0.0..1.0` to `0..65535` with rounding.

The APK maps its `bright_value` schema linearly to this raw range, but
clamps an outgoing nonzero brightness to at least `0x0001`. It controls
power separately with Generic OnOff. A Step 4 method whose contract is
"change brightness without changing power" should follow that policy;
a method that explicitly allows brightness zero to turn the lamp off
may send `0x0000`.

The encrypted app capture has one isolated `20 -> 22` Network-PDU pair
consistent with Lightness Get and short Status. This is supportive,
not authenticated opcode evidence.

## CTL temperature and delta-UV

Two related primary-element servers are available.

### Temperature-only path

This is the closest mapping for a Step 4 color-temperature control
because it does not also overwrite brightness.

| Operation | Request | Acknowledged response |
|---|---:|---:|
| Read | CTL Temperature Get `0x8261` | Status `0x8266` |
| Set | CTL Temperature Set `0x8264` | Status `0x8266` |
| Read range | Range Get `0x8262` | Range Status `0x8263` |

The no-transition Set parameters are little-endian
`temperature:u16, delta_uv:i16, tid:u8`. Status contains present
temperature and signed delta-UV. During transition it may append target
temperature, target delta-UV, and remaining time.

Temperature is expressed directly in kelvin. The model-wide valid
range is `0x0320..0x4e20`, or 800 K through 20000 K. That is a protocol
range, not a claim about the Sphere's emitters. Step 4 should issue the
read-only Range Get and use a successful returned min/max as the actual
device range. No saved local artifact contains that response yet.

Delta-UV is a signed 16-bit fixed-point value on approximately
`[-1, +1)`, with physical value `raw / 32768` and zero on the black-body
locus. The APK always sends raw delta-UV zero. Keeping zero as the
ordinary UI default matches vendor behavior; an expert/raw control may
expose the full `-32768..32767` representation.

### Combined CTL path

| Operation | Request | Acknowledged response |
|---|---:|---:|
| Read | CTL Get `0x825d` | CTL Status `0x8260` |
| Set | CTL Set `0x825e` | CTL Status `0x8260` |

Combined Set carries `lightness:u16, temperature:u16,
delta_uv:i16, tid:u8`. CTL Status reports present lightness and
temperature, plus optional target lightness, target temperature, and
remaining time. It does **not** report delta-UV. Use Temperature Get
when delta-UV readback matters.

The APK defines combined CTL Get `0x825d -> 0x8260`, but its
`temp_value` setter uses temperature-only Set `0x8264 -> 0x8266` and
delta-UV zero. It maps the Tuya schema interval linearly across the full
800..20000 K standard interval rather than discovering the configured
range.

The encrypted app capture's `20 -> 24` pair could structurally be
either combined CTL Get/Status or Temperature Get/Status: both have a
two-byte request opcode and four mandatory response bytes. The APK's
implemented Get action makes combined CTL Get the stronger candidate.

## HSL

Use the Light HSL Server `0x1307` on the primary element.

| Operation | Request | Acknowledged response |
|---|---:|---:|
| Read | HSL Get `0x826d` | HSL Status `0x8278` |
| Set | HSL Set `0x8276` | HSL Status `0x8278` |

The no-transition Set parameters are little-endian
`lightness:u16, hue:u16, saturation:u16, tid:u8`. Status contains the
three present values and may append one remaining-time byte. Target
values are available through separate Target Get/Status messages, not
in ordinary HSL Status.

Each component uses `0x0000..0xffff`:

- lightness is the same normalized/perceptual lightness state discussed
  above;
- hue normalized is `raw / 65535`, conventionally displayed as that
  fraction times 360 degrees; and
- saturation normalized is `raw / 65535`.

The APK's `colour_data` is HSV text, not an HSL wire payload. It parses
12 hexadecimal digits as Tuya hue `0..360`, saturation `0..1000`, and
value `0..1000`, converts HSV to HSL, then scales the HSL lightness, hue,
and saturation to 16-bit fields. Step 4 should accept true HSL or make
an HSV-to-HSL conversion explicit; treating HSV value as the HSL
lightness field would not match app behavior.

The APK implements acknowledged HSL Set `0x8276 -> 0x8278`, its
unacknowledged counterpart `0x8277`, and HSL Get
`0x826d -> 0x8278`. It also parses returned HSL into Tuya HSV.

Although the composition also has Hue Server `0x130a` and Saturation
Server `0x130b`, the APK/local allow-list binds only the composite HSL
Server `0x1307`. Step 4 should start with composite HSL Get/Set. It
must not assume the separate Hue and Saturation server models accept
the current AppKey without checking their bindings.

The old encrypted capture has two `20 -> 26` and five `27 -> 26`
transactions. Those lengths fit HSL Get/Status and no-transition HSL
Set/Status. xyL has the same wire lengths, so packet shape alone would
not distinguish them, but composition lacks xyL and the APK implements
HSL actions. The evidence therefore identifies these as HSL with high
confidence.

## xyL

The actual Sphere C cannot be addressed through standard xyL. The
required runtime server is Light xyL Server `0x130c`, and it is absent
from every authenticated composition dump.

| Operation | Request | Acknowledged response |
|---|---:|---:|
| Read | xyL Get `0x8283` | xyL Status `0x8286` |
| Set | xyL Set `0x8284` | xyL Status `0x8286` |
| Read range | Range Get `0x828b` | Range Status `0x828c` |

The table records the standard messages so a codec can remain generic;
it does not make them valid Sphere C operations. A no-transition Set
would carry little-endian `lightness:u16, x:u16, y:u16, tid:u8`, and
Status would contain present lightness, x, and y, with optional
remaining time.

All fields use `0x0000..0xffff`. Lightness has the same meaning as
above. CIE 1931 coordinates are `x = raw_x / 65535` and
`y = raw_y / 65535`. The protocol range does not guarantee that every
coordinate is in the Sphere's physical LED gamut. A successful xyL
Range Status supplies configured per-axis bounds, but a real gamut is
not necessarily a rectangle; callers should still expect clipping or
conversion differences at unreachable colors.

There is also no APK xyL action, no xyL bind in the APK or local
provisioner, and no decoded or uniquely identifiable xyL capture. Step
4 should report xyL as **unsupported by this composition**. It should
not send xyL opcodes to the HSL Hue Server at `0x130a`.

If an API accepts xyL for portability, it needs either an explicit
unsupported-model result or a documented xy-to-HSL conversion followed
by HSL Set. Such a conversion is a client feature, not native xyL
support, and will be limited by the Sphere's unknown physical gamut.

## Acknowledgment and state handling

For unicast Step 4 operations, use acknowledged Set and wait for the
listed Status from the primary element. Parse status as state, not just
as an acknowledgment:

- Lightness and CTL statuses can contain present and target values.
  During a transition, the target is the effective requested state.
- HSL ordinary status contains present values plus optional remaining
  time; use HSL Target Status if target values are required
  independently.
- A Status response proves receipt and current model state, not that a
  requested out-of-gamut color was reproduced physically.

Every Set above carries an 8-bit TID. A message with the same source,
destination, and TID within six seconds is the same transaction, not a
new command. Allocate a TID before sending, retain it for retransmission
of that one logical transaction, and use a new value for a new command.
This matches the project's current OnOff transaction policy.

No-transition forms are the shortest safe first implementation. The
optional transition form appends `transition_time:u8, delay:u8`; both
bytes must appear together. Status parsers should accept both settled
and transitioning lengths.

Group control should use the corresponding Set Unacknowledged opcode to
avoid one response per group member. Step 4's first implementation is
unicast and should stay acknowledged for deterministic readback.

## Device-specific implementation cautions

1. Derive the primary address and models from saved composition. Do not
   encode `0x0118` or assume every Sphere generation is identical.
2. Do not treat `0x130a` as xyL. It is the Light HSL Hue Server. xyL
   Server `0x130c` is absent and must be reported unsupported.
3. Bind and address the runtime Server IDs, not Setup or Client IDs.
4. Keep power separate from active brightness as the APK does. Raw
   lightness zero has off semantics; active brightness starts at one.
5. Prefer temperature-only CTL Set for a temperature slider. Combined
   CTL Set also overwrites brightness.
6. Preserve signed delta-UV end to end. Parsing it as unsigned changes
   negative tint offsets into large positive values.
7. A future xy-to-HSL adapter must be labelled as conversion rather
   than native xyL control.
8. The old capture cannot authenticate opcodes because its mesh keys
   are unavailable. Do not promote packet-length inference to a live
   device test result.
9. The local capture has only proven acknowledged Generic OnOff on the
   current network. Each new model path needs one conservative Get,
   then an acknowledged Set, then a Get/readback test.
10. Range Get is read-only and should precede CTL and HSL range
    assumptions. HSL Range Get is `0x827d -> 0x827e`.

## Suggested Step 4 rollout order

1. Lightness Get, followed by a small acknowledged Lightness Set and
   readback.
2. CTL Temperature Range Get, CTL Temperature Get, then a conservative
   Set with delta-UV zero.
3. HSL Get, then an acknowledged Set whose HSL values are known and
   reversible.
4. Return an unsupported-model result for native xyL. Add an optional
   xy-to-HSL conversion only if the Step 4 API requires it.

This order follows the evidence strength and avoids sending xyL
messages to a composition that does not expose an xyL Server.

## References

Local evidence:

- `state/sphere_state.json`
- `state/provision_run.log`
- `src/sphere_mesh/provision.py`
- `src/sphere_mesh/capture.py`
- `scripts/infer_mesh_models.py`
- `docs/analysis/compatibility.md`
- `docs/research/app/apk_post_provision_flow.md`
- `docs/research/mesh/post_provision_commit_diff.md`
- `/tmp/olight-dex.7xC3jV/smali6/com/thingclips/sdk/`
  `sigmesh/parse/bdpdqbp.smali`
- `/tmp/olight-dex.7xC3jV/smali6/com/thingclips/sdk/`
  `bluetooth/`

Normative references:

- [Bluetooth Mesh Model specification 1.1][mesh-model]
- [Bluetooth Assigned Numbers: Mesh model opcodes][assigned]

[mesh-model]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MMDL_v1.1/out/en/index-en.html
[assigned]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/index-en.html
