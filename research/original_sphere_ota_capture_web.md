# Public-web search for original Olight Sphere OTA artifacts

Research date: 2026-09-26

Scope: the original Olight Sphere with the proprietary magnetic
charging cable, not the later USB-C Sphere C. The search covered
firmware images, OTA packages and URLs, checksums, packet captures,
flash dumps, reverse-engineering repositories, and reports about
firmware 3.5 and contemporaneous app releases.

## Bottom line

I found credible public evidence that the original magnetic-charge
Sphere received firmware updates, including a version identified by
users as **3.5**. I did **not** find any publicly shared bytes or
recoverable artifact for that firmware:

- no Sphere `.bin`, `.hex`, `.ota`, `.fw`, `.img`, or firmware ZIP;
- no Sphere OTA download URL;
- no checksum or signature belonging to Sphere firmware;
- no packet capture containing a Sphere firmware transfer;
- no raw flash dump;
- no repository implementing or documenting Sphere OTA;
- no issue, gist, or forum attachment that contains the above.

The best public technical work concerns ordinary control and Tuya /
Bluetooth SIG Mesh provisioning. It does not capture the OTA payload.
The strongest OTA-specific public source is an older official Olight
manual for the **Obulb Pro**, which says Android users could download
a firmware package from Olight's website and select it in the app.
That is useful evidence about Olight's historical update workflow,
but it is neither a Sphere package nor proof that Sphere 3.5 was ever
published as a static website download.

## Evidence grading

### Actual firmware artifacts

**None found.** Searches of general web results, GitHub and other code
host results, official Olight pages, public archives, Reddit, Home
Assistant discussions, and firmware-related filename/extension
queries did not surface captured Sphere firmware bytes or metadata.

This is an evidence-of-absence statement for the public sources
searched, not proof that no private capture exists.

### Primary evidence that Olight supports device OTA

The official Olight Obulb Pro app manual says:

- OTA was available only on Android in that older app;
- compatible-device firmware could be updated over Bluetooth;
- the user was to download a firmware package from
  `olightworld.com`, unzip it, copy it to the phone, and select it on
  the device-info page.

