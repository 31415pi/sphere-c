# Olight 3.2.5: Thingclips SIG Mesh provisioning

## Scope and method

This note is a static inspection of:

- `old/apk/olight-3.2.5.apk`;
- the DEX implementation bundled in that APK; and
- the existing notes and packet-capture analysis in this repo.

The APK was not changed or run. ADB was not used. Its SHA-256 is:

```text
1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9
```

Names such as `dpdddpq` below are obfuscated DEX class names.
Public model and API names are retained where the APK exposes them.

## Result in one paragraph

The Olight app delegates ordinary SIG Mesh provisioning to the
Thingclips SDK. The mesh NetKey and AppKey come from the
cloud-backed `SigMeshBean`, not from the node. For a new mesh, the
app asks the Thingclips service to create that bean. Locally, the
provisioner generates a fresh P-256 key pair, exchanges the public
key with the node, and derives an ECDH secret. This inspected path
selects No OOB or Output OOB, but the caller supplies no user-entered
value. It therefore uses an all-zero authentication value in the
observed orchestration. The DeviceKey is derived locally from ECDH
and both provisioning randoms. It is not the NetKey or AppKey. The
NetKey enters the encrypted Provisioning Data PDU; the AppKey is
added afterward with a DeviceKey-protected configuration message.
The app then sends the derived DeviceKey and node metadata to the
authenticated Thingclips cloud join API.

## Credential origins

### NetKey and AppKey

`com.thingclips.sdk.bluetooth.pbdpppp` builds its provisioning
configuration from:

```text
SigMeshBean.getNetWorkkey()
SigMeshBean.getAppkey()
```

The storage aliases are important:

```text
SigMeshBean.name     -> NetKey
SigMeshBean.meshkey  -> AppKey
```

For new meshes,
`com.thingclips.sdk.bluetooth.pdppddb.createSigMesh()` calls the
obfuscated API wrapper `ddbpqbb`. That wrapper sends an authenticated
request to:

```text
thing.m.device.sig.mesh.create, version 1.0
```

It supplies the home/group ID as `gid`, requires a session, and
returns a `SigMeshBean`. The success path caches that returned bean.
There is no local NetKey or AppKey generator on this creation path.
The high-confidence conclusion is that the server response supplies
both keys for a newly created mesh.

An existing home can also restore its SIG mesh list from the
authenticated cloud. `docs/research/app/apk_storage.md` identifies:

```text
m.life.my.group.mesh.list, version 3.1
```

Its response includes the same `SigMeshBean` fields. Local cached
copies are described in that note.

`bbqqpqb` parses the NetKey hex and rejects any value that is not
16 bytes. It places the key in `UnprovisionedMeshNode` with NetKey
index zero. The IV Index comes from the local mesh manager. Flags are
zero in this path.

The AppKey is not part of a SIG Mesh Provisioning Data PDU. After
provisioning completes, the SDK proceeds through classes whose state
names expose the sequence:

```text
ConfigCompositionState
ConfigAppKeyAddState
ConfigModelBindState
ConfigModelPublicationState
ConfigNetworkTransmitState
```

The AppKey string passed into the provisioning coordinator is used
by the AppKey Add and later configuration steps.

### ECDH key material

`dpdqqdd` creates a new key pair using:

```text
curve:      secp256r1
agreement:  ECDH
provider:   bundled Thingclips Bouncy Castle code
```

It converts the provisioner's public X and Y coordinates to fixed
unsigned 32-byte values and sends their 64-byte concatenation. The
private key remains in the provisioning state object.

`ddbpdpq` receives the provisionee's 64-byte X/Y public value. It
reconstructs and validates a point on `secp256r1`, performs ECDH with
the provisioner's private key, and stores the 32-byte shared secret
on `UnprovisionedMeshNode`.

This is a local, ephemeral key agreement. The inspected code does not
ask the cloud to generate the P-256 pair or calculate ECDH.

## Exact provisioning flow

The implementation has distinct state/action classes for:

```text
Invite and Capabilities
Provisioning Start
Provisioning Public Key
Provisioning Confirmation
Provisioning Random and confirmation check
Provisioning Data
Provisioning Complete
post-provisioning configuration
```

The confirmation-input helper in `bbqqpqb` constructs:

```text
0x05 || Capabilities || Start ||
ProvisionerPublicKeyXY || ProvisioneePublicKeyXY
```

Here `0x05` is the Invite attention-timer value. Capabilities are
serialized as the standard 11 fields, and Start as the standard five
selection fields.

### Authentication selection

`pqdqbpp` and the Start-PDU helper select authentication as follows:

- If the mapped Output OOB action is zero, method, action, and size
  are all zero: No OOB.
- Otherwise, the method is `2`: Output OOB. The mapped output action
  and the device-advertised output size are used.
- The public-key field in Start is always zero, selecting an in-band
  public-key exchange.
- This path does not select Static OOB or Input OOB.

The confirmation action accepts an optional decimal string. If it is
present, the code parses it as an integer and writes it into the last
four bytes of the 16-byte authentication value. If it is absent, the
authentication value remains 16 zero bytes.

The actual provisioning coordinator calls that action with an empty
string. Therefore this app path supplies an all-zero authentication
value. For a No OOB device that is the expected value. Although the
generic Start builder can select Output OOB, this coordinator does
not show a UI value being passed back into the confirmation step. An
Output OOB device would therefore appear to fail unless zero were the
accepted output. This is a strong code-path finding, but it is not a
captured Sphere C Capabilities PDU.

### Confirmation and random exchange

`qqqbpdd` generates a fresh 16-byte provisioner random and computes:

```text
ConfirmationSalt = s1(ConfirmationInputs)
ConfirmationKey = k1(ECDHSecret, ConfirmationSalt, "prck")
Confirmation = AES-CMAC(
    ConfirmationKey,
    ProvisionerRandom || AuthValue
)
```

After random exchange, `bqdqdqq` recalculates and checks the
provisionee confirmation in the same way. The bundled
`SecureUtils` contains the standard provisioning labels `prck`,
`prsk`, `prsn`, and `prdk`.

### DeviceKey and Provisioning Data

`dpdddpq` calculates:

```text
ProvisioningSalt = s1(
    ConfirmationSalt ||
    ProvisionerRandom ||
    ProvisioneeRandom
)

SessionKey = k1(ECDHSecret, ProvisioningSalt, "prsk")
SessionNonceFull = k1(ECDHSecret, ProvisioningSalt, "prsn")
SessionNonce = final 13 bytes of SessionNonceFull
DeviceKey = k1(ECDHSecret, ProvisioningSalt, "prdk")
```

It writes the result with
`UnprovisionedMeshNode.setDeviceKey(byte[])`.

The 25-byte plaintext sent to the node is:

```text
NetKey (16) || key index (2) || flags (1) ||
IV Index (4) || primary unicast address (2)
```

This path uses key index zero and flags zero. It encrypts that
plaintext with AES-CCM under the session key and nonce and appends an
eight-byte MIC.

The DeviceKey is thus node-specific material derived from ephemeral
ECDH plus fresh randoms. It is not derived from the NetKey, AppKey,
Device UUID, MAC address, product ID, or a cloud secret.

## Cloud involvement

The cloud participates around, not inside, the provisioning
handshake:

1. Before provisioning, it returns the mesh `SigMeshBean`, including
   NetKey and AppKey, through an authenticated create or list API.
2. Local Bluetooth code performs P-256 ECDH, confirmation, random
   verification, DeviceKey derivation, and encrypted data transfer.
3. After provisioning, `pbdpppp` reads the DeviceKey from the new
   `ProvisionedMeshNode`, converts it to hex, and registers the node
   through:

```text
thing.m.device.ble.mesh.join, version 3.0
```

The request requires a session and contains at least:

```text
meshId, nodeId, productKey, uuid, devKey, mac, ver
```

It may also include `options`. The `nodeId` is the assigned unicast
address formatted as four hex digits. The registration helper builds
`uuid` by removing colons from the MAC address and lowercasing it.
This cloud field is distinct from the 128-bit SIG Mesh Device UUID
used during unprovisioned discovery.

Compatibility wrappers in the APK expose other join API versions.
The version above is the concrete call made by this inspected
provisioning path.

Static inspection therefore proves that the app uploads the derived
DeviceKey as `devKey`. It does not prove how the backend protects it,
whether it later returns it, or whether a cloud `devKey` field in a
different device model is always the SIG Mesh DeviceKey.

## UUID and factory-credential findings

The SDK obtains the SIG Mesh Device UUID from the unprovisioned
beacon or the 16-byte Mesh Provisioning Service data and stores it in
`UnprovisionedMeshNode`. This UUID is a public discovery identifier,
not an authentication secret.

No hard-coded Sphere C Device UUID, factory P-256 private key, Static
OOB value, or other Sphere-specific factory provisioning credential
was found in the APK or repo notes. The visible Sphere product ID and
name are identifiers, not credentials.

The APK also contains generic, non-SIG Thingclips BLE `authKey` code
and APIs. No call from the SIG Mesh provisioning path above to that
subsystem was found. Those values must not be relabeled as SIG Mesh
Static OOB or DeviceKey without runtime evidence.

The supplied packet capture is of an already-provisioned proxy
connection. It contains no provisioning bearer exchange, no
Unprovisioned Device Beacon, and no Capabilities PDU. It therefore
cannot reveal this physical Sphere C unit's Device UUID, advertised
OOB capability, public keys, randoms, or factory data.

## Confidence and boundaries

### High confidence

- NetKey and AppKey enter this path from `SigMeshBean`.
- New-mesh `SigMeshBean` data is returned by an authenticated cloud
  create API.
- ECDH uses a locally generated `secp256r1` provisioner key pair.
- The inspected Start builder uses an in-band public key.
- The orchestrated authentication value is all zero.
- DeviceKey uses the standard `prdk` derivation from ECDH and the
  provisioning salt.
- NetKey is sent in encrypted Provisioning Data; AppKey is added
  afterward.
- The derived DeviceKey is uploaded as `devKey` during cloud join.

### Medium confidence

- Sphere C follows this exact generic SDK branch in production. The
  Olight activator reaches these classes, but no live factory-reset
  trace was available to confirm every branch.
- Sphere C advertises No OOB. The coordinator behavior strongly
  suggests that expectation, but the unit's Capabilities PDU was not
  captured.

### Not established

- A Sphere C factory UUID or Static OOB secret.
- A reusable factory asymmetric private key.
- Any way to reconstruct the DeviceKey from an existing proxy
  capture.
- Backend retention, encryption, or later export of uploaded
  DeviceKeys.
- A public or unauthenticated cloud route for any mesh credential.

## Related repo notes

- `docs/research/app/apk_crypto.md` covers cryptographic primitives, cache
  exposure, and key logging.
- `docs/research/app/apk_storage.md` covers `SigMeshBean` aliases and durable
  storage.
- `docs/research/mesh/mesh_provisioning.md` explains the normative SIG Mesh
  procedure.
- `docs/analysis/capture.md` explains why the supplied capture cannot provide the
  provisioning secrets.
