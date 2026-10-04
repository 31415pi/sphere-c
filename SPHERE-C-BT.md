# Olight Sphere C Bluetooth protocol

Evidence and implementation reference, 2026-10-04.
Implementation baseline: `c6043fc`.

This document describes the USB-C Sphere C investigated in this
repository. It combines authenticated device observations, the
saved Olight app and product panel, and the current controller.
It is not an Olight specification or a complete Bluetooth Mesh
implementation guide. The magnetic-charge original Sphere is a
separate, untested target. Read [REPORT.md](REPORT.md) for the
investigation's narrative and [README.md](README.md) for setup.

## Evidence and notation

| Label | What it establishes | What it does not establish |
|---|---|---|
| Live | Authenticated traffic from the local Sphere C | Visible output, unless separately observed |
| Visual | A recorded human observation of the lamp | Exact internal state or general behavior across firmware versions |
| Panel | Sphere C mini-app schema or product UI call site | Acceptance or physical effect of every declared DP |
| APK | Executable behavior in Olight 3.2.5 / Thingclips SDK | Which generic SDK branches this product executes |
| Code | Current local source, pinned dependency and offline tests | New physical-device validation |
| Standard | Bluetooth SIG protocol/model definitions | Product support for an optional model or operation |
| Inference | Explanation consistent with available evidence | A confirmed firmware fact |

Hex bytes appear in wire order; `0x` numbers denote numeric
identifiers. `u8`, `u16` and `s16` denote field widths, with signed
`s16`. SIG parameters below use little-endian multi-byte fields.
Tuya VALUE records use big-endian integers. Opcode bytes have
their own encoding: HSL Set is `82 76`, while vendor WRITE is
`c9 d0 07`. Do not serialize an entire message using one byte order.

An **access message** consists of opcode and parameters. Examples
here omit Mesh encryption, source/destination, network sequence,
transport MIC and GATT framing. They are not raw GATT writes.

## Identity and tested scope

| Item | Recorded value | Meaning / source |
|---|---|---|
| Product | Olight Sphere C, USB-C | Tested device generation |
| Composition company ID | `0x07d0` | Tuya namespace used by this Olight product; not a chip ID |
| Composition product ID | `0x0200` | Mesh composition identifier |
| Composition version ID | `0x3530` | Raw composition version; not interpreted as a release string |
| Element count | 6 | One primary and five secondary elements |
| Feature bits | `0x0007` | Relay, Proxy and Friend supported; not a report of enabled state |
| Tuya product ID | `d0dpga0h` | Cloud/panel identifier; distinct from Mesh product ID |
| APK | Olight 3.2.5, version code 239 | Static-analysis artifact |
| Product panel | `0.5.9_RsPYtSG2yNGWYhKJ` | Saved mini-app version |
| Local dependency | `btmesh` revision `a8f7bbbebaeba124d7d08cb16122055074f2ef46` | Pinned in `pyproject.toml` and installed during validation |

Composition was obtained with DeviceKey-authenticated reads.
Addresses are allocations, not model constants. Use primary
address `A` and element offsets below; derive `A` from the owned
network's state. Historical allocations changed after provisioning.
Sources: [composition][models], [provisioning runbook][provision],
[artifact identities][firmware].

### Fresh read-only wake check

On 2026-10-04, the owner reported sleeping and waking the Sphere C.
The daemon was initially absent. Starting it with the existing
state found a Proxy advertisement, established a GATT session
(reported ATT MTU 247), and obtained these authenticated replies.
No access-model Set or vendor Write was issued for this check.
Proxy filter setup and normal sequence persistence still occurred.

| Read | Returned state |
|---|---|
| Generic OnOff | On; no target or remaining-time field |
| Lightness | `1956` |
| HSL | Lightness `4358`, hue `19660`, saturation `65535` |
| CTL Temperature | `18900 K`, delta-UV `0` |
| DP 7 / DP 101 | `0` / `0` seconds |
| DP 102 | `0`, canonical `flame` |
| DP 103 | `100%` |
| DP 104 | `false` |
| DP 2 | Omitted from the response to READ `[2,102]` |

This confirms current reachability and retained local access
after the reported wake. It is not a battery power-loss test.
Flame is the **reported selection**; neither its visible motion
nor current work mode was verified. Lightness and HSL lightness
are separate reported values and must not be silently equated.

## Discovery and GATT

The initial phone capture used standard Mesh Proxy GATT without
an observed SMP pairing or link-encryption exchange. Mesh carries
its own authenticated encryption inside GATT; lack of BLE bonding
does not expose the lighting commands. The capture did not contain
the old network's keys. [Capture analysis][capture]

| Advertisement | Relevant data | Interpretation and limit |
|---|---|---|
| Mesh Provisioning `0x1827` | Device UUID and OOB information | Unprovisioned discovery; UUID is public, not a key |
| Mesh Proxy `0x1828`, type `00` | Eight-byte Network ID | Derived from NetKey by `k3`; identifies a subnet, not uniquely a lamp |
| Mesh Proxy `0x1828`, type `01` | Node Identity hash and random | Can be checked using NetKey and a known node address |
| Olight-associated `0xfcf1` | Opaque service data | Stable payload observed across address rotation; exact device mapping unresolved in the old capture |
| Manufacturer `0x07d0` | Tuya-format data | Vendor ecosystem evidence, not silicon identification |

The current scanner uses fresh advertisement callbacks. It accepts
a matching Network ID or a verified Node Identity for the saved
node, then authenticates Mesh responses. Network ID alone is not
a per-device identity check. [Transport source][transport-code]

### Services and characteristic directions

