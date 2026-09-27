# Olight Sphere firmware and update artifact research

## Executive result

No Olight Sphere or Sphere C firmware image is bundled in the
available APK or copied external phone files. In particular, there is
no `.bin`, `.hex`, `.ota`, `.fw`, or device-image archive to inspect
for an MCU vector table, image magic, linker strings, or flash layout.

The APK does contain a complete ThingClips/Tuya OTA client. It queries
authenticated, region-selected Tuya APIs for product- and
device-specific update metadata, downloads the URL returned by the
cloud, verifies MD5, and then sends the bytes over Tuya BLE or SIG
Mesh OTA framing. The useful outcome is therefore an acquisition map,
not a recovered firmware binary.

The evidence below deliberately separates:

1. firmware that is physically bundled or cached locally (none found),
2. cloud OTA metadata and transfer implementation (present), and
3. MCU-family inferences (suggestive, not identified).

## Local evidence set

### APK identity

Inspected file:

```text
old/apk/olight-3.2.5.apk
SHA-256: 1fa9bddeb55ec39fba631e074be76f65a
         da37830735ae9a074f2aac83b6eecb9
```

Manifest and package facts:

- package `com.olight.omall`
- version `3.2.5`, version code `239`
- minimum SDK 24; target/compile SDK 36
- main activity
  `com.olightmall.core.login.ui.activity.SplashActivity`
- wrapped application `com.Proxy.ShellApplication`
- 25 DEX files
- no native libraries in this base/split APK artifact

The APK is v2/v3 signed. Two certificates are exposed by the signing
block (likely app signing plus Play-related lineage/material):

```text
SHA-256 a927605db2971af934df587979e6983cc
        b858f0f4d2285faf10e7ab2829aa734
Subject CN=Android, OU=Android, O=Google Inc.

SHA-256 56c3d8ed2b08527375ed8a5a5c20d745
        747aedbeabe9ae737447f0ac8032b816
Subject CN=ch
```

These are software-distribution fingerprints, not device-firmware
signing keys.

### Bundled firmware search

The APK ZIP listing and all project/phone files were searched for
common firmware extensions and update-related filenames. Results:

- no device `.bin`, `.hex`, `.fw`, `.ota`, or `.img`
- no firmware-containing ZIP archive
- no cached download beneath `old/phone_external/`
- the only APK `.bin` entries are Kotlin/compiler metadata, such as
  `DebugProbesKt.bin` and `previous-compilation-data.bin`
- `/layout/lay.bin` and `/route/rou.bin` occur as generic SDK strings,
  not as packaged Sphere firmware files
- copied Olight log files are encrypted/base64-looking records and do
  not expose plaintext firmware URLs or version metadata

Consequently there is no local firmware image header to characterize.
The APK string `Failed to find image header parser` comes from an
ordinary image-loading library context and is not evidence of an OTA
firmware header parser.

## Sphere-related identifiers in local artifacts

The copied ThingClips mini-app is a concrete Sphere C artifact:

```text
Mini-app directory ID: 9E6DD95017EDCABEA0673DC3A9AFAC12
Mini-app version:      0.5.9_RsPYtSG2yNGWYhKJ
Mini-program ID:       tycryc71qaug8at6yt
Settings entry code:   entrye0n05idydmmfv
```

Its `app-config.json` identifies these runtime dependencies:

```text
Ray renderer: 1.5.44
BaseKit:      3.0.6
MiniKit:      3.0.7
DeviceKit:    4.11.8
BizKit:       4.2.0
MediaKit:     3.2.0
```

The public and already-correlated Sphere C Tuya product ID is:

```text
d0dpga0h
```

That identifier occurs in the copied mini-app localization path and
is also the product ID in the public `tuya-local` Sphere C profile.
It is a Tuya cloud product ID, not the Bluetooth Mesh Composition Data
product ID.

The physical Sphere C in this repository reports these independent
Bluetooth Mesh Composition Data identifiers:

```text
Company ID:  0x07d0 (Olight)
Product ID:  0x0200
Version ID:  0x3530
```

Do not substitute `0x0200` for `d0dpga0h` in cloud APIs. The original
magnetic-charge Sphere and USB-C Sphere C must also not be assumed to
share a Tuya PID or firmware, even though they share the product
family and control semantics. No original-Sphere Tuya PID was found
in the inspected local artifacts.

## ThingClips/Tuya OTA software fingerprint

The APK contains the following component metadata:

