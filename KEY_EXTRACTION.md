# Mesh key extraction status

## Result on the connected phone

The phone is a Pixel 8 Pro running Android 17. The installed app is
Olight 3.2.5, package `com.olight.omall`, target SDK 36.

The app is not debuggable and the phone does not provide root. These
Android protections block:

- `run-as` access to the private app directory
- process heap dumping
- direct reads of `/data/user/0/com.olight.omall`
- access through the app's non-exported providers

The installed APK and the app's externally readable files were copied
without modifying the source phone. The external files include Olight
logs and the Sphere C mini-app for product `d0dpga0h`.

The external files contained 21 distinct values shaped like 128-bit
hex keys. None produces both capture Network ID
`1d497fd760450169` and NID `0x30`, so none is this mesh's NetKey.

## Exact private storage target

Static APK analysis found the durable SIG Mesh record in:

```text
/data/user/0/com.olight.omall/files/thingmmkv/
```

The encrypted MMKV map is named:

```text
preferences_global_key<Thingclips-user-UID>
```

Its entry `s_home_data_sigmesh_list<homeId>` contains Fastjson
`SigMeshBean` objects. In those objects:

```text
name       = SIG Mesh NetKey
meshkey    = SIG Mesh AppKey
ivIndex    = SIG Mesh IV Index
meshId     = Thingclips mesh identifier
```

The map encryption hierarchy also depends on:

```text
/data/user/0/com.olight.omall/shared_prefs/global_preference.xml
/data/user/0/com.olight.omall/shared_prefs/SPS_file.xml
Android Keystore state
```

An explicit SIG DeviceKey was found in live provisioning objects but
not in the durable home snapshot.

## Cloud recovery path

The signed-in app retrieves the same key-bearing record with:

```text
API:      m.life.my.group.mesh.list
version:  3.1
body:     {"gid": <homeId>}
session:  required
```

Pulling down on the app's Devices screen invokes the home refresh
that includes this request. The result's `sigmesh` array carries the
NetKey and AppKey under `name` and `meshkey`.

The request uses a live session and native, app-bound request signing.
It cannot be reproduced reliably from a copied URL and token.

## No-root routes assessed

- Modern Android backup rules include ordinary private app data, but
  no host-readable ADB backup command is available on this phone.
- Cloud backup and device transfer preserve data but do not expose a
  readable archive.
- The app's Log Upload screen submits logs to Olight and offers no
  local view, save, or share operation.
- The release app trusts only system CAs. Its Thingclips HTTP client
  also implements certificate pinning, so a user-installed proxy CA
  on this stock phone is insufficient.
- No exported provider, JavaScript bridge, or debug activity exposes
  the mesh record.

## Recommended next experiment

Use a disposable rooted Android device or rooted emulator. Install the
unmodified Olight app, sign in to the authorized account, select the
correct home, and pull down on Devices to refresh.

On that test system, hook these Java methods while refreshing:

```text
com.thingclips.smart.sdk.bean.SigMeshBean.getNetWorkkey()
com.thingclips.smart.sdk.bean.SigMeshBean.getAppkey()
```

Alternatively, use root to acquire `files/thingmmkv`, the associated
`.crc` files, preferences, and live Keystore-backed decryption state.
The recovered NetKey must validate against the capture's Network ID
and NID before it is accepted.

Do not unlock the bootloader of the current Pixel to obtain root. Pixel
bootloader unlocking wipes user data and would destroy the evidence we
are trying to acquire.

## Reproduction

Validate plaintext 128-bit candidates in copied phone artifacts with:

```sh
python3 scan_phone_artifacts.py
```

Detailed static findings are in:

- `research/apk_storage.md`
- `research/apk_crypto.md`
- `research/backup_paths.md`
- `research/tls_intercept.md`
