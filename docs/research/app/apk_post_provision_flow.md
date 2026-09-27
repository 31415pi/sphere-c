# Olight 3.2.5 post-provisioning flow

## Scope

This note traces the exact Thingclips SDK path from receipt of a
Bluetooth Mesh Provisioning Complete PDU through local mesh
configuration and the final cloud join. It is based on the baksmali
output under `/tmp/olight-dex.7xC3jV/smali6`, the prior APK notes,
and the now-captured Sphere C composition in
`state/sphere_state.json`.

The important conclusion is that there is no unconditional
Sphere-specific "activate" or "setup complete" vendor message after
provisioning. The normal path is standard SIG configuration followed
by cloud registration. A late Tuya/Olight vendor write does exist,
but only in a product-gated remote/group-control branch. Sphere C's
known product identifier does not set that branch's product bit.

## Provisioning Complete callback

`ppqqdpb$bdpdqbp.bdpdqbp(byte[])` parses the provisioning bearer
notification. When byte 1 is `0x08`, it logs
`TYPE_PROVISIONING_COMPLETE` and does the following:

1. Constructs `ProvisionedMeshNode` from `UnprovisionedMeshNode`.
2. Attaches it to `SigMeshSearchDeviceBean`.
3. Calls the `pqbdpqq` success callback.

That callback is `dddqpqq$dqdpbbd`. It reports progress code
`210013`, installs the current IV Index in the transport, changes the
per-device `SigConfigBean.step` to 1, and begins the proxy/config
phase. It does not send a vendor opcode.

The implementation may reconnect from the Provisioning Service to
the Mesh Proxy Service before configuration. This is transport
handling rather than another provisioning command.

## Unconditional local configuration sequence

The normal state sequence is:

```text
Provisioning Complete
  -> reconnect/enable Mesh Proxy notifications
  -> Config Composition Data Get
  -> Config AppKey Add
  -> Config Model App Bind, repeated for selected models
  -> Config Network Transmit Set
  -> final custom-code hook
  -> cloud mesh join
```

The state and progress names confirm the order:

| State | Request | Success progress |
|---|---|---:|
| `ConfigCompositionState` | Composition Data Get | `210015` |
| `ConfigAppKeyAddState` | AppKey Add | `210017` |
| `ConfigModelBindState` | Model App Bind(s) | `210019` |
| `ConfigNetworkTransmitState` | Network Transmit Set | `210023` |

After composition succeeds, the coordinator waits 200 ms before
starting AppKey Add. After model binding succeeds, the callback
directly invokes Network Transmit Set. This is significant: the
ordinary path does **not** invoke `ConfigModelPublicationState`
between bind and network-transmit.

### Composition Data Get

Class `dqpqpqq` emits request opcode `0x8008`, expects Composition
Data Status `0x02`, and supplies page byte `0xff`. The returned
composition is saved on the search bean and drives later model
selection.

### AppKey Add

Class `dqpddpd` emits Config AppKey Add opcode `0x8003`. The
NetKey/AppKey indexes are both zero in this activator, followed by
the 16-byte AppKey from `SigMeshBean`.

### Model App Bind targets

`ConfigModelBindState` scans every composition element and binds
AppKey index 0 to each present model in this fixed allow-list:

- SIG Generic OnOff Server `0x1000`;
- SIG Light Lightness Server `0x1300`;
- SIG models `0x1303`, `0x1306`, and `0x1307`;
- Tuya/Olight vendor model `0x07d0:0x0004`.

The literal list in `bbdpqbd` is exactly `0x1000`, `0x1300`,
`0x1306`, `0x1307`, `0x1303`, and packed vendor identifier
`0x07d00004`. Each request is Config Model App Bind opcode `0x803d`,
with element address, AppKey index 0, and either a SIG or vendor model
identifier. It expects Config Model App Status `0x803e`.

For the captured Sphere C composition at primary address `0x010c`,
the resulting bind set is:

- element `0x010c`: `0x1000`, `0x1300`, `0x1303`, `0x1306`,
  `0x1307`, and vendor `0x07d0:0x0004`;
- elements `0x010d` through `0x0111`: `0x1000` on each element.

The addresses vary with assigned primary address. Sphere C also has
vendor model `0x07d0:0x0005`, but the APK allow-list does not bind
that model here.

### State after Model App Bind

`dddqpqq$bppdpdq` handles successful completion of all bind requests.
It reports progress `210019` and immediately calls the activator's
`pqpbpqd()` method. In `qqqppqq`, that method starts
`ConfigNetworkTransmitState`.

There is no intervening setup-complete write, activation write, or
model-publication write in this callback.

### Network Transmit Set

`ConfigNetworkTransmitState` constructs `qbddddb(5, 2)` and sends
Config Network Transmit Set opcode `0x8024`, expecting status opcode
`0x8025`. The one-byte encoded parameters are:

```text
(intervalSteps 2 << 3) | transmitCount 5 = 0x15
```

This is the last unconditional on-node write in the standard path.
It configures network retransmission behavior; it does not carry an
activation flag or mesh credential.

## Model publication is present but not in the normal chain

The activator constructs `ConfigModelPublicationState`, but the
ordinary success callbacks above never enter it. If explicitly
entered, it sends Config Model Publication Set opcode `0x03` and
expects Config Model Publication Status `0x8019`. It publishes only
Generic OnOff Server `0x1000` from the primary element to group
`0xd000`, using AppKey index 0, credential flag false, and TTL 3.
The raw publication period fields are 24 steps at resolution 3. The
raw publication retransmit fields are count 5 and interval steps 4.
That state then proceeds to Network Transmit Set.

Therefore publication should be treated as an available conditional
state, not as a required step after Sphere C Model App Bind.

## Final custom hook and conditional group-control writes

After Network Transmit Status succeeds,
`dddqpqq$pppbppp` reports `210023` and invokes the final custom-code
hook. `pbdpppp.registCustomCodeForFinale()` chooses one of two paths.

### Ordinary device path

If the mapped mesh category is not `05` and product identifier bit
`0x0400` is clear, the hook sends no more mesh messages. It proceeds
directly to `registerToCloud()`.

Sphere C's measured composition reports product identifier `0x0200`.
Thus its product identifier does **not** set the `0x0400` gate. No
evidence currently shows Sphere C mapping to category `05`, and the
method names describe that category as a remote-control operation.
The ordinary-device path is therefore the likely Sphere C path.

### Remote/group-control path

If the mapped category is `05` or product bit `0x0400` is set, the
SDK first requests a group/local ID from the cloud. It derives a mesh
group address by parsing that ID as hex and adding `0x4000`.

There are then two subpaths:

1. With product bit `0x0400` set, `bindGroupControlDevice()` sends a
   Tuya/Olight vendor message called `LightModeSetAction`:

   ```text
   request opcode:  c9 d0 07
   response opcode: cd d0 07
   parameters:      86 02 || group_address
   model:           0x07d0:0x0004
   ```

   The group address is produced by the SDK's two-byte integer
   helper. The semantic clue is stronger than the misleading class
   name: this message is sent only while binding a group-control
   device, and success is handled by `BindGroupControlCallback`.
   It is the best APK candidate for a vendor-side bind or persistence
   command.

2. With that product bit clear, `bindConfigModelSubscription()`
   instead sends standard Config Model Subscription Add opcode
   `0x801b`, binding Generic OnOff Server `0x1000` on the primary
   element to the allocated group address.

Both branches then continue to cloud registration. Failure of these
operations is reported as activation failure; they are not silently
optional for a device which took the remote/group-control branch.

For Sphere C specifically, product ID `0x0200` rules out subpath 1
through the product-bit test. Even if Sphere C unexpectedly mapped
to category `05`, this code would use the standard subscription-add
subpath, not the `c9 d0 07` vendor write.