“In” means client to lamp; “Out” means lamp to client. Short UUIDs
use the Bluetooth base UUID. Handles are observed examples from
one capture, not values to hard-code. [Capture][capture],
[SIG Assigned Numbers][assigned], [Tuya porting guide][tuya-port].

| Service | Data In: write without response | Data Out: notify | Observed use |
|---|---|---|---|
| Provisioning `1827` | `2adb` | `2adc` | Local PB-GATT commissioning |
| Proxy `1828` | `2add` | `2ade` | Authenticated control path |
| Alternate `7fdd` | `2add` | `2ade` | Present in table; not the selected control service |
| Tuya `fd50` | `00000001-0000-1001-8001-00805f9b07d0` | `00000002-0000-1001-8001-00805f9b07d0` | Present; no product control exchange established here |

| Captured service | Out handle | CCCD handle | In handle |
|---|---|---|---|
| `1827` | `0013` | `0015` | `0017` |
| `1828` | `001c` | `001e` | `0020` |
| `7fdd` | `0025` | `0027` | `0029` |
| `fd50` | `0037` | `0039` | `003b` |

Enable notifications through the selected Out characteristic's
CCCD, then use its paired In characteristic. The duplicated Proxy
UUIDs make lookup by characteristic UUID alone ambiguous. Local
transport selects within the intended service and warns if that
service still contains multiple matches. Its UUID-only fallback
when service information is unavailable is an implementation
limitation, not a recommended discovery guarantee.

### Other observed GATT fingerprints

| Role in Telink SDK | Service UUID | Characteristic UUID |
|---|---|---|
| OTA | `00010203-0405-0607-0809-0a0b0c0d1912` | `00010203-0405-0607-0809-0a0b0c0d2b12` |
| User-defined | `00010203-0405-0607-0809-0a0b0c0d7fde` | `00010203-0405-0607-0809-0a0b0c0d7fdf` |

These pairs and `7fdd` match the official Telink SDK examined in
the [GATT fingerprint audit][chip]. Together they strongly support
Telink software lineage. They do not identify TLSR8258 or any
other exact MCU, establish a usable firmware-read command, or
prove that the original Sphere shares the same implementation.

## Layers, keys and state

```text
GUI / command-line client
          | private local JSON socket (not a Bluetooth protocol)
        daemon
          | SIG access message or Tuya vendor message
  upper transport: AppKey or DeviceKey protection
  lower transport: segmentation and reassembly
  network: NetKey protection, addresses, SEQ, IV Index
  Proxy PDU: type and GATT segmentation/reassembly
  GATT Data In / Data Out
          |
       Sphere C
```

The Proxy header is one byte: upper two bits SAR, lower six bits
message type. Its SAR values are `00` complete, `01` first,
`10` continuation, and `11` last. This is distinct from lower
transport segmentation inside Mesh. [Mesh Protocol][mesh-protocol],
[SIG Proxy SAR tests][mesh-tests]

| Proxy message type | Payload |
|---|---|
| `00` | Network PDU |
| `01` | Mesh beacon |
| `02` | Proxy configuration |
| `03` | Provisioning PDU |

| State/key | Role | Local handling and limit |
|---|---|---|
| NetKey, 128 bits | Network encryption/authentication and related identity derivations | Locally generated and private |
| AppKey, 128 bits | Application access protection | Added after provisioning and bound to model instances |
| DeviceKey, 128 bits | Node-specific configuration protection | Derived during provisioning; not sent as plaintext key material |
| SEQ, 24 bits | Network freshness and nonce uniqueness | Advanced cursor persisted before outbound PDU queueing |
| IV Index, 32 bits | Network security context | Fixed session value; no complete IV Update/recovery |
| TID, 8 bits | SIG Set transaction identity | Persisted allocator, modulo 256; not a substitute for SEQ |
| Receive replay state | Reject stale authenticated traffic | Dependency has immediate-duplicate suppression, not durable standards-complete RPL |

Sequence allocation covers application PDUs, lower-transport
segment acknowledgments and Proxy configuration. State updates
use an exclusive lock, atomic replacement and fsync of file and
directory. Sequence gaps are safe; reusing a nonce is not. Never
copy a network state into two concurrent transmitters or roll
back its sequence cursor. [State][state-code],
[transport][transport-code]

The installed pinned `btmesh` acknowledges and reassembles
incoming segmented unicast traffic. Old comments claiming that
segments are never acknowledged are superseded. Reliable resend
of missing outbound segments is still absent. A whole-message
request retry is not equivalent to transport segment recovery.

The current runtime ignores non-network Proxy messages after
connection, including beacons. Network decoding checks IVI and
authenticates under the fixed IV Index; it does not implement a
beacon-driven IV Update state machine. Do not describe this as
complete authenticated-beacon mismatch detection or recovery.
These are client limitations, not observed device deficiencies.

## Provisioning and configuration

Local commissioning uses PB-GATT and standard P-256 ECDH. The
saved app path uses in-band public keys and No OOB; the local
provisioner supports No OOB and optional static OOB. Output/input
OOB and PB-ADV are not implemented. No OOB does not protect
commissioning from an active man-in-the-middle. No firmware
extraction or vendor cloud secret was needed for local ownership.
[Provisioning analysis][provision-analysis], [runbook][provision]

