# Olight 3.2.5: Thingclips mesh storage

## Scope and method

This is static analysis of `apk/olight-3.2.5.apk` only. The APK
was not modified. The relevant DEX files were inspected with local
string extraction and baksmali 2.2.1. The Android package name is
`com.olight.omall`.

The main result is that this SDK build does not use a dedicated
Bluetooth Mesh SQLite database. Its durable home snapshot is JSON
inside an encrypted, per-user MMKV file. A second cache layer holds
the same objects only in memory. Provisioning-only node state,
including the explicit Bluetooth Mesh DeviceKey byte array, appears
to remain in memory rather than in the durable home snapshot.

## Durable mesh and node cache

### Physical files

`com.thingclips.smart.mmkv.MMKV.initialize(Context)` selects:

```text
/data/user/0/com.olight.omall/files/thingmmkv/
```

(`/data/data/com.olight.omall/files/thingmmkv/` is the usual alias.)
The relevant map ID is:

```text
preferences_global_key<Thingclips-user-UID>
```

MMKV normally represents that map with a data file and matching
`.crc` file bearing the map ID. Both must be acquired together.

`com.thingclips.smart.android.base.utils.PreferencesUtil` builds the
map ID by appending `IBaseUser.getUid()` to
`preferences_global_key`. It constructs `MMKVManager`, whose
three-argument constructor opens MMKV in multi-process mode with an
encryption key obtained from:

```text
SecuredStore.securityUserKey(mapId)
```

Therefore a copied map is not expected to decode as plain MMKV
without its key hierarchy and, potentially, the original Android
Keystore state.

### MMKV entries

The home ID is appended directly, with no delimiter:

```text
s_home_data_sigmesh_list<homeId>
s_home_data_blemesh_list<homeId>
s_home_data_devlist<homeId>
s_home_data_grouplist<homeId>
s_home_data_product_list<homeId>
s_home_data_beacon_list<homeId>
s_home_data_ble_mesh_v2_list<homeId>
s_home_detail<homeId>
```

`com.thingclips.sdk.home.OooOo00` declares these key prefixes.
`com.thingclips.sdk.home.OooO00o$OooO00o` serializes list objects
with Alibaba Fastjson and writes them through
`PreferencesUtil.set(key, json)`. `com.thingclips.sdk.home.OooO0O0`
reads the same keys and calls `JSON.parseArray` with the relevant
bean class.

The two primary forensic entries are:

* `s_home_data_sigmesh_list<homeId>`: array of `SigMeshBean`.
* `s_home_data_devlist<homeId>`: array of `DeviceRespBean`.

The legacy/non-SIG list is
`s_home_data_blemesh_list<homeId>`, an array of `BlueMeshBean`.

### NetKey and AppKey schema

`com.thingclips.smart.sdk.bean.SigMeshBean` extends
`BlueMeshBean`. Its storage mapping is unusual but explicit:

* `getNetWorkkey()` returns inherited `BlueMeshBean.name`.
* `setNetWorkkey(value)` writes inherited `name`.
* `getAppkey()` returns `SigMeshBean.meshkey`.
* `setAppkey(value)` writes `meshkey`.

Thus, in each JSON object in
`s_home_data_sigmesh_list<homeId>`:

```text
name       = SIG Mesh NetKey
meshkey    = SIG Mesh AppKey
ivIndex    = SIG Mesh IV Index
meshId     = Thingclips mesh identifier
localKey   = Thingclips mesh local key
password   = legacy mesh password/credential
pv         = protocol version
code       = mesh code
```

Other `BlueMeshBean` fields serialized with the object include
`share`, `tempShare`, `startTime`, `endTime`, and `resptime`.
Fastjson may also emit getter-derived aliases such as `appkey`,
`meshKey`, or `netWorkkey`; extraction should accept both the raw
field names and these accessor names. The definitive backing-field
mapping is `name` for NetKey and `meshkey` for AppKey.

The remote source for this list is also visible. Class
`com.thingclips.sdk.home.oo000o` calls API
`m.life.my.group.mesh.list`, version `3.1`, with `gid`, parses a
`com.thingclips.sdk.home.bean.MeshListResponseBean`, and consumes
its `sigmesh` list. This response is what the home cache persists.