## Cloud join and persistence boundary

`registerToCloud()` formats the assigned node address as four hex
digits, reads the derived DeviceKey, and submits mesh/device metadata
through the mesh business layer. Prior static work resolves the
concrete API as:

```text
thing.m.device.ble.mesh.join, version 3.0
```

The request includes `meshId`, `nodeId`, `productKey`, `uuid`,
`devKey`, `mac`, `ver`, and JSON options such as company/category
metadata. On success, the resulting `DeviceBean` is delivered to the
activator's success callback and added to the success list.

This cloud join is the app/account persistence step and the best
high-level definition of "add device complete." It is not a Bluetooth
Mesh access command. Static inspection alone cannot establish which
preceding local configuration step makes Sphere C durably commit its
mesh state.

## Best candidates, ranked

1. **Sphere device-side access enablement:** Config AppKey Add plus
   the Model App Bind requests make application-key traffic usable.
   The partial local run did not prove that this subset is durably
   retained.
2. **Last unconditional device write:** Config Network Transmit Set,
   opcode `0x8024`, parameter `0x15`. Although its defined purpose is
   retransmit configuration, it is also the APK's final local setup
   boundary and is now the leading missing commit trigger to test.
3. **App/account completion:** authenticated cloud mesh join. This is
   the persistence/activation boundary visible to the Olight app.
4. **Conditional vendor bind:** `c9 d0 07 86 02 <group>` with response
   `cd d0 07`. This is a real bind-like vendor command, but the known
   Sphere C product ID does not select its product-gated subpath.

The local self-provisioning experiment showed only transient control,
not durable provisioning. During that session Sphere C accepted
AppKey index 0, one Generic OnOff Model App Bind, and responded to
Generic OnOff. A fresh 15-second scan later saw the same `0x010c`
Sphere advertising Mesh Provisioning Service UUID `0x1827` again.
The device therefore rolled back to an unprovisioned state after the
partial sequence.

That result invalidates the earlier claim that AppKey Add plus one
Model App Bind was sufficient for retention. It does not identify a
vendor completion command: the inspected normal APK chain still has
none. Instead, the exact next persistence test should reproduce the
whole unconditional APK sequence:

1. Composition Data Get.
2. AppKey Add at index 0.
3. Bind every allowlisted model actually present: on primary `0x010c`,
   bind `0x1000`, `0x1300`, `0x1303`, `0x1306`, `0x1307`, and
   `0x07d0:0x0004`; on `0x010d` through `0x0111`, bind `0x1000`.
4. Send Network Transmit Set `0x8024` with payload `0x15` and require
   Network Transmit Status `0x8025`.
5. Disconnect, wait beyond the previous rollback interval, scan for
   `0x1827`, and preferably power-cycle before testing control again.

Model Publication Set should remain excluded from this first exact
replication because it is not in the ordinary post-bind callback
chain. Likewise, do not add the conditional vendor group-bind message
unless runtime evidence shows Sphere took that category/product path.

## Confidence limits

- The state transitions and literal opcodes/parameters above are
  high-confidence static findings.
- Sphere C company/product/version and composition were measured
  during the transient local provisioning session; the device later
  returned to unprovisioned advertising.
- The exact app-supplied category string for Sphere C was not found
  in the available encrypted phone logs. The conclusion that it is
  not remote category `05` is therefore a well-supported inference,
  not a captured runtime fact.
- Publication is initialized in the activator but is not called by
  the ordinary callback chain inspected here.

## Experimental resolution

The exact active APK sequence was later reproduced at unicast
`0x0112`: all 11 allowlisted model binds succeeded, Config Network
Transmit Set `8024 15` returned Status, and read-only AppKey traffic
worked. The node continued to advertise the matching Network ID after
disconnect, accepted a delayed proxy reconnect, and retained the same
keys and binding across a complete power cycle. Publication, the
conditional group branch, vendor writes, and cloud join were not
needed for durable provisioning.
