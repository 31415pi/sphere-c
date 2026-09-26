# Original Sphere and Sphere C compatibility

## Result

The evidence supports treating Sphere C as the same Olight/Tuya
Bluetooth Mesh product family as the original Sphere. It does not
yet prove that the two devices have identical composition data,
vendor features, or firmware.

Work on the original Sphere should be reused in layers:

1. Reuse Bluetooth Mesh provisioning, proxy transport, key handling,
   sequence handling, and encryption unchanged.
2. Reuse the Olight/Tuya app key-extraction approach.
3. Start with the same SIG lighting models and Tuya vendor model.
4. Treat product IDs, composition, scene numbers, and optional DPs as
   per-model data that must be verified.

## Evidence for a shared platform

The Sphere C capture proves that it uses:

- Standard Mesh Provisioning service `0x1827`
- Standard Mesh Proxy service `0x1828`
- Tuya company ID `0x07d0`
- Tuya service `0xfd50`
- Tuya vendor-model-sized status traffic
- Olight discovery service `0xfcf1`
- The Olight Android app as the mesh provisioner

The official manuals show that both generations have essentially the
same app-visible behavior: power, brightness, white and colored light,
replaceable static/dynamic modes, music rhythm, group control,
automations, gateway support, and a 30-minute sleep mode.

The main user-visible revision is charging hardware. The original has
magnetic charging; Sphere C has USB-C. The dimensions, weight, battery,
maximum output, IP rating, app family, and control distance remain
substantially the same.

Sources:

- Original manual:
  https://cdn.olightstore.com/file/sphere-user-manual.pdf
- Sphere C manual:
  https://cdn.olightstore.com.au/file/user-manual-sphere-c.pdf
- Tuya mesh DP/model mapping:
  https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_data_control?id=Kd5wq9xmg0u3i
- Tuya mesh categories:
  https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_category?id=Kd5wqhyn32zoe
- Public Sphere C DP profile:
  https://github.com/make-all/tuya-local/blob/main/custom_components/tuya_local/devices/olight_sphere_light.yaml

## Concrete model mapping

Tuya documents this lighting mapping:

| DP | Function | Mesh model |
|---:|---|---|
| 1 | Power | Generic OnOff |
| 2 | White/color mode | Tuya vendor model |
| 3 | Brightness | Light Lightness |
| 4 | Color temperature | Light CTL Temperature |
| 5 | Color | Light HSL |
| 6 | Scene | Tuya vendor model |
| 7 | Music | Tuya vendor model |
| Other | Product-specific state | Tuya vendor model |

The public Sphere C gateway profile uses the same first five DP IDs.
It also identifies:

- Product ID `d0dpga0h`
- Timer DP `7`
- Scene DP `102`
- Battery DP `103`
- Sleep DP `104`

The gateway profile describes cloud/gateway DP semantics. Over the
Bluetooth Mesh bearer, Tuya maps the common lighting DPs to SIG model
opcodes. Therefore, DP numbers are useful semantic hints but are not
bytes that necessarily appear in every direct Mesh command.

## Structural matches in this capture

For an unsegmented access message with four-byte Network and Transport
MICs:

```text
Network PDU length = 18 + Access payload length
```

This lets encrypted packet lengths be compared with known SIG model
message lengths. The following isolated request/response shapes occur:

| Network PDU shape | Count | Strong candidate |
|---|---:|---|
| 20 -> 21 | 11 | Generic OnOff Get -> Status |
| 22 -> 21 | 3 | Generic OnOff Set -> Status |
| 20 -> 22 | 1 | Light Lightness Get -> Status |
| 20 -> 24 | 1 | Light CTL Temperature Get -> Status |
| 20 -> 26 | 2 | Light HSL Get -> Status |
| 27 -> 26 | 5 | Light HSL Set -> Status |

The two high-confidence shapes are especially persuasive:

- Generic OnOff Get is a two-byte opcode, producing a 20-byte Network
  PDU. Generic OnOff Status adds one state byte, producing 21 bytes.
- Generic OnOff Set adds state and transaction ID, producing 22 bytes,
  followed by the same 21-byte status.
- Light HSL Set contains a two-byte opcode, three 16-bit values, and a
  transaction ID. Its nine-byte access payload produces a 27-byte
  Network PDU. The six-byte HSL state plus opcode in Status produces
  a 26-byte Network PDU.

This is structural evidence, not authenticated decryption. An unrelated
vendor message can have the same length. Repetition of the expected
request/status pair makes coincidence less likely.

## Tuya vendor traffic fit

Tuya's vendor model uses company ID `0x07d0` and these opcodes:

| Operation | Opcode |
|---|---:|
| Write | `0xC9D007` |
| Write unacknowledged | `0xCAD007` |
| Read | `0xCCD007` |
| Data/status | `0xCDD007` |

A vendor Read of one or all DPs can occupy a six-byte access payload,
which gives a 24-byte Network PDU. The capture contains 16 phone-to-
proxy PDUs of length 24. Several are followed by repeated response fans
with lengths:

```text
22, 28, 28, 25, 28, 25
```

Tuya DP status units naturally vary in length by data type. Boolean and
enum values are short, while four-byte values and raw/string data are
longer. This response fan is consistent with a query that causes the
Sphere or a group of nodes to report several standard and vendor
states. Keys are still required to prove the opcodes and DP IDs.

## What original-Sphere work can contribute immediately

High-value reusable inputs are:

- A NetKey/AppKey extraction method for `com.olight.omall`
- Any exported mesh configuration database
- Original-Sphere composition data
- Known node element addresses and model bindings
- Decrypted access opcodes and parameters
- Product ID and Tuya DP schema
- Scene and effect payload formats

If original-Sphere code implements only the old proprietary Telink/Tuya
service `0xfe07`, it is not directly applicable. Sphere C is using
Bluetooth SIG Mesh, not the older proprietary Telink mesh framing.
The control semantics may still help, but its packet encoder and keys
would not.

## Compatibility boundary

Safe to reuse:

- Mesh crypto and nonce construction
- Proxy GATT client
- Replay and sequence-number persistence
- Generic OnOff, Light Lightness, CTL Temperature, and HSL models
- Tuya company ID and vendor model framing
- Olight/Tuya app instrumentation strategy

Must be discovered or verified per model:

- Mesh composition and element count
- Product ID
- Node address and all security keys
- Supported optional SIG states
- Vendor DP list
- Scene IDs and compressed scene format
- Battery and sleep behavior
- Firmware and OTA behavior

## Public-source search boundary

No public implementation specifically identifying itself as an
original-Sphere protocol implementation was found. The reusable-layer
conclusion therefore comes from the official feature descriptions,
Sphere C's captured standards-based transport, Tuya's documented model
mapping, and the matching encrypted message shapes. It is not a claim
that an original-Sphere command table has already been tested unchanged
against Sphere C.

## Reproduction

`infer_mesh_models.py` reads the exact extracted PDU inventory and lists
isolated request/response pairs with structural model candidates:

```sh
python3 infer_mesh_models.py
```

These candidate labels should be replaced with decoded results as soon
as the NetKey and AppKey are available.