### Device and node metadata schema

`s_home_data_devlist<homeId>` contains Fastjson serialization of
`com.thingclips.smart.interior.device.bean.DeviceRespBean`.
High-value fields include:

```text
devId, devKey, localKey, secKey, uuid, mac, name, ownerId,
productId, productVer, activeTime, protocolAttribute, devAttribute,
baseAttribute, communication, deviceTopo, meta, skills, resptime
```

Its nested
`DeviceRespBean$DeviceTopoMoudle` contains:

```text
meshId, nodeId, parentDevId
```

Its nested `DeviceRespBean$CommunicationModule` contains:

```text
communicationModes, communicationNode, localCommunicationNode,
connectionStatus, dataModel, localDataModel, localNodeId,
mqttTopicAttr
```

Because `DeviceRespBean` also exposes top-level `getMeshId()` and
`getNodeId()` methods which delegate to `deviceTopo`, a Fastjson
snapshot may contain top-level copies as well. Extractors should
inspect both the top level and `deviceTopo`.

The SDK also uses a runtime metadata map key named
`ble_mesh_node_id`. This is read as an integer by the multimode and
BLE activation code, including
`com.thingclips.sdk.hardware.qpqddpb`,
`com.thingclips.sdk.hardware.qdddbpp`, and the obfuscated BLE
controller `com.thingclips.sdk.bluetooth.qdbdddp`. It is protocol
metadata, not a preference or database filename.

## Explicit DeviceKey storage

The explicit Bluetooth Mesh DeviceKey is a byte array field:

```text
com.thingclips.sdk.sigmesh.bean.ProvisionedBaseMeshNode.deviceKey
```

Related node fields in that class include `networkKey`,
`unicastAddress`, `meshUuid`, `meshId`, `uuid`, `nodeIdentifier`,
`nodeName`, `ivIndex`, `netKeyIndex`, `mElements`, added NetKey and
AppKey indexes, sequence state, company/product/version IDs, and
feature flags.

During provisioning,
`com.thingclips.sdk.bluetooth.dpdddpq` derives the DeviceKey and
calls `UnprovisionedMeshNode.setDeviceKey(byte[])`. The constructor
`ProvisionedMeshNode(UnprovisionedMeshNode)` then copies that byte
array and the other provisioning state. `ProvisionedMeshNode` is
Parcelable and its parcel representation includes `deviceKey`,
network-key data, addresses, sequence state, elements, and models.

No call path was found that writes `ProvisionedMeshNode` or its
`deviceKey` to the MMKV home keys, SharedPreferences, or SQLite.
The durable `SigMeshBean` schema has no DeviceKey field. The wrapper
`com.thingclips.sdk.sigmesh.bean.ThingSigMeshBean` can hold a
`ProvisionedMeshNode`, but the durable cache is parsed as the base
`SigMeshBean`, not this wrapper.

Consequently:

* NetKey and AppKey are recoverable from the SIG mesh MMKV JSON.
* The explicit provisioning DeviceKey is only confirmed in live
  heap/Parcelable state in this APK.
* `DeviceRespBean.localKey`, `devKey`, and `secKey` should still be
  collected, but static evidence does not prove any one of them is
  the Bluetooth Mesh DeviceKey. Do not relabel them without a value
  comparison or runtime trace.

## In-memory cache, not another database

`com.thingclips.sdk.device.cache.SmartCacheEntityManager` stores
SDK entities in `com.thingclips.smart.cache.core.CacheManager`.
Its configuration explicitly calls `skipDisk(true)`. The entity
types used by `com.thingclips.sdk.cache.business.bdpdqbp` are:

```text
1  DeviceRespBean, keyed by devId
2  GroupRespBean, keyed by group ID
3  BlueMeshBean, keyed by meshId
4  SigMeshBean, keyed by meshId
5  ProductBean, keyed by product ID
```

`SmartCacheRelationManager` also calls `skipDisk(true)` and keeps
home/mesh/device/group relations in memory. This explains why
classes such as `SigMeshProperty` and `BlueMeshProperty` appear to
be cache accessors but do not identify an additional on-disk store.

