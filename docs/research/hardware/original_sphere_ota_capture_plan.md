# Original Sphere OTA capture plan

Status: parked for a future experiment. Do not provision the boxed
original Sphere until the capture tooling has been tested.

## Opportunity

The first original Sphere downloaded an approximately 140 KB firmware
update during initial provisioning. A second boxed Sphere from the
same batch may offer the same update. Treat that device as a one-shot
capture opportunity: a successful update may remove the cloud offer.

## Preferred capture point

Capture the firmware inside the official Android app after download
and verification, but before Bluetooth OTA framing. This avoids both
HTTPS certificate pinning and Bluetooth Mesh encryption.

The inspected Olight APK contains these useful SDK objects and fields:

```text
BleOtaParam.firmwareData: byte[]
UpgradeInfoBean: url, md5, sign, fileSize, version, currentVersion
FirmwareUpgradeInfoBean: url, filePath, md5, sign, hmac, diffOta
BLEUpgradeBean: url, md5, sign, fileSize
```

The exact runtime class used by the original Sphere must be confirmed
with instrumentation; the SDK includes direct-BLE and SIG Mesh paths.

## Prepared experiment

Use a disposable rooted Android phone with the official, unmodified
Olight app and Frida server. Prepare hooks that:

1. Log the complete firmware-check metadata object.
2. Record HTTP response headers and the authenticated response JSON.
3. Copy any staged file as soon as its `filePath` is assigned.
4. Dump the final firmware byte array passed to the OTA implementation.
5. Compute SHA-256 and MD5 without altering the captured file.

Also enable Android Bluetooth HCI snooping and retain the app logs.
These are supporting records, not the primary extraction route.

An HTTPS proxy is optional corroboration. The release app trusts only
system CAs and includes OkHttp certificate-pinning machinery, so a
normal user-installed proxy CA on an unrooted phone is insufficient.

## Before touching the boxed Sphere

- Test all hooks against the installed app without accepting an OTA.
- Confirm that firmware-check objects are observed and serialized.
- Confirm that binary dumps can be moved off the phone intact.
- Start HCI and app logging before provisioning.
- Keep the boxed Sphere unpaired until those checks pass.

## Artifacts to retain

- raw firmware bytes, without conversion;
- firmware URL and complete metadata JSON;
- MD5, signature/HMAC, version, type, and differential-OTA flags;
- HTTP response headers;
- Frida event log and app log;
- Bluetooth HCI snoop covering provisioning and OTA;
- Olight app version, APK hash, phone model, and Android version.

Do not publish device, account, session, mesh, or cloud credentials
with a future firmware sample.

## Preparation checkpoint: 2026-10-04

The report and [protocol reference](../../../SPHERE-C-BT.md)
can be completed without this acquisition. The Sphere C was
reachable through its existing local network on this date.
That does not establish any update offer or firmware identity
for the boxed original Sphere.

No phone instrumentation, account access, original-device pairing
or OTA transfer was performed during documentation. Hook and
phone-export validation remain pending. Do not mark them tested
because the SDK contains a promising class or field name.

### Exact-device preflight

| Check | Evidence needed before acquisition |
|---|---|
| Device generation | Explicit original Sphere versus Sphere C; charging connector and product identity |
| Installed app | Version, package and SHA-256 of the actual APK being instrumented |
| Runtime path | Loaded direct-BLE/SIG Mesh classes and real metadata object, not merely a static class match |
| Hook coverage | Metadata, staged-file and final-byte-array capture exercised without accepting a device update |
| Export integrity | Synthetic byte array/file exported through the intended phone-to-host path with equal size and SHA-256 |
| Private destination | New ignored directory; restricted file permissions; no overwriting a prior capture |
| Logs | HCI/app/instrumentation capture started and timestamps synchronized |
| Operation boundary | Agreed device/account and whether the app can download without starting installation |

If the app combines download and device update, stop preparation
at the observation point and agree that behavior before proceeding.
Do not assume that opening an update screen, confirming an offer
or interrupting a transfer is harmless. In particular, do not use
resume behavior as an experiment to identify a chip.

### Private artifact layout and manifest

Use a new capture directory under ignored `old/`, for example
`old/firmware-captures/<date>-<model>-<capture-id>/`. Keep the
first acquired bytes unchanged as `firmware.raw`; extract or
transform only separate derivatives. Filename extensions do not
establish an image format.

| Artifact | Contents | Public-document handling |
|---|---|---|
| `firmware.raw` | Original downloaded file or final OTA byte array | Do not redistribute automatically |
| `metadata.private.json` | Full API objects, URL, headers, signing fields | Keep private; URLs may contain credentials |
| `manifest.private.json` | Identity, provenance, digests, byte counts | Publish only a deliberately sanitized summary |
| `events.private.jsonl` | Timestamped hook events and capture origin | Remove identifiers, tokens and payload secrets before sharing |
| HCI/app logs | Supporting transfer records | Assume private until reviewed |
| `derived/` | Unpacked sections and later analysis | Record parent SHA-256 and transformation |

Suggested manifest shape; `null` means not yet observed, not
absent or unsupported:

```json
{
  "schema_version": 1,
  "captured_at_utc": null,
  "device": {
    "model": null,
    "tuya_product_id": null,
    "current_firmware_version": null,
    "target_firmware_version": null,
    "region": null
  },
  "environment": {
    "app_package": "com.olight.omall",
    "app_version": null,
    "apk_sha256": null,
    "phone_model": null,
    "android_version": null,
    "instrumentation_version": null
  },
  "capture": {
    "origin": null,
    "runtime_class_and_method": null,
    "transfer_path": null,
    "channel_type": null,
    "diff_ota": null,
    "metadata_file": "metadata.private.json",
    "raw_file": "firmware.raw"
  },
  "integrity": {
    "observed_bytes": null,
    "metadata_bytes": null,
    "sha256": null,
    "computed_md5": null,
    "metadata_md5": null,
    "size_matches": null,
    "md5_matches": null,
    "signature_verification": "not performed"
  }
}
```

Record SHA-256 and MD5 on the acquired bytes, then compare size
and MD5 with the applicable metadata before any unpacking. If a
staged file and final byte array differ, preserve both and record
the transformation boundary instead of choosing one silently.
MD5 agreement checks consistency with the metadata; it does not
independently establish authenticity. A `sign` or `hmac` field
being present is not proof that we verified it.

### Questions to carry into analysis

| Priority | Question | Required boundary |
|---|---|---|
| 1 | What exact model/release and package type were acquired? | Establish identity, full/differential status and integrity first |
| 2 | What container, architecture and SDK clues exist? | Match headers/code to evidence; do not infer a chip from UUIDs alone |
| 3 | Where are vendor DP and preset handlers? | Distinguish reachable product code from generic SDK code |
| 4 | How do mode, color, brightness and effect timing interact? | Static interpretation still needs controlled behavioral corroboration |
| 5 | Is there any pixel interface or secondary-element mapping? | Absence in one image/search is bounded negative evidence |
| 6 | What controls persistence and OTA validation? | Do not turn analysis into unplanned flashing or fault injection |

An original-Sphere result belongs to that generation. Use a
separate Sphere C image or direct Sphere C observation before
promoting a finding to the Sphere C protocol reference.