| Component | Version |
|---|---:|
| `thingsmart-bleota-api` | `4.5.0-rc.3` |
| `thingsmart-bleota` | `6.4.1` |
| `thingsmart-ota-api` | `6.6.0` |
| `thingsmart-ota-biz` | `6.6.0` |
| `thingsmart-ota-uikit` | `6.8.0` |
| `thingsmart-ota` | `6.4.5` |
| `thingsmart-upgrade` | `6.4.0` |
| `thingsmart-sdk-api` | `4.5.0-rc.2` |
| `tunimeshmanager` | `1.1.0` |

Module routes and services include:

```text
single_ble_ota_upgrade
update_firmware
device_upgrades_list

com.thingclips.smart.ota.api.BleOtaService
com.thingclips.smart.bleota.BleOtaServiceImpl
com.thingclips.smart.panel.ota.service.AbsOTACheckService
com.thingclips.smart.panel.newota.OTACheckService
```

The APK includes both direct BLE and SIG Mesh implementations. Key
class fingerprints include:

```text
com.thingclips.sdk.ble.core.protocol.entity.BleOtaParam
com.thingclips.sdk.ble.core.packet.bean.OTAStartRep
com.thingclips.sdk.ble.core.packet.bean.OTAFileRep
com.thingclips.sdk.sigmesh.bean.OTAStartRep
com.thingclips.sdk.sigmesh.bean.OTAFileRep
com.thingclips.smart.bleota.presenter.NewFirmwareUpgradeBLEPresenter
```

The generic SDK supports many device families. Their presence alone
does not prove which branch the Sphere uses. The Sphere C's observed
SIG Mesh transport makes the SIG Mesh branch the relevant candidate.

## Cloud update API and returned artifacts

### API selectors

DEX disassembly shows these authenticated API calls:

```text
m.thing.firmware.upgrade.info.get  version 2.0
POST: devId=<device virtual ID>
Response: FirmwareCheckResult

m.thing.firmware.upgrade.info.get  version 1.1
POST: devId=<device virtual ID>
Response: array of BLEUpgradeBean

m.thing.firmware.upgrade.sign.get  version 1.0
POST: devId, type, signType, version
Response: String

m.thing.firmware.upgrade.confirm   version 1.0
POST: devId, types

m.thing.firmware.upgrade.confirm   version 2.0
POST: devIds (JSON)
```

All set `sessionRequire=true`. They are ThingClips signed API
selectors, not literal REST URL paths. Existing static analysis in
`docs/research/app/tls_intercept.md` identifies the region-selected mobile API
bases as:

```text
CN: https://a1.tuyacn.com
US: https://a1-us.iotbing.com
EU: https://a1.tuyaeu.com
IN: https://a1-in.iotbing.com
```

A valid Olight session and the native app-bound signing process are
required. A copied URL or bare device ID is insufficient.

### Update metadata objects

`UpgradeInfoBean`, returned by the SDK's OTA information path, exposes:

```text
url, md5, sign, fileSize, version, currentVersion
type, devType, upgradeType, upgradeMode, upgradeStatus
firmwareDeployTime, lastUpgradeTime, timeout
canUpgrade, disableBleDirectOTA, autoSwitch
```

The downloaded-package object `FirmwareUpgradeInfoBean` adds:

```text
url, filePath, md5, sign, hmac, diffOta
```

The older mesh `BLEUpgradeBean` contains:

```text
url, md5, sign, fileSize
```

These fields establish that the firmware URL and authenticity data are
delivered dynamically by the Tuya cloud. They also show support for
signed/HMAC and differential packages, although they do not prove
that a specific Sphere release uses all of those options.

The code logs `Download checkFirmwareFile MD5 error` and has error
code `5009` for BLE OTA MD5 mismatch. Therefore a captured OTA session
should preserve the response JSON, downloaded bytes, HTTP headers,
and the computed MD5 before attempting to interpret the file.

## OTA wire/image metadata visible in the APK

The low-level Tuya BLE structures line up with Tuya's public OTA
protocol documentation.

`BleOtaParam` contains:

```text
firmwareData: byte[]
productId:    String
type:         int
version:      String
```

Channel/type constants are:

```text
normal firmware type:    0
MCU firmware type:       1
normal channel ID:       1
MCU channel ID:          9
custom channel IDs:      10 through 19
```

`OTAStartRep` parses the device response as:

```text
allowUpdate, otaVersion, type, current version, maxLimit,
flag2, signatureAlgorithm, keyVersion
```

Types 10 through 19 are accepted as custom channels. This is relevant
because a Sphere package might be classified as a custom firmware
channel rather than generic type 0.

`OTAFileRep` and its SIG Mesh equivalent track:

```text
type, state, alreadyLength, alreadyCRC32, alreadyMD5
```

