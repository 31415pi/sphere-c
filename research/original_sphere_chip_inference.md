# Original Sphere MCU and firmware-stack inference

## Bottom line

The captured Sphere C GATT table is strong evidence for a
**Telink Bluetooth Mesh SDK-derived firmware stack**. It is not, by
itself, evidence for one exact Telink part number.

The most discriminating signatures are not the Tuya UUID or company
ID. They are the complete Telink OTA UUID pair and Telink's otherwise
unused `0x7fdd` service. Both occur byte-for-byte in Telink's official
Mesh SDK source and in the captured device's attribute table.

This raises Telink silicon from a generic guess to the leading hardware
hypothesis. It does **not** distinguish TLSR8258, TLSR8278, B91, or
another Telink target on which the same stack was ported. No exact MCU
marking, firmware image, SWire response, or chip-specific ROM behavior
has yet been obtained.

The evidence below was measured on Sphere C. Applying it to the
magnetic-charge original Sphere remains a family-level inference until
the original device's GATT table or PCB is checked.

## 1. Exact Telink OTA fingerprint

The capture exposes this 128-bit service and characteristic:

```text
service:        00010203-0405-0607-0809-0a0b0c0d1912
characteristic: 00010203-0405-0607-0809-0a0b0c0d2b12
```

These are not generic Bluetooth Mesh or Tuya UUIDs. In Telink's
official `tl_ble_mesh` repository, `firmware/stack/ble/service/uuid.h`
defines the identical byte sequences as `TELINK_OTA_UUID_SERVICE` and
`TELINK_SPP_DATA_OTA`. The Mesh GATT table in
`firmware/vendor/mesh/app_att.c` instantiates those exact definitions
as `my_OtaServiceUUID` and `my_OtaUUID`.

Sources:

- [Telink UUID definitions at the inspected commit](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/firmware/stack/ble/service/uuid.h#L655-L661)
- [Telink Mesh GATT-table use](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/firmware/vendor/mesh/app_att.c#L223-L229)

The capture also exposes a second Telink-shaped 128-bit pair:

```text
service:        00010203-0405-0607-0809-0a0b0c0d7fde
characteristic: 00010203-0405-0607-0809-0a0b0c0d7fdf
```

The same Telink source defines these as `TELINK_USERDEFINE_GATT` and
`TELINK_USERDEFINE_UUID`. Together with the OTA pair, this is a much
stronger implementation fingerprint than normal SIG Mesh services.

Source:

- [Telink custom and unused GATT definitions](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/firmware/vendor/common/chip_adapt_layer/al.h#L143-L145)

## 2. The unusual `0x7fdd` service is also in Telink source

The device exposes service `0x7fdd` with standard Mesh Proxy Data In
and Out characteristics `0x2add` and `0x2ade`. Telink's source defines
`SIG_MESH_ATT_UNUSED` as little-endian `{0xdd, 0x7f}`. Its default Mesh
configuration says that enabling `ATT_REPLACE_PROXY_SERVICE_EN` makes
`0x7fdd` disappear from the attribute table. With the default disabled,
the stack retains a second proxy-shaped attribute-table slot and swaps
the active SIG provisioning/proxy service according to device mode.

Telink's own Android sample log shows the same combination:
`0x1827`, `0x7fdd`, and the corresponding standard Mesh
characteristics.

Sources:

- [Telink `SIG_MESH_ATT_UNUSED` definition](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/firmware/vendor/common/chip_adapt_layer/al.h#L148-L156)
- [Telink configuration comment about `0x7fdd`](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/firmware/vendor/common/mesh_config.h#L134-L138)
- [Telink sample discovery log](https://github.com/telink-semi/tl_ble_mesh/blob/eae736a26fed53041922a28c62633703b22233ad/app/android/TelinkBleMesh/sample-data/scan_prov_bind_autoconnect_get_onoff.txt#L187-L199)

This exact combination makes a Telink-derived stack highly likely.
It is still logically possible that Tuya or Olight copied/ported the
Telink attribute layout to non-Telink silicon.

## 3. What the Tuya identifiers prove, and what they do not

The device also exposes:

- service `0xfd50`;
- write characteristic
  `00000001-0000-1001-8001-00805f9b07d0`;
- notify characteristic
  `00000002-0000-1001-8001-00805f9b07d0`;
- manufacturer/company ID `0x07d0`.

Tuya's official porting guide specifies exactly this service, these
characteristics, and company ID. Therefore these values prove use of
the Tuya Bluetooth SDK/interface layer.

They do not identify the MCU. The same official guide explicitly uses
Nordic nRF52832 sample code and describes the SDK as a porting layer
whose application supplies the BLE service. Thus `0xfd50` is designed
to be platform-independent.

Source:

- [TuyaOS Bluetooth SDK porting guide](https://developer.tuya.com/en/docs/iot-device-dev/Porting-Guide-BLE?id=Kam0xjtz4n6e0)

Likewise, composition company ID `0x07d0` and vendor models `0x0004`
and `0x0005` identify the application/vendor model namespace. They are
not silicon IDs.

## 4. Architecture that can be inferred

The evidence supports this firmware architecture:

1. A Telink SDK-derived BLE controller and SIG Mesh stack provides
   PB-GATT, Proxy GATT, mesh crypto/configuration, Telink GATT OTA, and
   the characteristic Telink attribute-table layout.
2. A Tuya/Thingclips integration layer adds the `0xfd50` service,
   Tuya advertising/manufacturer data, mesh vendor models, datapoints,
   and cloud/app provisioning semantics.
3. Olight product firmware adds discovery service `0xfcf1`, product
   behavior, LED effects, charging/battery logic, and product-specific
   datapoints.

This looks like one BLE SoC running the radio, mesh stack, OTA, and
application, rather than a separate generic radio module plus host MCU.
That last point is an inference from the unified GATT/OTA/mesh layout,
not direct board proof.

The captured composition is consistent with an SDK light example:
company `0x07d0`, product `0x0200`, version `0x3530`, six elements,
standard OnOff/Lightness/CTL/HSL models, and vendor models `0x0004`
and `0x0005`. Telink documents light-node examples with these standard
model families and GATT/mesh OTA, but composition is configurable and
does not narrow the chip.

Source:

- [Telink SIG Mesh documentation](https://doc.telink-semi.cn/doc/en/software/res/sdk/ble_mesh/sig_mesh_en/sig_mesh_en/)

## 5. Why an exact MCU cannot yet be named

Telink has reused this SDK architecture across multiple MCU families.
Public Telink material includes TLSR8258-era examples, while the current
official repository contains B91 and TL321X targets. UUIDs are compiled
protocol constants and survive ports between those targets.

Consequently, statements such as “the Sphere uses TLSR8258” remain
plausible but unproven. The available protocol evidence establishes a
Telink software lineage, not a package marking or CPU core.

The original Sphere's public FCC filing is the best currently known
direct-hardware source. FCC ID `2A44S-SPHERE`, exhibit `7603966`, is a
five-page internal-photo filing. The public index confirms the exhibit
and its SHA-256, but no legible MCU marking was recovered during this
review. The filing must not be cited as naming a chip until a readable
photo actually does so.

Sources:

- [Original Sphere FCC filing](https://fccid.io/2A44S-SPHERE)
- [Original Sphere internal-photo exhibit](https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966)

## 6. Highest-value confirmation steps

1. Dump the original magnetic Sphere's complete GATT table. Finding
   all four fingerprints (`0x7fdd`, Telink OTA, Telink user-defined
   UUID, and Tuya `0xfd50`) would strongly link both generations.
2. Obtain a macro PCB photo and read the QFN marking. This is the
   shortest path to an exact part.
3. Capture an Olight firmware update. The image header, load address,
   packet size, and CRC may match a specific Telink SDK generation.
4. If hardware access is acceptable, identify SWire pads and query the
   part without erasing flash. Do not infer a pinout from package shape.
5. Compare the firmware image against official SDK reset vectors,
   flash layout, and OTA header definitions before naming TLSR8258 or
   any newer family.

## Confidence summary

| Claim | Confidence | Reason |
|---|---:|---|
| Tuya/Thingclips BLE integration | Very high | Exact official `0xfd50` interface |
| Telink Mesh SDK lineage | High | Exact OTA, custom, and `0x7fdd` fingerprints |
| Single Telink BLE SoC architecture | Medium-high | Normal Telink light-node design; no board proof |
| Same stack in original Sphere | Medium | Product-family/app evidence, not yet its GATT dump |
| Exact MCU is TLSR8258 | Low | Compatible hypothesis, no unique discriminator |