| Stage | Exchange / action | Result and evidence |
|---|---|---|
| Discover | Unprovisioned UUID and capabilities | Identify the intended device and number of elements |
| Negotiate | Invite, Capabilities, Start | Select algorithm/public-key/authentication methods |
| Establish shared secret | Public Key exchange, ECDH | Derive common session material |
| Confirm | Confirmation and Random exchange | Validate the selected authentication transcript |
| Provision | Encrypted Data; Complete response | Transfer NetKey, key index, flags, IV Index and primary unicast |
| Reconnect | Mesh Proxy | Node advertises its new network/identity |
| Inspect | Config Composition Data Get | Authenticate layout and allocate all six addresses |
| Authorize | Config AppKey Add; Model App Bind | Install AppKey and bind the 11 matching instances |
| Finish configuration | Network Transmit Set `80 24 15` | Count field 5, interval steps 2; matching Status recorded |
| Verify | AppKey-authenticated OnOff Get | Confirm usable application access and later reconnect |

The DeviceKey is derived with the provisioning salt and ECDH
secret (`k1(..., "prdk")`). Merely recording a standard public-key
provisioning exchange does not reveal the shared keys.

Early local attempts completed provisioning but later rolled back.
The durable run added the remaining allowlisted bindings and
Network Transmit Set. Both are standard configuration operations;
the experiment did not isolate a single firmware commit trigger.
Retention was observed across disconnect and button-off/wake,
then local ownership was recovered after a round trip through
the phone app. None of these proves survival of battery power loss.

Factory lighting restoration (DP 106) is distinct from Mesh Node
Reset. Reset/reprovisioning changes ownership; the runbook records
the tested recovery procedure rather than making it part of a
normal read or reconnect operation.

## Composition and bindings

All reported element locations were zero. “Bound” below means
included in the successful local AppKey-binding sequence. A
present client or setup model is not another runtime server.
[Composition][models], [secondary-element audit][elements]

| Offset | Model ID | Model | Bound / observed use |
|---|---|---|---|
| `A+0` | `0000` | Configuration Server | DeviceKey configuration |
| `A+0` | `0002` | Health Server | Present, not in bind allowlist |
| `A+0` | `0003` | Health Client | Present, not in bind allowlist |
| `A+0` | `1000` | Generic OnOff Server | Bound; live Get/Set |
| `A+0` | `1001` | Generic OnOff Client | Present, not in bind allowlist |
| `A+0` | `1300` | Light Lightness Server | Bound; live Get/Set |
| `A+0` | `1301` | Light Lightness Setup Server | Present, not in bind allowlist |
| `A+0` | `1302` | Light Lightness Client | Present, not in bind allowlist |
| `A+0` | `1303` | Light CTL Server | Bound; combined CTL not separately live-validated here |
| `A+0` | `1304` | Light CTL Setup Server | Present, not in bind allowlist |
| `A+0` | `1305` | Light CTL Client | Present, not in bind allowlist |
| `A+0` | `1306` | Light CTL Temperature Server | Bound; live Get/Set |
| `A+0` | `1307` | Light HSL Server | Bound; live Get/Set |
| `A+0` | `1308` | Light HSL Setup Server | Present, not in bind allowlist |
| `A+0` | `1309` | Light HSL Client | Present, not in bind allowlist |
| `A+0` | `130a` | Light HSL Hue Server | Present; separate binding/use not established |
| `A+0` | `130b` | Light HSL Saturation Server | Present; separate binding/use not established |
| `A+0` | `07d0:0004` | Tuya vendor server | Bound; DP traffic |
| `A+0` | `07d0:0005` | Tuya vendor client | Not bound or used as a DP server |
| `A+1` | `1000` | Generic OnOff Server | Bound; physical semantics unknown |
| `A+2` | `1000` | Generic OnOff Server | Bound; physical semantics unknown |
| `A+3` | `1000` | Generic OnOff Server | Bound; physical semantics unknown |
| `A+4` | `1000` | Generic OnOff Server | Bound; physical semantics unknown |
| `A+5` | `1000` | Generic OnOff Server | Bound; physical semantics unknown |

There is no Generic Level Server `1002` or xyL family
`130c`–`130e` in this composition. `130a` and `130b` are HSL
Hue/Saturation, not xyL. The five secondary Boolean models have
no demonstrated physical LED, region or color-channel mapping.
Their binding is not proof of independently controlled emitters.

## Standard lighting access

These are opcode values, not little-endian parameter integers.
All rows target the appropriate server at `A+0` on this device.
Unacknowledged variants are documented for interpretation; the
local high-level controller uses acknowledged operations.
[Current access codecs][access-code], [lighting controller][lighting-code],
[SIG model definitions][mesh-model], [assigned opcodes][assigned]

| Model state | Get | Set | Set unacknowledged | Status | Device evidence |
|---|---|---|---|---|---|
| OnOff | `8201` | `8202` | `8203` | `8204` | Live Get/Set |
| Lightness Actual | `824b` | `824c` | `824d` | `824e` | Live Get/Set |
| Combined CTL | `825d` | `825e` | `825f` | `8260` | Present/bound; APK path, not local CLI operation |
| CTL Temperature | `8261` | `8264` | `8265` | `8266` | Live Get/Set |
| HSL | `826d` | `8276` | `8277` | `8278` | Live Get/Set |
| xyL | `8283` | `8284` | `8285` | `8286` | Generic codec only; absent on this Sphere C |

Get has no parameters. Set layouts below are the short forms;
multi-byte parameters are little-endian.

