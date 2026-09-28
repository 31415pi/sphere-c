# Original Olight Sphere compatibility

Research date: 2026-09-27. This note evaluates whether the current
Sphere C tools are suitable for the original magnetic-charge Olight
Sphere (FCC ID `2A44S-SPHERE`). It does not claim that the original
Sphere has been tested with this repository.

## Assessment

The code is not established to work unchanged with the original
Sphere. It is a reasonable candidate for careful, read-only
discovery, and the generic Bluetooth Mesh transport and SIG lighting
messages are likely reusable if the original reports the corresponding
models. Product-specific operations need verification on that unit.

There is no authenticated original-Sphere composition dump, GATT dump,
Mesh capture, or DP read in this repository. The actual captured and
provisioned device is Sphere C: CID `0x07d0`, PID `0x0200`, VID
`0x3530`, six elements, and vendor models `0x07d0:0x0004` and
`0x07d0:0x0005`. These values must not be assumed for the original.

The best current conclusion is same product family and probably the
same broad Tuya/Telink Mesh platform, with unverified model-level
differences. Confidence is moderate for family-level compatibility,
low for unchanged product-specific behavior, and absent for a
successful original-Sphere session.

## Evidence for shared behavior

The original Sphere manual describes the same Olight app family and
the same core lamp features present on Sphere C: on/off, brightness,
white and color output, selectable modes, music rhythm, groups,
automation, gateway support, and sleep behavior. This is evidence of
shared product behavior, not a protocol trace.

Sphere C's Android capture shows standard Bluetooth Mesh Provisioning
(`0x1827`) and Proxy (`0x1828`) GATT, Tuya company ID `0x07d0`, Tuya
service `0xfd50`, and Olight discovery service `0xfcf1`. The capture
does not include the NetKey or AppKey, so its encrypted access payloads
are not proof of the original Sphere's message set.

The captured Sphere C GATT table also matches Telink Mesh SDK UUIDs
for OTA and user-defined services and contains the unusual `0x7fdd`
proxy-shaped service. This is strong evidence of Telink SDK lineage
for Sphere C. The original's FCC photo shows an integrated radio-board
QFN but its top marking is obscured. A possible `SWS` test-pad label
is suggestive, not enough to identify a Telink part or prove identical
firmware.

Tuya's public lighting documentation maps common lighting functions
to the Bluetooth Mesh SIG models. Sphere C implements these live:
Generic OnOff, Light Lightness, CTL/CTL Temperature, and HSL. If the
original composition reports the same model IDs, standard opcodes and
payload formats are defined by Mesh and are strong reuse candidates.
The original's model list and configured ranges remain unknown.

## What may be reused

Likely reusable after confirming standard services and models:

- Bluetooth Mesh Proxy GATT transport and Mesh cryptography;
- key extraction/provisioning concepts used by the Olight app;
- standard SIG OnOff, Lightness, CTL Temperature, and HSL messages;
- Tuya vendor framing if the original composition advertises that
  vendor model and the company ID matches.

The code discovers the destination element for SIG models and vendor
server `0x07d0:0x0004` from saved composition data. This helps avoid
hard-coding Sphere C's primary address layout. It does not make the
commands safe for a different firmware: code still assumes the same
model meanings, encodings, and vendor server ID when those models are
present.

## Differences and unknowns

Confirmed physical distinction: original Sphere charges through a
magnetic USB cable; Sphere C uses USB-C. The original manual specifies
a 3.7 V, 700 mAh battery and magnetic charging. The original and C
have separate FCC IDs (`2A44S-SPHERE` and `2A44S-SPHEREC`). Public
photos do not identify the original MCU conclusively. Firmware and
OTA behavior may differ, and the original has reported firmware
updates including user-reported version 3.5; no original firmware
image or OTA trace was found.

Not established for the original:

- Bluetooth Mesh versus a proprietary/Telink-only control path;
- GATT services, advertising behavior, address rotation, and proxy
  availability;
- company/product/version IDs, element count, and model bindings;
- whether it exposes Lightness, CTL, HSL, or the same vendor models;
- its Tuya cloud product ID, DP schema, effect IDs, and payloads;
- firmware version, supported ranges, and behavior after provisioning;
- whether app-extracted keys or network state can be reused across
  generations. Treat keys and node records as device/network-specific.

The current code's DP 102 effect catalog, DP 101 countdown control,
DP 103 battery interpretation, and other Sphere C datapoint meanings
come from Sphere C's saved app schema and live tests. They are not
portable assumptions. The Sphere C cloud PID `d0dpga0h` is not its
Mesh PID and is not known to identify the original product.

