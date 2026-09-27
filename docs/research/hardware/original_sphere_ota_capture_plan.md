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