| State | Set parameters | Status parameters, short / extended |
|---|---|---|
| OnOff | `onoff:u8, tid:u8` | `present:u8` / add `target:u8, remaining:u8` |
| Lightness | `lightness:u16, tid:u8` | `present:u16` / add `target:u16, remaining:u8` |
| Combined CTL | `lightness:u16, temperature:u16, delta_uv:s16, tid:u8` | `present_lightness:u16, present_temperature:u16` / add target lightness, target temperature and remaining byte |
| CTL Temperature | `temperature:u16, delta_uv:s16, tid:u8` | `present_temperature:u16, present_delta_uv:s16` / add target temperature, target delta-UV and remaining byte |
| HSL | `lightness:u16, hue:u16, saturation:u16, tid:u8` | Same three color fields / add `remaining:u8`; no target tuple |
| xyL | `lightness:u16, x:u16, y:u16, tid:u8` | Same three fields / add `remaining:u8` |

| Field | Units/range | Interpretation |
|---|---|---|
| OnOff | `0` or `1` | Boolean output state |
| Lightness | `0..65535` | Mesh Lightness Actual; zero has off semantics |
| HSL hue | `0..65535` | Normalized hue, conventionally scaled to 360 degrees |
| HSL saturation | `0..65535` | Normalized HSL saturation |
| CTL temperature | `800..20000 K` protocol range | Not a measured emitter range or promised visible color |
| Delta-UV | `-32768..32767` | Signed fixed-point; approximately raw / 32768 |
| xyL x/y | `0..65535` | Normalized chromaticity; unsupported on this composition |

Applicable Sets may append `transition_time:u8, delay:u8` as a
pair. Transition/remaining time packs a two-bit resolution and
six-bit step count; resolutions are 100 ms, 1 s, 10 s and 10 min.
Step count `0x3f` denotes unknown in status and is not a valid
requested transition. Delay uses 5 ms units. Local HSL encoding
exposes only the short Set form and accepts 6/7-byte Status
parameters. [Model specification][mesh-model], [code][access-code]

Use a new TID for a new logical Set; retransmitting the same
transaction retains its TID. Duplicate detection uses source,
destination and TID within six seconds. The current allocator
persists TIDs but does not throttle rapid modulo-256 reuse; this
matters before designing a high-rate streaming controller.

Range queries documented by the standard/app audit include CTL
Temperature Range Get/Status `8262/8263` and HSL Range Get/Status
`827d/827e`. Their Sphere-specific bounds have not been established
here; they are not exposed by the current lighting CLI.

### State is not always the value just sent

Historical Lightness, CTL Temperature and HSL Sets returned
authenticated Status with the requested values. A later HSL Get
could return a different, stable tuple: restoring
`(3637,1820,65535)` initially acknowledged exactly, then read
`(3276,1638,65535)`. Calibration showed this was not one fixed
scale factor. Firmware color conversion/quantization is a plausible
explanation, not a decoded implementation fact. Independent
Lightness remained `1956`. [Live lighting trials][lighting-live]

CTL Temperature has wire-level support, but later desktop trials
produced no obvious visible temperature change. The GUI omits it.
A reported `18900 K` in the fresh check is model state, not a
measurement of emitted light. [Operational observations][operations]

## Phone color and brightness mapping

The product panel's color wheel is HSV. Its DP 5 string is not
the payload of an HSL Set: the APK converts it first. Ordinary
power, white brightness and color are routed through SIG models,
even though the panel names them as datapoints. [Color audit][color],
[phone mode semantics][phone-mode]

| Representation | Order/range | Conversion / behavior |
|---|---|---|
| Panel HSV | Hue `0..360`, saturation/value `0..1000` | Wheel preserves value; color brightness preserves hue/saturation |
| DP 5 string | Four hex digits each: hue, saturation, value | Twelve hex characters, e.g. `000003e803e8` for full red |
| Mesh HSL | Lightness, hue, saturation, each `u16` | HSV converted to HSL, then scaled |
| DP 3 white brightness | `10..1000` | APK scales to Mesh Lightness; nonzero active values are clamped to at least 1 |
| Desktop HSV | Three `0..65535` components internally | `hsv_to_mesh_hsl` rounds converted fields; QML slider position is logarithmic |

For normalized HSV saturation `s` and value `v`, the local GUI
implements:

```text
l = v * (1 - s / 2)
s_hsl = 0                         when l is 0 or 1
s_hsl = (v - l) / min(l, 1 - l)   otherwise
mesh_l = round(65535 * l)
mesh_s = round(65535 * s_hsl)
```

Hue is unchanged in normalized space. Reverse conversion uses
`v = l + s_hsl * min(l,1-l)` and, for nonzero `v`,
`s = 2 * (1-l/v)`. This documents the current GUI arithmetic;
it does not claim identical rounding in the APK or firmware.
No gamma-correction step was found in the examined panel/APK
path. [GUI source][gui-code]

The phone selects color mode with DP 2, then DP 102. Its Color
Wheel choice is DP 102=`7`, followed by DP 5 updates. Moving the
brightness slider during a color preset writes color data without
reselecting DP 102. The local GUI likewise changes HSV value via
HSL when it knows color/effect mode is active. The effect of that
update on each firmware animation remains incompletely observed.

## Tuya vendor messages

DP messages target vendor server `07d0:0004` on the primary
element. The company field is little-endian inside the three-byte
opcode. The embedded `07d0:0005` client is not an alternate DP
server. [DP inventory][dp-inventory], [DATA codec audit][dp-codec],
[current vendor implementation][vendor-code]

| Operation | Access opcode bytes | Parameter layout |
|---|---|---|
| Acknowledged WRITE | `c9 d0 07` | `command_type:u8` followed by records |
| Unacknowledged WRITE | `ca d0 07` | Same record layout; no DATA required |
| READ | `cc d0 07` | `command_type:u8, count:u8, ids:u8[count]` |
| DATA | `cd d0 07` | `command_type:u8` followed by zero or more records |