Source: [official Olight Obulb Pro app manual](https://ca.olight.com/file/manual-obulb-pro-app.pdf),
especially the text labeled “OTA UPDATE.”

This manual is for Obulb Pro, not Sphere. No Sphere filename, version,
URL, checksum, or payload appears in it. It is therefore evidence of
a historical Olight OTA mechanism only.

The official original-Sphere manual identifies the magnetic-charge
model and its app workflow. It shows the USB magnetic charging cable,
30-minute sleep behavior, pairing, group control, smart scenes, and
automation. It does not expose an OTA URL, package format, checksum,
or firmware version.

Source: [official original Sphere manual](https://cdn.olightstore.com/file/sphere-user-manual.pdf).

The FCC filing confirms the original Sphere's BLE radio and product
identity (`FCC ID 2A44S-SPHERE`), but the published filing is test and
compliance material, not firmware. One FCC mirror reports SHA-256
`9e67b14e5459e125c8c8e59cafec318c01282e34ea96571cf6bb1c94689be4f1`
for the **BLE test-report PDF**. That hash must not be mistaken for a
firmware checksum.

Source: [FCC Sphere BLE test report](https://fccid.io/2A44S-SPHERE/Test-Report/BLE-Test-Report-1-7603977).

The same FCC filing contains public original-Sphere internal photos:

- document ID `7603966`;
- 5 pages and 2,253,465 bytes;
- SHA-256
  `c8661b501fd27ce3a841372ea35014c8cf1a58f40928024f817fc420f81ffce7`.

Source: [FCC Sphere internal photos](https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966).

This is a useful primary hardware artifact, but it contains no flash
dump or OTA payload. The FCC listing also identifies the original and
Sphere C as separate filings (`2A44S-SPHERE` and `2A44S-SPHEREC`).

A mirrored product manual identifies original-Sphere model
`O-OBULB2-300000` and the MCC 1A magnetic charger. Those are useful
search terms and help exclude Sphere C results, but the manual has no
firmware package.

Source: [original-Sphere product manual mirror](https://manuals.plus/asin/B0CW1QN3DF).

## Firmware 3.5 reports

The clearest version-specific report is a 19 September 2025 Reddit
thread whose author explicitly limited the question to the
“magnetic charger not Sphere C” model. Multiple owners said they had
installed 3.5. Reported behavior varied:

- some could turn a Sphere on from the app after it had been off for
  a long time;
- two Spheres used as night lights reportedly woke from the app and
  restored/synchronized their last patterns;
- another owner reported two of three units behaved that way while
  the third still required a manual wake and lost its last setting;
- another person later said connectivity was still unreliable;
- one owner described the unit as buggy immediately after the update,
  then somewhat improved after about a day.

Source: [“Anyone updated Sphere to v3.5 firmware?”](https://www.reddit.com/r/Olightflashlights/comments/1nks4zd/).

This is useful corroboration that a rollout called 3.5 existed for the
original model. It contains no screenshot of update metadata, package
URL, hash, transfer trace, or firmware attachment. The behavior
reports are user observations, not release notes.

A second thread from 23–27 October 2025 says newly paired Spheres were
updated to 3.5. The detailed device-control screen then showed a
spinner, while on/off commands continued to work; the author later
said it recovered and suspected a server issue.

Source: [“Olight App: Sphere controls don't load”](https://www.reddit.com/r/Olightflashlights/comments/1oei3ln/).

Again, this confirms only the displayed version and update event. No
artifact was posted.

## Earlier firmware-update discussion

A 15 November 2024 thread announced a festive Sphere update with new
colors and other changes. One reply said the new festive patterns all
played the fireplace pattern after updating. It does not give a
firmware version or attach update material.

Source: [“Sphere has had a festive update!”](https://www.reddit.com/r/Olightflashlights/comments/1gru2p1/).

Another 2024 support thread contains an owner saying the Sphere had
just run a firmware update and gained a Christmas pattern. It likewise
has no version, URL, hash, or capture.

Source: [“Olight Sphere issues”](https://www.reddit.com/r/Olightflashlights/comments/1fxf46a/).

A separate October 2024 thread correctly notes that the products can
receive OTA updates, but only as a general capability claim.

Source: [“Olight app and spheres”](https://www.reddit.com/r/Olightflashlights/comments/1ftjhin/).

These posts should not be treated as recovered firmware. They show
that firmware delivery occurred and provide rough rollout dates only.

## App releases around firmware 3.5

Apple's first-party App Store history shows these Olight app releases
around the 3.5 reports:

| Olight app | Date |
|---|---|
| 2.6.4 | 2025-09-05 |
| 2.6.5 | 2025-09-10 |
| 2.6.6 | 2025-09-21 |
| 2.7.0 | 2025-10-01 |
| 2.7.5 | 2025-10-30 |
| 2.7.8 | 2025-11-18 |

Source: [Olight on Apple's App Store](https://apps.apple.com/us/app/olight/id6476482087).

The public change text for these versions is generic experience
optimization language. It does not mention Sphere firmware 3.5, an
OTA endpoint, a package name, or a firmware hash.

The 19 September 2025 3.5 discussion falls between app 2.6.5 and
2.6.6. This is a timeline correlation only. It does not show that the
firmware was bundled in either app release; cloud-side firmware
rollouts can be independent of app-store releases.

Third-party APK indexes expose hashes for the **Android application**.
For example, Uptodown lists Olight Android 3.1.1 as an XAPK and gives
SHA-256
`5e2476bc54e20b02be422c63f7ba52985d311092d44288ab588b4cf7c23fbcd0`.
That is an app-package hash, not a Sphere firmware hash.

Source: [Uptodown Olight Android download metadata](https://olight.en.uptodown.com/android/download).

## Public reverse engineering: useful, but no OTA capture

### Home Assistant / Tuya work

A July 2025 Home Assistant investigation provisioned an Olight Sphere
through a Tuya-compatible Bluetooth gateway. The author instrumented
`com.olight.omall` with Frida and recovered ordinary Tuya device
identifiers and gateway keys, then used a LocalTuya fork. The post
shares a device-property template and control procedure.

Source: [Home Assistant Sphere integration post](https://community.home-assistant.io/t/integrating-olight-spheres-in-ha/907929/2).

Important limitations:

- the post's purchased product is described as Sphere C;
- the extracted values are control/provisioning data, not OTA
  metadata;
- no firmware URL, version response, checksum, OTA packet capture, or
  firmware bytes are shared;
- the linked LocalTuya repository is generic and contains no Olight
  Sphere firmware.

The public `tuya-local` repository has an Olight Sphere C profile with
Tuya product ID `d0dpga0h` and data-point mappings. It has no device
firmware or OTA capture and applies to Sphere C, not necessarily the
original magnetic model.

Source: [tuya-local Sphere C device profile](https://github.com/make-all/tuya-local/blob/main/custom_components/tuya_local/devices/olight_sphere_light.yaml).

### Proposed sniffing, not completed capture

In a December 2024 Reddit discussion, one participant said the newer
Olight products use Bluetooth SIG Mesh / Tuya-like behavior and wrote
that they might obtain application keys and sniff traffic later with
Nordic tools and Wireshark. No follow-up capture, keys, repository, or
OTA bytes are attached.

Source: [smart-home compatibility discussion](https://www.reddit.com/r/Olightflashlights/comments/1hd8vrj/).

This is a plan to reverse engineer, not evidence that the work was
completed.

## Search details and negative results

Queries combined `Olight`, `Sphere`, the explicit original-model
phrases `magnetic charger` and `not Sphere C`, and terms including:

```text
firmware, 3.5, v3.5, OTA, update, bin, hex, img, fw, zip,
download URL, md5, sha256, dump, flash dump, packet capture,
pcap, Wireshark, nRF Connect, Bluetooth mesh, BLE protocol,
reverse engineering, GitHub, GitLab, Gitee, gist
```

The following plausible-looking hits were rejected:

- manuals and FCC files: documentation, not firmware;
- app APK/XAPK hashes: hashes of the phone app, not device firmware;
- `tuya-local` and LocalTuya code: control mappings and integrations,
  not firmware or OTA traffic;
- generic BLE OTA repositories: unrelated projects that happen to
  match the search terms;
- Sphere C material: a separate USB-C model unless a source explicitly
  identifies the magnetic model;
- Reddit statements such as “ran a firmware update”: discussion of an
  update event, not captured bytes or metadata;
- the FCC report's SHA-256: hash of a PDF test report;
- archived Olight pages and extension searches: no publicly indexed
  Sphere firmware ZIP or binary was found.

Code-host checks were also negative:

- GitHub repository search for exact `"olight sphere"`: no repository;
- GitHub commit searches combining the product name with `firmware`,
  `OTA`, `btsnoop`, and `pcap`: no captured artifact;
- searches for `O-OBULB2-300000` and Amazon identifier
  `B0CW1QN3DF`: no firmware-bearing code result;
- GitLab public project search for `olight sphere`: no project;
- Sourcegraph global exact-string search for `"Olight Sphere"`: no
  code match.

One subtle false positive deserves special mention. The public
`tuya-local` pull request links an original-Sphere product page, but
the committed profile itself explicitly says model `Sphere C` and
product ID `d0dpga0h`. The implementation must therefore not be used
as original-Sphere firmware metadata.

Source: [tuya-local pull request 4149](https://github.com/make-all/tuya-local/pull/4149).

## Practical acquisition implication

The public web currently supplies a target version number and rollout
window, but not a reusable artifact. Recovering original-Sphere 3.5
will likely require a fresh authenticated update check or a capture
from a device/app installation that has not already consumed the
update. The most valuable capture would preserve, before download:

1. the complete update-metadata response;
2. the resolved firmware URL and HTTP headers;
3. any MD5/SHA/signature/version/size fields;
4. the unmodified downloaded file;
5. a BLE packet capture of the transfer;
6. the exact app version, device model, current firmware, region, and
   timestamp.

Nothing in the public sources above supplies those items today.
