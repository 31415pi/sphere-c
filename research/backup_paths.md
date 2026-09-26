# Safe backup paths for `com.olight.omall`

## Scope and safety

This note assesses non-root, non-destructive ways to preserve or
extract installed app data from an Android 17 phone. It is based on
the local APK and local host tools. No device-facing command was run,
and no phone state was changed.

The important distinction is between:

- preserving data through Android's restore system, and
- obtaining ordinary files that can be inspected on the host.

Android supports the former much more readily than the latter for a
non-debuggable app.

## Local evidence

APK inspected:

- path: `apk/olight-3.2.5.apk`
- size: 227,974,212 bytes
- SHA-256:
  `1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9`

The binary manifest says:

- package: `com.olight.omall`
- version: `3.2.5` (`versionCode=239`)
- minimum SDK: 24
- target and compile SDK: 36
- `android:allowBackup="true"`
- `android:debuggable` is not set; the installed package is also
  reported as non-debuggable
- `android:fullBackupContent="@xml/appsflyer_backup_rules"`
- `android:dataExtractionRules=`
  `"@xml/appsflyer_data_extraction_rules"`
- application class: `com.Proxy.ShellApplication`
- `android:requestLegacyExternalStorage="true"`

The two backup rule files decode as follows.

The older full-backup rules exclude only:

- shared preference `appsflyer-data`
- shared preference `appsflyer-purchase-data`
- database `afpurchases.db`

The Android 12-and-newer extraction rules have both
`cloud-backup` and `device-transfer` sections. Each excludes the
same three AppsFlyer items and has no `include` entries.

That is strong static evidence that the developer intended Android
backup and device transfer to include the normal default data set,
apart from those three items. It does **not** mean that every byte is
eligible. Android always omits categories such as cache, code, and
`noBackupFilesDir`; transport quota and policy also apply. App-level
encryption can make restored files unusable if keys are not restored.

The APK declares 26 content providers. The ordinary file providers
are all non-exported:

- `com.olight.omall.fileprovider`
- `com.olight.omall.cn.sharesdk.ShareSDKFileProvider`
- `com.olight.omall.luckProvider`
- `com.olight.omall.utilcode.fileprovider`
- `com.olight.omall.provider`

They allow temporary URI grants, but an ADB shell does not possess a
grant merely because `grantUriPermissions` is true.

Five providers are exported and have no manifest permission:

- Facebook attachment provider
- three Forter SDK providers
- `qrom.component.wup.runInfo.QRomWupProvider`

Their authorities are:

- `com.facebook.app.FacebookContentProvider351325629718248`
- `com.forter.mobile.com.olight.omall.ftrhxcontentprovider`
- `com.forter.mobile.com.olight.omall.appstartcontentprovider`
- `com.forter.mobile.com.olight.omall.`
  `deviceaccountvisibilityprovider`
- `com.olight.omall.wup.QRomProvider`

The Facebook class and strings identify its normal SDK attachment
provider, not a general filesystem provider. The Forter and QRom
names indicate SDK coordination or telemetry providers. None is an
exported `FileProvider`, and none is static evidence of access to the
app's databases or preferences. A packed application and third-party
SDK obfuscation limit what names alone can prove.

The base APK declares required ABI and density split types. A copy of
this one APK is therefore not necessarily a complete reinstall set.

## Route assessment

### 1. Device-to-device transfer: best preservation route

**Plausibility: high for preservation; low for host extraction.**

The explicit `device-transfer` rules and `allowBackup=true` make the
platform/OEM migration flow the strongest supported route. It should
be attempted while the source phone and app still work.

Limits:

- it normally runs during destination-device setup;
- it may require a reset of the *destination*, though not the source;
- vendor policy, app signing, user/account state, and key storage can
  still prevent a complete restore;
- restored data remains in the destination app sandbox;
- it does not produce a readable archive for ADB.

Thus this is the first choice if the goal is continuity or disaster
recovery. It is not a forensic extraction method.

### 2. Android cloud/Auto Backup: good secondary preservation

**Plausibility: medium to high for restore; none for direct download.**

The `cloud-backup` rules opt into the default data set, with only the
three AppsFlyer exclusions. A current Android backup may therefore
preserve app data for restoration on a compatible install.

Uncertainties include whether backup is enabled, whether the chosen
transport is healthy, when the last backup ran, quota, idle/charging
conditions, and app-side encryption. Google/OEM backup storage is not
exposed as a user-downloadable archive or by an ADB pull operation.

Read-only inspection of backup settings is reasonable later. Forcing
a backup is a device state change and should be separately approved.

### 3. App-supported export or account resync

**Plausibility: medium; potentially the best readable result.**

An in-app export, share flow, web account, or vendor privacy-data
request can produce data that Android deliberately keeps out of ADB.
This should be checked before experiments. It may cover server-side
orders, account data, and paired-device state better than a sandbox
copy. It might not preserve local-only settings or cryptographic
material.

UI capture through screenshots or accessibility can preserve a small
amount of user-visible data, but it is a manual fallback rather than
a private-data extraction method.

### 4. Shared or app-external storage

**Plausibility: medium for exported media; low for core state.**

Photos, downloads, labels, or explicit exports may already exist in
shared storage and can often be copied without root. App-specific
external storage is subject to current Android scoped-storage and ADB
shell policy. `requestLegacyExternalStorage=true` is ineffective as a
general bypass for a target-SDK-36 app.