DATA and WRITE have **no record count**. This ordinary format has
no vendor transaction ID. Records continue to the end of the
access message. The READ encoder supports 1–255 IDs; the daemon
additionally rejects duplicates. DP ID 0 is an APK all-DP query
convention, not a product-schema DP or a tested local API feature.

### Record fields

Each record starts with `dp_id:u8, dp_type:u8`. The value is:

| Type | Byte | Encoding after type | Local decoding |
|---|---|---|---|
| Raw | `00` | `length:u8, bytes[length]` | Opaque bytes |
| Boolean | `01` | One byte | Zero false, nonzero true; encoder emits 0/1 |
| Value | `02` | Four-byte big-endian integer | Unsigned `0..4294967295` in local codec |
| String | `03` | `length:u8, UTF-8[length]` | Length counts bytes, not characters |
| Enum | `04` | One byte | Index into the product's enum list |
| Bitmap | `05` | `length:u8, bytes[length]` | Big-endian, length restricted to 1, 2 or 4 |

The codec rejects unknown types, truncated headers/values,
missing length, invalid bitmap widths and invalid UTF-8. A
generic encodable record is not automatically a supported product
write. The high-level API restricts mutations to visible preset
selection/activation and DP 101 countdown.

### Command types and response selection

| Command type | Evidence | Current controller policy |
|---|---|---|
| `01` | Ordinary APK and live DP DATA | Read and write response accepted |
| `05` | Alternate ordinary APK DATA/read form | Read response accepted; write response excluded |
| `00` | APK dispatches as generic status | Not parsed as typed DP records |
| `86` | APK group-association command family | Separate grammar; not a DP response |

The meaning of the APK flag selecting READ type `05` is unknown.
Do not assign it a product meaning or assume type echoing without
evidence. The local reader emits type `01`.

The node first matches response source and opcode. The vendor
controller then collects newly received DATA from that element,
waits 100 ms for a response burst, filters command types and
concatenates accepted records. Reads can return unsolicited DPs
and omit requested ones. They do not wait until every requested
ID has arrived. These timings are current client policy, not a
firmware guarantee. [Vendor source][vendor-code]

Writes accept an empty type-`01` acknowledgment or a typed DATA
containing an expected DP. A late echo for an unrelated DP is
ignored while waiting for a usable response. The service checks
reported relevant values against requested values. No request ID
exists to eliminate all ambiguity from delayed empty replies.

| Result | Meaning | What to do with it |
|---|---|---|
| Empty DATA | Expected response received, no value | Record acknowledgment only |
| Typed matching echo | Response reports a value | Distinguish from independent readback |
| Independent READ/Get | A subsequent request reports state | Preserve missing IDs and actual returned values |
| Contradictory typed value | Firmware reports something else | Return protocol error; do not claim success |
| Timeout | No expected reply in the request window | Outcome may be uncertain; inspect state before another mutation |
| Visual observation | Human saw the output | Record separately; API never claims this automatically |

Current vendor requests use a five-second timeout per node
attempt. Reads inherit the dependency's one whole-message retry;
vendor writes explicitly use zero retries there. The controller's
unsupported-response wait uses its own original deadline, so it
is not a general ten-second snapshot collection guarantee.

The daemon has one exception: after DP 2 acknowledges during
`effect.activate`, it waits 250 ms and sends DP 102. If that
selection gets no DATA and raises `VendorUnavailable`, it repeats
only DP 102 once. It never retries the full activation or the
mode write. `effect.select` and countdown writes have no such
retry. A contradictory echo or parsing failure is not retried.
[Daemon][daemon-code], [timeout audit][timeout]

## Product datapoints

The saved Sphere C schema has 17 entries. Access is the schema's
declaration; it does not prove a live read/write or vendor route.
`R/W` means both, `R` read-only and `W` write-only. Generic DP 4
is deliberately excluded from this product table. [Inventory][dp-inventory]

| DP | Name | Schema access/type | Range / format | Route and evidence |
|---|---|---|---|---|
| 1 | `switch_led` | R/W boolean | 0/1 | APK routes ordinary control to SIG OnOff; live SIG access |
| 2 | `work_mode` | R/W enum | Four modes below | Vendor enum; colour write/echo confirmed, read snapshots omit it |
| 3 | `bright_value` | R/W value | `10..1000`, step 1 | APK maps white brightness to SIG Lightness |
| 5 | `colour_data` | R/W string | HSV, 12 hex digits; schema max 255 | APK converts to SIG HSL; not an observed vendor string |
| 6 | `scene_data` | R/W string | Schema max 255 | Generic APK compresses to vendor raw; no panel preset call or live validation |
| 7 | `countdown` | R/W value | `0..43200` seconds | Live vendor VALUE read zero; no validated write |
| 8 | `music_data` | W string | Schema max 255; compact raw stream | Panel microphone path and APK encoding; local control deferred |
| 33 | `power_memory` | R/W raw | Max 255 bytes | Schema only; opaque power-restoration data |
| 37 | `rtc_timer` | R/W raw | Max 128 bytes | Schema only; opaque timer configuration |
| 38 | `timer_report` | R/W raw | Max 128 bytes | Schema only; opaque timer report |
| 101 | `countdown_2` | R/W value | `0..43200` seconds | Panel countdown; live VALUE read and zero write/echo/readback |
| 102 | `colour_mode` | R/W enum | `0..31` | Built-in preset selector; live vendor enum reads/writes |
| 103 | `battery_percentage` | R value | `0..100%`, step 1 | Live vendor VALUE; opportunistically displayed |
| 104 | `sleep_switch` | R/W boolean | 0/1 | Live vendor BOOL read false; write/timing semantics unknown |
| 105 | `game_flag` | R/W boolean | 0/1 | Schema game label; no verified wire action or behavior |
| 106 | `factory_mode_reset` | W boolean | Panel writes true | Restore Factory Colors call; effects/reset scope not live established |
| 107 | `water_mode` | R/W boolean | 0/1 | Schema label; no verified wire action or behavior |