## Cautious trial procedure

1. Identify the unit physically as the magnetic-charge original and
   record its app-reported firmware version. Keep it in its current
   Olight app setup. Do not factory-reset it or start provisioning.
2. Make a protected copy of the current app/device configuration if
   available. Do not point this tool at Sphere C's live state file,
   keys, IV index, sequence cursor, node address, or composition.
3. Begin with passive BLE scanning only. Record fresh advertisements,
   service UUIDs, service data, and manufacturer data. Do not replay
   captured packets. A random address by itself is not a stable ID.
4. If a GATT inspection is needed, read the service/characteristic
   table and read-only characteristic values only. Do not write to
   provisioning, OTA, vendor, or control characteristics. Compare
   `0x1827`, `0x1828`, `0xfd50`, `0xfcf1`, Telink UUIDs, and `0x7fdd`
   with the Sphere C evidence; a match supports shared stack lineage,
   not full command compatibility.
5. If the device is already provisioned, recover its own valid Mesh
   credentials and composition through the Olight app artifacts or an
   authorized read-only capture. A new controller needs the matching
   NetKey, AppKey, IV Index, unicast address, and sequence state. Do
   not guess keys or use the Sphere C network state.
6. Before any Mesh control, decode the original's Composition Data
   page 0 and compare supported models and element addresses. Start
   with authenticated Get requests only. A successful response proves
   that particular read path; it does not validate vendor commands.
7. Only after a verified backup and positive read-only results, test
   a reversible standard Set: save current state, change one small
   step, read it back, and restore the saved value. Confirm the final
   read. Avoid effect selection, countdowns, factory-mode reset,
   firmware update, unbind/reset, and any undocumented vendor writes.
8. Stop if the lamp disappears from the Olight app, no longer
   advertises its previous Mesh Network ID, or fails authenticated
   reads. Do not reset or reprovision as an automatic recovery step;
   restore the saved Olight configuration or use Olight's normal
   recovery path first.

Provisioning changes durable network membership, keys, and model
bindings. The tested Sphere C workflow is specifically for a
factory-reset Sphere C and commits state that survives disconnect and
button-off/wake. It should not be used on the original as a discovery
shortcut. A reset or replacement network could remove ordinary app
control until the original is reprovisioned through a compatible
workflow.

## Repository evidence and sources

Primary repository evidence:

- `docs/analysis/capture.md`: Sphere C capture and GATT evidence;
- `docs/analysis/compatibility.md`: prior generation comparison and
  structural model candidates;
- `docs/research/mesh/step4_sphere_model_map.md`: authenticated
  Sphere C composition and model mapping;
- `docs/guides/self-provisioning.md`: persistent Sphere C provisioning
  and its recovery implications;
- `docs/guides/vendor-datapoints.md` and
  `docs/research/mesh/vendor_dp_inventory.md`: Sphere C DP evidence;
- `docs/research/hardware/original_sphere_hardware_public.md`,
  `original_sphere_chip_inference.md`, and
  `original_sphere_firmware_artifacts.md`: original hardware and
  firmware evidence;
- `docs/project/original-task.md`: original capture context. There is
  no root `TASK.md`; the referenced original capture is Sphere C.

Recorded source links (primary where available):

- [Original Sphere manual](https://cdn.olightstore.com/file/sphere-user-manual.pdf)
- [Sphere C manual](https://cdn.olightstore.com.au/file/user-manual-sphere-c.pdf)
- [Tuya Mesh data control mapping](https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_data_control?id=Kd5wq9xmg0u3i)
- [Tuya Mesh categories](https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_category?id=Kd5wqhyn32zoe)
- [Sphere C public Tuya profile](https://github.com/make-all/tuya-local/blob/main/custom_components/tuya_local/devices/olight_sphere_light.yaml)
- [Original Sphere FCC filing](https://fccid.io/2A44S-SPHERE)
- [Original Sphere internal photos](https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966)
- [Telink TLSR8258 datasheet](https://wiki.telink-semi.cn/doc/ds/DS_TLSR8258-E_Datasheet%20for%20Telink%20BLE%2BIEEE802.15.4%20Multi-Standard%20Wireless%20SoC%20TLSR8258.pdf)
- [Tuya TLSR825x Bluetooth Mesh SDK](https://developer.tuya.com/en/docs/iot-device-dev/bluetooth-sigmesh-lighting-sdkdemo-instructions?id=Ka5e0524ewo4n)

No web search was needed for this note: the local research files
already preserve the source links and evidence boundary. No original
Sphere live interoperability result is claimed.