This route cannot reach `/data/user/0/com.olight.omall`. It is still
worth inventorying public media and files that the app intentionally
exported, using read-only operations after authorization.

### 5. Exported content providers

**Plausibility: low and narrow.**

The shell `content` utility can query an exported provider when its
URI contract permits it. Static inspection does not reveal a provider
that intentionally exposes Olight private state. The five exported
providers appear to be narrow third-party SDK endpoints.

Blind URI fuzzing is not recommended. A provider query can start the
app and may have implementation side effects even if the command is
nominally a read. If pursued, first derive exact, documented URI paths
and MIME behavior from the provider implementation. Then issue only
a minimal query/open against those known paths. Never use `insert`,
`update`, `delete`, or `call` during a backup attempt.

The non-exported file-provider authorities are not candidates unless
the app itself grants the shell a URI, which is unlikely.

### 6. Legacy `adb backup`

**Plausibility on Android 17: very low.**

The locally installed host ADB is Debian's 34.0.5 build. Its binary
still contains the `backup` and `restore` clients and explicitly calls
both deprecated. Host-side command presence does not establish that
an Android 17 device still offers a working full-backup service.

Legacy `adb backup -noapk com.olight.omall` historically depended on
`allowBackup`, a confirmation UI, device policy, and the framework's
backup implementation. Modern Android releases have deprecated or
removed practical support for this path. The Android 12+ extraction
rules describe cloud backup and device transfer, not a promise of an
ADB-readable archive.

It is reasonable to check read-only command availability later, but
an actual backup attempt may prompt on the phone and should not be the
primary plan. A zero-byte or header-only `.ab` must not be mistaken
for success. If a controlled test is approved, validate the archive
size and contents before relying on it.

### 7. `bmgr`

**Plausibility for triggering/diagnosis: medium; for extraction: none.**

`bmgr` talks to the device backup manager. Read-only subcommands can
report whether backup is enabled and which transport is selected.
Other subcommands can schedule or run a backup or select a transport.
Those mutate device or remote backup state and are outside a purely
read-only inspection.

Most importantly, `bmgr` does not stream an app's private backup set
to the ADB caller. A transport owns the resulting data. Even a local
transport normally stores it in system-only device storage, not in a
shell-readable directory. Therefore `bmgr` can help establish backup
health but does not solve host extraction without root or a custom,
privileged transport.

### 8. Direct ADB sandbox access and adjacent tools

**Plausibility: effectively none for private files.**

- `run-as com.olight.omall` is blocked for a non-debuggable app.
- `adb pull /data/user/0/com.olight.omall` is blocked by Linux/SELinux
  permissions.
- copying the installed APK or its splits copies program code, not
  the installed account, databases, or preferences.
- `bugreport`, `dumpsys`, and `logcat` can expose metadata and limited
  diagnostics, not an authorized sandbox backup.
- package downgrade/reinstall, clearing data, repacking, signature
  substitution, restore tricks, or exploiting an exported component
  are unsafe and should not be used on the source phone.

## Ranked safe plan

1. **Preserve the current source state.** Do not uninstall, clear app
   data, downgrade, repack, or change accounts. Keep the phone
   unlocked, powered, and usable. Record app and OS versions.

2. **Check the app itself.** Look for export, sync, account, device
   sharing, diagnostic export, and privacy-data-request functions.
   Export only to a new destination and verify the files.

3. **Use device-to-device migration for continuity.** Prefer the
   official Android/OEM setup transfer to a spare destination. Keep
   the source untouched until login, paired devices, and important
   records are verified on the destination.

4. **Verify cloud-backup health.** With separate approval, inspect
   the system Backup settings and read-only backup-manager status.
   Confirm the account, selected transport, and recent backup state.
   Do not change transport or force a backup during inspection.

5. **Inventory public/shared files.** With device authorization, copy
   only media, Downloads, and clearly exported Olight files. Hash the
   copies and leave source files in place.

6. **Treat legacy ADB backup as a controlled last-chance test.** Only
   after the supported preservation routes exist, check whether the
   Android 17 build advertises the command. If the user approves an
   attempt, write to a new host file, accept only the expected phone
   prompt, and verify that useful records are present.

7. **Probe providers only with a derived URI contract.** First
   decompile the exact installed split set or consult SDK docs. Query
   only a known exported, read-only URI. Expect SDK metadata or cached
   attachments, not the private app database.

8. **Escalate to the vendor before technical circumvention.** If the
   required information is server-held, an account export or privacy
   request is safer and more complete. If it is local-only and absent
   from platform backup, there is no general non-root ADB route for a
   non-debuggable Android 17 app.

## What to inspect later, without extracting data

After explicit permission to contact the phone, start with read-only
facts:

- Android build and patch level
- installed APK and split paths
- package flags and exact signing/version information
- Backup settings, enabled state, transport, and last-known status
- public/shared files attributed to the app
- provider declarations as installed, to catch manifest differences

Do not start with `adb backup`, provider probing, a forced `bmgr`
operation, or any install/uninstall action. The first session should
only confirm which supported preservation path is actually available.

## Bottom line

`ALLOW_BACKUP` is meaningful here: the app supplies modern cloud and
device-transfer rules that exclude only three AppsFlyer records. The
best safe avenue is official device-to-device transfer, backed by a
verified Android cloud backup and any app/vendor export.

It does not create a general ADB read permission. Neither `bmgr` nor
modern cloud backup yields a host-readable archive, legacy `adb
backup` is not dependable on Android 17, and the exported providers
do not presently look like gateways to Olight private data.