DP 4 (`temp_value`) is generic APK functionality mapped to CTL
Temperature. The Sphere C panel has neither this schema entry nor
a temperature slider. Its absence from the panel does not negate
the authenticated CTL model responses.

Focused reads for DP 2, DP 102 or both historically returned an
empty acknowledgment and a broader five-record snapshot: DPs 7,
101, 102, 103 and 104. The fresh check returned the same ID set.
This suggests a general status refresh rather than strict request
filtering. DP 2's omission does not establish write-only access.

### Work mode: DP 2

| Decimal / hex | Schema name | Evidence boundary |
|---|---|---|
| 0 / `00` | `white` | Panel white-light selection; not independently characterized here |
| 1 / `01` | `colour` | Live typed write echo and successful preset activation |
| 2 / `02` | `scene` | Schema/generic SDK; untested on the Sphere C |
| 3 / `03` | `music` | Panel microphone-stream path; local implementation deferred |

### Presets: DP 102

Canonical names and enum order come from the product schema.
Labels/slugs below match current `effect_catalog()`. “Grid” means
one of 25 desktop choices; “Wheel” is the separate color control;
“Hidden” means absent from the saved picker and blocked by the
guarded write API. Hidden firmware support is unknown.

Motion `?` means unclassified, not static. All names are Panel/Code
evidence; only the motion entries explicitly marked observed have
Visual evidence. Wire selection was tested for a subset, not every
catalog member. [Motion audit][motion], [catalog source][vendor-code]

| Dec | Hex | Canonical | Label | Slug | UI | Motion |
|---|---|---|---|---|---|---|
| 0 | `00` | `flame` | Flame | `flame` | Grid | ? |
| 1 | `01` | `colour_1` | Cotton Candy | `cotton-candy` | Grid | ? |
| 2 | `02` | `colour_2` | Aurora | `aurora` | Grid | Observed animation |
| 3 | `03` | `colour_3` | Fireworks | `fireworks` | Grid | ? |
| 4 | `04` | `colour_4` | Fountain | `fountain` | Grid | ? |
| 5 | `05` | `colour_5` | Firefly | — | Hidden | ? |
| 6 | `06` | `colour_6` | Spiral | `spiral` | Grid | ? |
| 7 | `07` | `colour_7` | Color Wheel | `color-wheel` | Wheel | Solid-color interaction |
| 8 | `08` | `colour_8` | Red Flash | `red-flash` | Grid | Observed animation |
| 9 | `09` | `colour_9` | Confetti | `confetti` | Grid | ? |
| 10 | `0a` | `colour_10` | Butterfly | `butterfly` | Grid | ? |
| 11 | `0b` | `colour_11` | Fireball | `fireball` | Grid | ? |
| 12 | `0c` | `colour_12` | Gradient | `gradient` | Grid | ? |
| 13 | `0d` | `colour_13` | Carousel | `carousel` | Grid | ? |
| 14 | `0e` | `colour_14` | Floating | — | Hidden | ? |
| 15 | `0f` | `colour_15` | Magic Ball | `magic-ball` | Grid | ? |
| 16 | `10` | `colour_16` | Lilac | `lilac` | Grid | ? |
| 17 | `11` | `colour_17` | Sunset | `sunset` | Grid | ? |
| 18 | `12` | `colour_18` | Vibrant | `vibrant` | Grid | ? |
| 19 | `13` | `colour_19` | Autumn | — | Hidden | ? |
| 20 | `14` | `colour_20` | Red Light | `red-light` | Grid | ? |
| 21 | `15` | `colour_21` | Blue sky | `blue-sky` | Grid | ? |
| 22 | `16` | `colour_22` | New Year | — | Hidden | ? |
| 23 | `17` | `colour_23` | Christmas | — | Hidden | ? |
| 24 | `18` | `colour_24` | Harvest | `harvest` | Grid | ? |
| 25 | `19` | `colour_25` | Willow Wisp | `willow-wisp` | Grid | ? |
| 26 | `1a` | `colour_26` | Zombie | `zombie` | Grid | ? |
| 27 | `1b` | `colour_27` | Snowfall | — | Hidden | ? |
| 28 | `1c` | `colour_28` | Phantom Purple | `phantom-purple` | Grid | ? |
| 29 | `1d` | `colour_29` | Christmas | `christmas` | Grid | ? |
| 30 | `1e` | `colour_30` | Candy Cane | `candy-cane` | Grid | ? |
| 31 | `1f` | `colour_31` | Snowfall | `snowfall` | Grid | ? |

The two Christmas and two Snowfall labels are distinct enum
entries. Use canonical values or slugs, not translated labels as
identifiers. Other panel translations include Fireplace, Small
Rocket, Cheerleaders and Magic Blue Flame; none proves motion.
There are 32 schema entries, 26 picker choices including the
wheel, and 25 desktop grid entries.

## Operations and examples

| Intent | Current sequence | Important distinction |
|---|---|---|
| Read power | SIG OnOff Get | No output change |
| Select a preset | DP 102 WRITE | Does not write DP 2; selection alone may not activate it |
| Activate a preset | DP 2=`colour`; wait; DP 102 WRITE | Two separate acknowledgments; partial completion possible |
| Activate solid color | Activate Color Wheel; HSL Set | Current GUI path; subsequent wheel edits update HSL |
| Phone picker entry | Power on if needed; DP 2; DP 102 | Phone's power step is not automatically part of daemon activation |
| Read countdown | READ DP 101 | Preserve other snapshot records and missing-ID information |
| Set/cancel countdown | DP 101 VALUE; zero cancels | Positive delayed-off behavior not live-validated |

