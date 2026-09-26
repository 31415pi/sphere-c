# Olight 3.2.5 APK: mesh crypto and key exposure

## Scope and method

This is a static inspection of `apk/olight-3.2.5.apk` only. No
APK modification, installation, ADB, account login, or live traffic
capture was used.

APK facts:

- Package: `com.olight.omall`
- Version: 3.2.5 (`versionCode` 239)
- Minimum/target SDK: 24/36
- SHA-256:
  `1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9`
- The base APK contains 25 DEX files and no `.so` files. Thus, the
  inspected Bluetooth Mesh implementation is Java/Kotlin bytecode,
  not a bundled native library. Native code may exist in an
  unprovided split APK, but none exists in this artifact.

Class and method names below are DEX names. Some Thingclips SDK
implementation names are obfuscated, but public model and API names
are mostly intact.

## High-confidence findings

### The SDK implements standard Bluetooth SIG Mesh crypto

`com.thingclips.sdk.sigmesh.util.SecureUtils` contains recognizable
Bluetooth Mesh primitives and labels:

- AES-CMAC using the bundled Thingclips Bouncy Castle `CMac` and
  `AESEngine` classes.
- AES-CCM authenticated encryption/decryption.
- AES block encryption/decryption and AES-CBC/NoPadding elsewhere in
  the SIG Mesh utility layer.
- Secure random 8-byte and 16-byte generation.
- SIG Mesh derivation labels `smk2`, `smk3`, `smk4`, `id64`, `id6`,
  `id128`, `nkik`, and `nkbk`.
- Provisioning labels `prck`, `prsk`, `prsn`, and `prdk`.

The K2-shaped routine uses `smk2`, AES-CMAC, three chained CMAC
steps with counters 1, 2, and 3, masks the last byte with `0x7f`,
and returns a `K2Output`. Other routines derive network ID,
identity/beacon material, and private beacon material using the
labels above. This is strong evidence of normal SIG Mesh key
derivation rather than a proprietary substitute.

### Key models and serialized field names

`com.thingclips.sdk.sigmesh.bean.NetworkKey` has these fields:

- `meshUuid`
- `keyIndex`
- `name`
- `key` (byte array)
- `phase`
- `minSecurity`
- `oldKey` (byte array)
- `timestamp`

It is Parcelable and writes both `key` and `oldKey` in clear byte
array form to a Parcel.

`com.thingclips.sdk.sigmesh.transport.ApplicationKey` has:

- `meshUuid`
- `keyIndex`
- `name`
- `boundNetKeyIndex`
- `key` (byte array)
- `oldKey` (byte array)
- `id`

It likewise writes both key byte arrays to a Parcel.

The cloud/cache-facing model uses less obvious aliases:

- `SigMeshBean.meshkey` is returned by `getMeshKey()`,
  `getMeshkey()`, and `getAppkey()`.
- `SigMeshBean.getNetWorkkey()` returns the inherited
  `BlueMeshBean.name` field.
- `SigMeshBean.ivIndex` contains the IV Index.
- Other inherited fields include `meshId`, `localKey`, `password`,
  `code`, `pv`, and `share`.
- `BeaconMeshBean` separately has `appKey`, `meshId`, and `id`.

Therefore, an exported JSON object need not literally say `netKey`:
for SIG Mesh, `name` can contain the NetKey and `meshkey` can contain
the AppKey. This aliasing is important when examining cache or API
responses.

### Validation is mostly structural

The clearest NetKey validation is in obfuscated
`com.thingclips.sdk.bluetooth.bbqqpqb`:

1. Convert the supplied hex string to bytes.
2. Reject it with `"Network key is error"` unless the result is
   exactly 16 bytes.
3. Store it on `UnprovisionedMeshNode`, with key index zero.

Key indices are serialized as 12-bit values in the same utility
layer (`value & 0x0fff`), matching SIG Mesh semantics. Status parsing
also includes `INVALID_NETKEY_INDEX`, `INVALID_APPKEY_INDEX`, and
`Invalid NetKey Index` strings.

No comparable cryptographic-strength validation of an AppKey was
found. Call sites generally convert the AppKey hex string to bytes
and use it. The model setters themselves accept arbitrary strings or
byte arrays. Thus, the main visible checks are length/format and
protocol status, not proof that a supplied key is correct before use.

## Persistence and cloud recovery

### Local cache

`SigMeshProperty` obtains entity type `4` from
`IThingCachePlugin`/`ISmartCacheManager` and casts its `content` to a
`SigMeshBean`. `getMeshKey()` then directly exposes that object's
`meshkey` value.

The included default cache implementation stores Fastjson-serialized
`CacheObj` data in a `DiskLruCache`. Its default directory is:

`<external-cache-dir>/CACHE_MANAGER`