## Preferences and encryption dependencies

The per-user MMKV map is encrypted with a random key returned by
`SecuredStore.securityUserKey(mapId)`. `SecuredStore` keeps the
per-map encryption keys in another encrypted MMKV map named:

```text
GLOBAL_SECURITY_KEY
```

That map is itself opened with a key from
`SecuredStore.securityUserKeyV2("GLOBAL_SECURITY_KEY")`.
`SecuredPreferenceStore` is initialized with preference file
`global_preference` and alias `preference_alias`; its encrypted JSON
entry `__SECURITY_KEY__` participates in the key hierarchy. The
class also has a default preference filename `SPS_file`, so acquire
it if present, though this initialization path names
`global_preference` explicitly.

Relevant extraction targets are therefore:

```text
/data/user/0/com.olight.omall/files/thingmmkv/
/data/user/0/com.olight.omall/shared_prefs/global_preference.xml
/data/user/0/com.olight.omall/shared_prefs/SPS_file.xml
/data/misc/keystore/user_0/             (privileged acquisition)
```

On newer Android releases, Keystore material is hardware- or
system-bound and copying its files alone may not permit offline
decryption. A live, authorized process-level extraction is more
likely to preserve access to the keys.

The SDK also has generic MMKV IDs `Login_user_main`,
`Login_user_other`, and `MMMANAGER_THING_KV_1`, but no mesh-key
read/write path was found for them. They are secondary acquisition
targets, not the primary mesh store.

## SQLite/database finding

The APK contains Room and SQLite users for unrelated functions,
including sensor and health records, alarms, MQTT messages,
analytics, and WorkManager. Static searches across all 25 DEX
files found no mesh table DDL, mesh Room entity/DAO, or mesh database
filename. The Thingclips mesh persistence described above is MMKV
JSON, not SQLite.

## Authenticated cloud recovery route

The cloud source of the mesh cache is implemented by
`com.thingclips.sdk.home.oo000o`, a `Business` subclass. Its
`OooO0o(long)` method constructs this inner request:

```text
API:       m.life.my.group.mesh.list
version:   3.1
postData:  {"gid": <homeId>}
session:   required
```

During a full home refresh, `oo000o.OooO00o(long, List,
ResultListener)` places that `ApiBean` in an outer batch request:

```text
API:       thing.m.api.batch.invoke
version:   1.0
postData:  {"apis": [<inner ApiBean>, ...], "gid": <homeId>}
gid:       <homeId>
session:   required
result:    ArrayList<ApiResponeBean>
```

For the response whose `ApiResponeBean.api` equals
`m.life.my.group.mesh.list`, `oo000o` takes
`BusinessResponse.result` and parses it with Fastjson as
`com.thingclips.sdk.home.bean.MeshListResponseBean`. Its `sigmesh`
array contains `SigMeshBean` objects. That is the response subtree
whose `name`, `meshkey`, `meshId`, and `ivIndex` fields should be
captured. `name` is the NetKey and `meshkey` is the AppKey.

### Session and request signing

Setting `sessionRequire(true)` does more than add a cookie.
`com.thingclips.smart.android.network.Business$RequestTask` obtains
the live SID from the `IBaseUser` service registered in
`PluginManager`, using `IBaseUser.getSid()`, and passes it to
`ThingApiParams.setSession()`. Without it the task fails with
`session is not exist and need login again`.

`ThingApiParams` then supplies the normal Thingclips envelope,
including `a`, `v`, `clientId`, `appVersion`, `deviceId`, `lang`,
`os`, `postData`, `requestId`, `sid`, `time`, and other SDK and
regional fields. The final set is environment-dependent.

`com.thingclips.sdk.network.ThingApiSignManager` sorts an allowlist
of fields, substitutes `postDataMD5Hex(postData)` for the raw body,
and joins non-empty `key=value` pairs with `||`. Despite its name,
`postDataMD5Hex()` uses `MD5Util.md5AsBase64()`. The canonical text
then goes through obfuscated signer class
`com.thingclips.sdk.network.pqdbppq.bdpdqbp(String)`, which invokes:

```text
ThingNetworkSecurity.doCommandNative(context, 1,
    canonicalBytes, null, ThingSmartNetWork.mD)
```

That delegates to
`com.thingclips.smart.security.jni.JNICLibrary`. A captured SID and
static app secret are therefore not enough for a durable HTTP replay:
the request also needs fresh envelope fields and the native,
app-bound signing context. The practical authorized route is to let
the signed-in app or an in-process SDK client issue the call.

### What the installed release app can expose

The normal UI has an exact refresh path. Pulling down the device list
in `com.olight.device.ui.fragment.DeviceNewFragment` calls
`OlightDeviceManager.L(homeId, callback)`. That method obtains an
`IThingHome` with `ThingHomeSdk.newHomeInstance(homeId)` and calls
`IThingHome.getHomeDetail()`, which reaches the batch request above.
Thus, while signed into the target account and home, open the Devices
screen and pull to refresh. This refreshes the encrypted MMKV records;
it does not display the raw response.

The device drawer also has a `Log Upload` entry. `TuYaFragment`
navigates to ARouter path `/device/setting_upload_tuya_log`, backed
by `com.olight.device.ui.activity.UploadTuyaLogActivity`. That screen
calls `TLogSDK.fetchLogs(context, LogFileCallback)`, zips the returned
files, and uploads them through `CommitTuYaLogViewModel` to Olight's
support backend. The activity is `android:exported="false"`, the APK
is not debuggable, and the screen offers no save/share/view-response
operation. It is therefore a support submission route, not a local
mesh export. Logs may establish that the call ran, but static code
does not show that they contain the decrypted mesh response.

The SDK has a deprecated global flag
`ThingSmartNetWork.mPacketCaptureEnabled` and preference-name
constant `network_packet_capture`. No release UI or writer that
enables it was found. No mesh export, database export, response
viewer, or externally launchable debug activity was found either.
Proxy capture should not be treated as reliable because the SDK uses
its own transport security and native signer.

Without ADB, the immediately actionable sequence is therefore:

1. Sign in to the correct Olight/Thingclips account and select the
   target home.
2. Open Devices and pull down to refresh. This invokes the
   authenticated cloud call and repopulates the MMKV cache.
3. Use `Log Upload` only if Olight support has agreed to return or
   inspect the submitted logs; the app itself will not reveal them.
4. Recover key material from the refreshed MMKV with an authorized
   app-private backup or live in-process acquisition. The stock app
   provides no no-ADB export of the response or encrypted cache.

## Recommended acquisition and parsing order

1. Acquire the complete app-private `files/thingmmkv` directory,
   preserving each file with its `.crc` partner.
2. Acquire all `shared_prefs`, especially `global_preference.xml`
   and `SPS_file.xml`, plus an Android Keystore-aware backup or a
   live process context when authorized.
3. Determine the Thingclips UID and open map
   `preferences_global_key<UID>` with its recovered crypt key.
4. Enumerate keys beginning `s_home_data_sigmesh_list` and decode
   their JSON arrays. Extract `name` as NetKey and `meshkey` as
   AppKey, with `meshId` and `ivIndex` as context.
5. Decode the matching `s_home_data_devlist<homeId>` and join on
   `deviceTopo.meshId`. Extract node/device IDs, MAC, UUID, product,
   local key material, and communication metadata.
6. Treat an explicit SIG DeviceKey as unconfirmed unless obtained
   from live `ProvisionedMeshNode.deviceKey`, a Parcel containing
   that object, or a value-correlated runtime trace.

## Confidence and limitations

Confidence is high for the MMKV path, map ID, home-key names,
Fastjson serialization, and NetKey/AppKey backing-field mapping.
Confidence is high that the SDK's `SmartCache*` layer is memory-only
and that there is no dedicated mesh SQLite schema in the APK.

The negative DeviceKey result is bounded by static code inspection:
native code, dynamically delivered modules, server-side state, or
runtime-only IPC could add another copy. The APK evidence itself
only confirms `ProvisionedMeshNode.deviceKey` in memory/Parcel state
and does not show it in the durable mesh JSON.