The daemon caches acknowledged/reported mode only for its current
connection generation. After restart/reconnect, missing DP 2 can
leave `effect_active` unknown even when DP 102 is readable. The
proposed power-plus-DP-102 startup inference is not implemented.
It would be a UI policy based on a single-controller assumption,
not a discovered wire rule. [Service guide][service]

### Synthetic access-message vectors

These vectors were checked against the local codecs. They are
illustrations, not newly captured device traffic or permission
to transmit a write. A real sender supplies the correct keys,
binding, address, fresh sequence and appropriate TID.

| Intent | Complete access-message bytes |
|---|---|
| OnOff Get | `82 01` |
| HSL Set: L=32768, H=0, S=65535, TID=42 | `82 76 00 80 00 00 ff ff 2a` |
| HSL Status for that tuple | `82 78 00 80 00 00 ff ff` |
| READ DPs 2 and 102 | `cc d0 07 01 02 02 66` |
| Empty DATA acknowledgment | `cd d0 07 01` |
| WRITE colour mode | `c9 d0 07 01 02 04 01` |
| DATA colour-mode echo | `cd d0 07 01 02 04 01` |
| WRITE Aurora selection | `c9 d0 07 01 66 04 02` |
| DATA Aurora echo | `cd d0 07 01 66 04 02` |
| WRITE countdown zero | `c9 d0 07 01 65 02 00 00 00 00` |
| DATA countdown zero | `cd d0 07 01 65 02 00 00 00 00` |

In the READ, the first `02` after command type is the count and
the second is the DP ID. In the mode WRITE, `02` is the DP ID,
`04` the enum type and `01` its value. There is no WRITE count.

A synthetic snapshot matching the fresh decoded values is one
access message, displayed across lines for readability:

```text
cd d0 07 01
07 02 00 00 00 00
65 02 00 00 00 00
66 04 00
67 02 00 00 00 64
68 01 00
```

The five rows encode countdown, panel countdown, Flame,
battery 100 and sleep false. The 28-byte access message requires
Mesh lower-transport segmentation; it is not five independent
notifications. GATT Proxy SAR is a separate layer.

A combined mode/selection WRITE can be encoded as
`c9 d0 07 01 02 04 01 66 04 02`. The panel's group path publishes
both DPs together, but current individual-device activation uses
two sequential writes. Do not substitute this combined vector
for evidence that both changes completed on the physical device.

## Less-understood surfaces

| Surface | What is known | Remaining boundary |
|---|---|---|
| DP 6 scenes | Generic APK normal form has a one-byte header and 13-byte units; compact form has a two-byte header and four bytes per unit | Packed semantics/product use unresolved; not the preset picker |
| DP 8 music | Panel streams microphone-derived updates after mode=`music`; APK emits raw record `08 00 04` plus four packed bytes | No local live validation or production streaming API |
| Music source format | One mode hex digit, then four hex digits each for hue, saturation, value, brightness and temperature | Compaction retains mode/H/S/V; exact bitfield reference is not established here |
| Group association | APK-derived WRITE parameters `86 02 <group-address>`, reply `86 01 00` | Non-DP grammar; Sphere execution and address encoding not independently validated here |
| Opaque timer/power DPs | Schema names, types and size limits | No payload decoder or safe semantic write contract |
| Secondary OnOff elements | Composition and successful AppKey bindings | No recorded secondary runtime sweep or physical mapping |
| Host animation | Standard HSL could carry a series of whole-lamp colors | No per-pixel interface, guaranteed rate, backpressure or long-run IV support |
| USB-C | Charging with no enumeration in tested states | Unknown factory/debug paths remain possible |

Sources: [DP inventory][dp-inventory],
[device-animation audit][animation-audit],
[secondary elements][elements], [host animation][host-animation],
[USB investigation][usb]. None establishes an individual-LED
index, pixel buffer or spatial mapping command.

## Firmware and OTA boundary

No device image was found in the APK or copied phone files
examined in the [firmware artifact audit][firmware]. The APK
contains Tuya OTA clients and authenticated metadata lookups;
that is an acquisition map, not recovered firmware.

| Artifact / field | Saved evidence | Interpretation |
|---|---|---|
| `m.thing.firmware.upgrade.info.get` | Authenticated selectors, versions 2.0 and 1.1 | Returns device-specific metadata; not a literal unauthenticated URL |
| Update metadata | URL, MD5, size, version, type; signature/HMAC/differential fields in SDK objects | Fields may vary by actual path/package |
| `BleOtaParam.firmwareData` | Byte array passed to OTA implementation | Candidate capture point before Bluetooth framing |
| Start reply | Allow-update, OTA version, type, limit, signature algorithm/key version | Generic SDK structure, not a captured Sphere exchange |
| File reply | State, length, CRC32, MD5 progress | Transfer metadata, not an Olight image header |
| Direct BLE and SIG Mesh OTA | Both implementations present | Actual original-Sphere branch must be observed |
| Approximately 140 KB offered update | Owner's earlier original-Sphere experience | Historical opportunity for a boxed original, not a Sphere C image |

A later image could clarify handlers, effects and image structure,
but may be signed, compressed, encrypted or differential. An
original-Sphere image would not establish Sphere C internals.
The [prepared acquisition procedure][ota-plan] captures downloaded
bytes inside the official app and preserves metadata/hashes.
The exact-device checklist and private manifest are there; no
update was accepted or phone instrumentation performed for this
documentation pass.