Both cache keys and JSON values are encrypted with AES-CTR. The key
comes from `SecuredStore.securityUserKey("CACHE_MANAGER")`; the IV is
deterministically derived from that key. This is obfuscation at rest,
not a plaintext JSON cache. The key's exact secure-store derivation
was not established in this review.

Android normally treats external cache as app-scoped, and it is not
a dependable public-storage recovery path. No exported provider was
found that grants arbitrary access to it. Several very broad
FileProvider path definitions exist, including entire external and
external-cache trees, but their providers are declared
`exported="false"` and require an explicitly issued URI grant.

### Cloud/API path

The SDK contains explicit cloud list APIs and callbacks:

- `API_SIG_MESH_LIST_GET`
- `API_MESH_LIST_GET`
- `IRequestSigMeshListCallback`
- `requestSigMeshList`
- `getHomeSigMeshList`
- `getMeshDeviceListFromCloud`

The returned/cached object is `SigMeshBean`, whose `name` and
`meshkey` fields expose the NetKey and AppKey through the aliases
described above. The app therefore appears capable of recovering
mesh credentials from the authenticated Thingclips cloud and
repopulating local state after login. Static analysis did not
identify a public, unauthenticated endpoint, nor enough request
signing detail to reproduce the API outside the app. Any practical
cloud recovery requires valid account/session credentials and
authorization for the home.

## Plaintext logging and export surfaces

### Direct key logging

Multiple production SDK methods construct plaintext strings
containing live mesh keys and pass them to
`com.thingclips.smart.android.common.utils.L`:

- `BleMeshController.updateNetKey(...)` logs device ID and the full
  replacement `netKey` at info level.
- `BleMeshControllerManager.updateNetKey(...)` does the same.
- `BleMeshManager.updateBleMeshBean(...)` logs `meshId`, full
  `appkey`, and full `netKey` at info level.
- The SIG Mesh activator implementation (obfuscated class
  `pdpqbpq`) logs full `networkKey` and `appKey` at debug level in its
  constructor.
- The proxy wake-up builder (obfuscated class `dqbpdbq`) logs full
  `networkKey` and `appKey` alongside plaintext input before
  encryption.
- BLE pairing code logs the full BLE `netKey` at debug level.

These are actual concatenations of key values, not merely harmless
log labels.

### Whether a normal user can obtain those logs

The logging facade only emits when its global `isLogOn` flag is true.
When enabled, it sends messages through Datadog's `TrackLog` and can
also forward them to an installed `ILogInterception`. The flag is
controlled by `ThingSmartSdk.setDebugMode(boolean)` and by the
application initializer's debug argument. The manifest has no
`android:debuggable="true"`; this is a release-targeted APK.

The APK nevertheless contains an end-user log-upload UI
(`log_upload_btn`, `log_upload_cell`, `log_upload_content`, and
`log_upload_success`), a Thingclips log-uploader pipeline, and local
log-file support (`enableLocalLogFileRecord`). Thus, if the
application enables Thingclips logging in this build, the plaintext
keys above can enter locally collected/support-uploaded logs.

What static analysis does **not** establish is that the release app
actually enables `isLogOn` during ordinary use, or that its upload UI
lets the user preview/download the raw file. Modern Android also
prevents an ordinary third-party app or shell-less user from reading
another app's logcat. Therefore this is a serious latent disclosure
and support-log risk, but not a proven one-click, non-root key export.

No JavaScript bridge method in the bundled `TUNIMeshManager` exposes
keys: its public methods return device IDs or mesh DP data only.

### Backup/device transfer

The manifest sets `allowBackup="true"`. Its backup and extraction
rules exclude only AppsFlyer preferences and `afpurchases.db`; they
do not explicitly exclude Thingclips cache, preferences, or
databases. This makes cloud backup/device-to-device transfer a
possible non-root migration path for any credential data stored in
normal backup-eligible app data.

However, the identified mesh `DiskLruCache` defaults to an external
cache directory, which is not a reliable backup target, and its JSON
is AES-CTR encrypted. Target SDK 36 also makes legacy `adb backup`
an unsuitable assumption. Backup is therefore a plausible indirect
recovery mechanism, not a demonstrated raw-key extraction method.

## Bottom line

The APK contains a complete, recognizable SIG Mesh implementation
and retains NetKey/AppKey material in Java model objects. It can
apparently retrieve `SigMeshBean` objects from authenticated
Thingclips cloud APIs. Local default cache data is encrypted, and no
unprivileged exported content provider or JavaScript API was found
that directly returns mesh keys.

The strongest finding is avoidable plaintext key logging at several
normal mesh operations. If logging is enabled, support-log collection
may capture both NetKey and AppKey. Static inspection alone cannot
prove the release runtime flag or a user-readable export route, so a
no-root extraction claim should be phrased as conditional. The most
credible no-root routes are authenticated cloud re-fetch or an
in-app support-log workflow when SDK logging is enabled.