Tuya's official BLE OTA protocol defines the file-info message as:

```text
1 byte   channel/type (0 for the documented default)
8 bytes  PID/reserved field
4 bytes  target version
16 bytes MD5
4 bytes  file length
4 bytes  CRC32
```

The protocol supports offset queries and resumable transfer. Tuya's
Bluetooth Mesh OTA documentation makes an important platform-specific
exception: resumable download is supported on all adapted chip
platforms except TLSR825x.

None of this describes an Olight-specific file header. It describes
the transfer envelope around opaque firmware bytes. Only a captured
download can reveal whether the actual Sphere image begins with a
Telink boot marker, Tuya signature header, raw vector table, compressed
container, or an encrypted/signed wrapper.

## MCU-family clues and limits

### What can be said

- The Sphere family is implemented with Olight/Tuya Bluetooth SIG
  Mesh software, not merely Android's standard Device Firmware Update
  service.
- Tuya's current Bluetooth Mesh device kit documentation explicitly
  names TLSR825x as a supported mesh platform and singles it out in
  OTA behavior (no resumable download).
- The APK preserves the historical Tuya BLE/SIG Mesh OTA protocol
  classes, including MD5, CRC32, signature algorithm, key version,
  and custom firmware channels.
- Existing project research notes that older proprietary Tuya/Telink
  products used service `0xfe07`, but Sphere C is standards-based SIG
  Mesh. That protocol distinction does not by itself identify silicon.

### What cannot be said

There is no local binary, PCB marking, crash log, compiler string,
map file, vector table, or device-reported chip ID tying this unit to
TLSR825x, PHY6222, nRF52, Realtek, Bouffalo, or another MCU family.
TLSR825x is the leading ecosystem clue, not a confirmed identification.

The FCC filing for the original Olight Sphere is `2A44S-SPHERE`, with
internal-photo exhibit 7603966. Its public text establishes the radio
range and exposes board photographs, but the searchable exhibit
metadata does not name the MCU. A visual read of a legible package
marking or an electrical/debug-port examination remains necessary.
Sphere C has a distinct FCC ID, `2A44S-SPHEREC`.

## Best acquisition path

The shortest path to the original firmware artifact is an authorized
runtime capture:

1. Use a test account/device for the exact Sphere model and region.
2. Intercept the signed ThingClips request named
   `m.thing.firmware.upgrade.info.get` while opening Device Update.
3. Preserve the complete response containing `url`, `md5`, `sign`,
   `hmac`, `version`, `type`, and `fileSize` where present.
4. Capture the subsequent CDN request without modifying the payload.
5. Save the raw response body and verify MD5 and size before unpacking.
6. Record the device's OTA-start response (`otaVersion`, `type`,
   `maxLimit`, signature algorithm, key version).
7. Compare behavior after an interrupted transfer. Lack of resume is
   useful supporting evidence for, but not proof of, TLSR825x.

The app trusts system CAs and may enable OkHttp certificate pinning.
The exact authorized interception constraints are already documented
in `docs/research/app/tls_intercept.md`.

## Primary and direct references

- Tuya, BLE OTA protocol and deployment:
  <https://developer.tuya.com/en/docs/iot-device-dev/OTA_BLE?id=Kawer1ehuv8si>
- Tuya, Bluetooth Mesh OTA behavior:
  <https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_OTA?id=Kd5wtdf7b4ytk>
- Tuya, Bluetooth Mesh firmware identity initialization:
  <https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_map_mesh_init?id=Kc9ah4zf5lbuw>
- Tuya, Bluetooth Mesh device kit capability map:
  <https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_software_mesh?id=Kc6fiv84nxl5d>
- Original Sphere FCC internal-photo exhibit metadata:
  <https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966>
- Public Sphere C Tuya profile:
  <https://github.com/make-all/tuya-local/blob/main/custom_components/tuya_local/devices/olight_sphere_light.yaml>

## Confidence summary

High confidence:

- no firmware binary is present in the inspected local artifacts
- the Olight app uses ThingClips/Tuya OTA modules
- update metadata is fetched per device from authenticated Tuya APIs
- the metadata includes URL, size, MD5, and signing fields
- the transfer protocol carries version, MD5, length, and CRC32

Medium confidence:

- the Sphere C normally follows the APK's SIG Mesh OTA branch
- a clean runtime update check is sufficient to recover the download
  URL if an update is currently deployed to that device/PID

Low confidence / hypothesis only:

- the original Sphere or Sphere C uses a Telink TLSR825x MCU
- the original and C models share firmware or cloud product identity