## Evidence ledger and implementation limits

| Claim | Date / evidence | Source | Remaining uncertainty |
|---|---|---|---|
| Standard Mesh over Proxy | 2026-09-26 capture | [Capture][capture] | Original app-network access payloads remain encrypted |
| Local ownership persists | 2026-09-26/27 Live | [Runbook][provision] | True battery-off retention and isolated commit trigger |
| Primary lighting Get/Set works | Recorded Live trials | [Lighting][lighting-live] | CTL visible effect; HSL settled quantization |
| DP selection/activation works | Recorded Live and Visual | [Effect trial][effect-trial] | Not every catalog value visually classified |
| Aurora and Red Flash animate | Recorded Visual evidence | [Motion][motion] | 23 other grid entries unclassified |
| Countdown zero works | Recorded Live echo/readback | [Remaining DPs][remaining-dps] | Positive countdown and delayed-off behavior |
| Woken unit is reachable | 2026-10-04 Live read-only | Fresh-check table above | No hard power-cycle or appearance inference |
| Offline suite passes | 2026-10-04 Code: 235 tests, 12.91 s | `tests/`, `pytest -q` | Tests use synthetic/fake transports; not firmware conformance |
| Telink stack lineage | Saved source/GATT comparison | [Fingerprint audit][chip] | Exact silicon; applicability to original Sphere |

| Area | Present in current implementation | Still absent or limited |
|---|---|---|
| State ownership | Exclusive lock; durable sequence/TID allocation | No safe sharing through copied state files |
| Identity/reconnect | Fresh scans, Network ID and Node Identity checks, supervised reconnect | Network ID is subnet-wide |
| Incoming segmentation | Reassembly, segment acknowledgments, duplicate-segment handling | Not a complete durable receive replay defense |
| Outgoing segmentation | Encoding and sending | Reliable missing-segment retransmission |
| IV/replay | Fixed IV authentication; immediate-duplicate suppression | IV Update/recovery and durable RPL |
| Transactions | Serialized API requests; source/opcode matching; vendor filtering | No vendor transaction ID; late-reply ambiguity |
| Presets | Named visible catalog and guarded activation | Motion previews, active-mode recovery after restart |
| Desktop | Power, brightness, HSV color, preset grid, battery | Human accessibility/HiDPI acceptance and user-menu installation remain separate |
| Multiple lamps | Architecture proposal | Shared network/device targeting not implemented |

The current public evidence supports implementing the documented
single-lamp controls. It does not support speculative factory
writes, copying state to another transmitter, or assuming the
entire schema has a working product feature behind it.

## Source map

Repository links below point to reproducible source or public
research summaries. Some original captures and app artifacts are
private/ignored; those summaries preserve provenance without
publishing keys. Historical notes retain their original context;
this reference reconciles them with the stated code baseline.

SIG Mesh Protocol 1.1, Mesh Model 1.1 and Assigned Numbers were
consulted on 2026-10-04. Direct Protocol HTML access was
intermittent; the official test suite corroborated Proxy SAR.
The Telink comparison is the saved audit at its cited SDK commit,
not a newly measured chip identification. Tuya's porting page was
rechecked on 2026-10-04 for the `fd50` characteristic interface.

[capture]: docs/analysis/capture.md
[provision]: docs/guides/self-provisioning.md
[provision-analysis]: docs/analysis/provisioning.md
[models]: docs/research/mesh/step4_sphere_model_map.md
[lighting-live]: docs/analysis/step4-live-validation.md
[operations]: docs/analysis/step6-operational-validation.md
[dp-inventory]: docs/research/mesh/vendor_dp_inventory.md
[dp-codec]: docs/research/mesh/vendor_dp_response_codec.md
[color]: docs/research/app/sphere_color_mapping_audit.md
[phone-mode]: docs/research/app/phone_effect_mode_semantics.md
[motion]: docs/research/app/effect_motion_classification.md
[timeout]: docs/research/mesh/effect_timeout_runtime_audit.md
[effect-trial]: docs/research/mesh/effect_write_experiment_plan.md
[remaining-dps]: docs/research/mesh/remaining_dp_audit.md
[animation-audit]: docs/research/mesh/host_device_animation_audit.md
[elements]: docs/research/mesh/individual_led_audit.md
[host-animation]: docs/research/mesh/host_animation_feasibility.md
[chip]: docs/research/hardware/original_sphere_chip_inference.md
[usb]: docs/research/hardware/usb_port.md
[firmware]: docs/research/hardware/original_sphere_firmware_artifacts.md
[ota-plan]: docs/research/hardware/original_sphere_ota_capture_plan.md
[service]: docs/guides/local-service.md
[access-code]: src/sphere_mesh/access.py
[lighting-code]: src/sphere_mesh/lighting.py
[vendor-code]: src/sphere_mesh/vendor.py
[daemon-code]: src/sphere_mesh/daemon.py
[transport-code]: src/sphere_mesh/transport.py
[state-code]: src/sphere_mesh/state.py
[gui-code]: src/sphere_mesh/gui/backend.py
[mesh-protocol]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html
[mesh-model]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MMDL_v1.1/out/en/index-en.html
[assigned]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/index-en.html
[mesh-tests]: https://files.bluetooth.com/wp-content/uploads/dlm_uploads/2025/02/MESH.TS_.p15.pdf
[tuya-port]: https://developer.tuya.com/en/docs/iot-device-dev/Porting-Guide-BLE?id=Kam0xjtz4n6e0
