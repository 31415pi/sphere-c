# Post-provision configuration and persistence comparison

## Question and result

The local captures do not include an official Olight-app
provisioning session. `sphere-c.btsnoop` starts with an already
provisioned node using the Mesh Proxy service. It has no PB-GATT
provisioning PDU and no Provisioning Complete PDU. The three
`mesh-lab/captures/` files are from the local Python provisioner.

The official app's likely post-Provisioning Complete sequence can
nevertheless be recovered with high confidence from the Olight 3.2.5
APK. It performs normal SIG Mesh configuration and then application
and cloud setup. No special vendor opcode is an unconditional
"commit provisioning" operation.

The successful local configuration exchange contains only AppKey Add,
Composition Data Get, and one SIG Model App Bind. It did not receive
Network Transmit Set, the official app's full set of model binds, a
vendor-model access command, or cloud registration in that capture.

That exchange was protocol-successful but was not durable. A later
fresh 15-second Bleak discovery, rather than a cached BlueZ mapping,
saw `DC:23:52:09:A8:CF` actively advertising service `0x1827` with
UUID `dc235209a8cf10146430647067613068`. The node had returned to the
unprovisioned state. The missing official-app tail is therefore a
live persistence hypothesis and should be reproduced experimentally.

## Capture inventory

| Capture | What it contains | Official provisioning trace? |
|---|---|---|
| `sphere-c.btsnoop` | Existing Olight network through Mesh Proxy, 251 encrypted Network PDUs | No |
| `self-provision-20260926.btsnoop` | Local PB-GATT experiments and successful Provisioning Complete | No |
| `self-config-20260926.btsnoop` | Earlier local configuration attempts | No |
| `self-config-fast-20260926.btsnoop` | Successful local post-provision configuration | No |

`sphere-c.btsnoop` cannot be decrypted because the old Olight NetKey,
AppKey, and DeviceKey are unavailable. Its message bursts therefore
cannot identify an official post-provision commit command.

## Decrypted successful local sequence

The following values were independently recovered from
`self-config-fast-20260926.btsnoop` using the saved lab NetKey and the
DeviceKey for node `0x010c`. The relevant traffic occurs at the end of
the capture, from 507.315 to 508.346 seconds.

| Frame(s) | SEQ | Direction | Decrypted access message |
|---:|---:|---|---|
| 5673, 5676 | 0, 1 | `0x0001 -> 0x010c` | Config AppKey Add (`0x00`), indexes 0/0, lab AppKey |
| 5704 | 2926 | `0x010c -> 0x0001` | Config AppKey Status (`0x8003`), success |
| 5706 | 2 | `0x0001 -> 0x010c` | Config Composition Data Get (`0x8008`), page 0 |
| 5709-5716 | 2938-2945 | `0x010c -> 0x0001` | segmented Composition Data Status (`0x02`) |
| 5718 | 3 | `0x0001 -> 0x010c` | Segment Acknowledgment for composition |
| 5721 | 4 | `0x0001 -> 0x010c` | Config Model App Bind (`0x803d`), element `0x010c`, AppKey 0, Generic OnOff Server `0x1000` |
| 5725 | 2946 | `0x010c -> 0x0001` | Config Model App Status (`0x803e`), success |

The exact outbound decrypted access payloads were:

```text
00 000000 <redacted-live-app-key>
8008 00
803d 0c01 0000 0010
```

The first message is segmented because AppKey Add is 20 access bytes.
The bind is little-endian in its parameters: element `0x010c`, packed
AppKey index 0, and SIG model `0x1000`.

There are no further Network PDUs in the locally successful
configuration exchange after Model App Status at frame 5725. In
particular, the capture has no vendor opcode from company `0x07d0`.
The later active `0x1827` advertisement proves this short exchange was
not sufficient for durable provisioning.

## Official app sequence recovered from the APK

The Thingclips coordinator constructs and registers these state
objects:

```text
ConfigCompositionState
ConfigAppKeyAddState
ConfigModelBindState
ConfigModelPublicationState
ConfigNetworkTransmitState
final product/cloud hook
```

The names alone misleadingly suggest that all five run in sequence.
Following the success callbacks shows the active ordinary path is:

```text
Composition -> AppKey Add -> Model Bind -> Network Transmit -> finale
```

`ConfigModelPublicationState` is present and fully implemented, but
no success callback in this inspected coordinator invokes its start
method. Its own success callback advances to Network Transmit, which
shows where it would fit in another branch or SDK integration. It is
not an unconditional operation in this app path.

The corresponding active messages and constants are:

1. Config Composition Data Get, opcode `0x8008`, parameter `0xff`.
   The SDK uses page `0xff`, unlike the local harness's page `0x00`.
2. Config AppKey Add, opcode `0x00`, with NetKey index 0, AppKey
   index 0, and the `SigMeshBean` AppKey.
3. Repeated Config Model App Bind, opcode `0x803d`, for every
   composition model matching this allowlist:

   ```text
   SIG:    0x1000, 0x1300, 0x1303, 0x1306, 0x1307
   Vendor: company 0x07d0, model 0x0004
   ```

   On this Sphere C composition, that means the primary element's
   Generic OnOff, Light Lightness, Light CTL, CTL Temperature, HSL,
   and Tuya/Olight vendor server. It also means Generic OnOff Server
   on each of the five secondary elements. The vendor operation is
   still the standard DeviceKey-protected Config Model App Bind, not
   a vendor access opcode.
4. Config Network Transmit Set, opcode `0x8024`, with count 5 and
   interval steps 2. The exact access payload is `8024 15`; the
   expected status opcode is `0x8025`. This is the first
   unconditional active operation missing after the local bind.
5. After Network Transmit Status succeeds, the generic SDK invokes
   `registCustomCodeForFinale`. The Olight/Thingclips implementation
   either registers the node to the cloud immediately or first sets
   up remote/group control, depending on category and product flags.

For completeness, the dormant/alternate Config Model Publication
implementation uses opcode `0x03` for the primary Generic OnOff
Server and constructs:

   ```text
   element:                 primary unicast
   publish address:         0xd000
   AppKey index:            0
   credential flag:         false
   publish TTL:             3
   period resolution:       3
   period steps:            24
   retransmit count:        5
   retransmit interval:     4 steps
   model:                   0x1000
   expected status opcode:  0x8019
   ```

The cloud registration is the previously identified
`thing.m.device.ble.mesh.join` operation. It uploads node metadata and
the derived DeviceKey. It commits the app/backend's knowledge of the
node, but it does not commit mesh state inside the lamp.

## Conditional post-configuration group operations

The product finale contains two conditional paths that could be
mistaken for a persistence command.

### Standard group-subscription path

For one remote-control branch, the cloud returns a group identifier.
The app converts it into a mesh group address and sends Config Model
Subscription Add (`0x801b`) for the primary Generic OnOff Server
`0x1000`. The expected status is `0x801f`.

This is ordinary model configuration for group control, not a commit.

### Vendor group-control path

If product-identifier bit `0x0400` is set, the app instead constructs
a Tuya vendor WRITE using:

```text
opcode:       0xc9, company 0x07d0  (wire c9 d0 07)
model:        company 0x07d0, model 0x0004
parameters:   86 02 || encoded group address
response:     opcode 0xcd, company 0x07d0  (wire cd d0 07)
success body: 86 01 00
```

The APK has a `ConfigGroupConfirmState` listener that treats
`cd d0 07 86 01 00` as success and reports other status values as
"group bind fail". This is a group-control association, not a generic
provisioning commit. It is conditional and occurs after the standard
configuration and Network Transmit Set.

The local artifacts do not establish whether Sphere C takes this
conditional branch. Composition PID `0x0200` is not necessarily the
same field as the SDK's product identifier, so its `0x0400` test
cannot be inferred from Composition Data alone.

## Persistence conclusion and next test

The local status responses establish only that the node accepted the
messages at that time. They do not establish durable storage. The
later fresh `0x1827` discovery proves that the node rolled back.

There is still no APK symbol or message whose documented semantics is
"commit provisioning." Provisioning Complete should mean that the
node accepted its NetKey, unicast address, IV Index, flags, and
derived DeviceKey. Network Transmit Set configures retransmission, the
conditional vendor command configures group control, and cloud join
persists the app/backend's record. Nonetheless, this device's observed
rollback makes the difference operationally important even if the
commit is an undocumented firmware side effect.

The next provisioning attempt should reproduce the full active APK
sequence before disconnecting or power-cycling:

1. request Composition Data page `0xff`;
2. add AppKey index 0;
3. bind AppKey 0 to every allowlisted model actually present in the
   composition, including vendor server `0x07d0:0x0004` and Generic
   OnOff Server on all six elements;
4. send Config Network Transmit Set access payload `8024 15` and wait
   for successful `0x8025` status;
5. run the product finale appropriate to the device flags: standard
   Generic OnOff subscription (`0x801b`) or the vendor group write
   `c9 d0 07 86 02 || group`, waiting for its success response;
6. only then disconnect, wait, power-cycle if desired, and use a fresh
   active scan to distinguish `0x1828` from rollback to `0x1827`.

Cloud join should be logged for parity but cannot be reproduced
offline and should not directly change node flash. The dormant Model
Publication Set is not part of the traced active callback chain; add
it only as a second controlled experiment if the full active sequence
above still rolls back.

## APK artifacts inspected

The key decompiled classes were:

```text
qqqppqq                provisioning/configuration coordinator
qddppqd / dqpqpqq      Composition Data state/action
qqdpbbd / dqpddpd      AppKey Add state/action
bbdpqbd / pbdppqq      Model App Bind state/action
bqpbqbp / dpppqbq      Model Publication state/action
dqbdpdd / qbddddb      Network Transmit state/action
pdqbdbq                vendor group-confirm listener
pbdpppp                Olight/Thingclips finale and cloud join
pbppppp                Config Model Subscription Add action
ppbddqd                Tuya vendor group-control write action
```

These artifacts were available in the existing temporary APK
disassembly under `/tmp/olight-dex.7xC3jV/`. The durable source APK is
`apk/olight-3.2.5.apk`.

## Experimental resolution

The recommended exact-chain experiment subsequently succeeded at
primary address `0x0112`. All 11 allowlisted binds returned Success,
Network Transmit Status returned the requested encoded value `0x15`,
and an AppKey-authenticated Generic OnOff Get returned Status. A fresh
scan after disconnect matched the locally derived Network ID, a delayed
proxy reconnect succeeded, and another authenticated Get succeeded
after a full power cycle. No publication, group/subscription finale,
vendor access write, or cloud join was used.

This narrows the persistence checkpoint to the missing portion of the
ordinary standard sequence: completing the full allowlisted bind set
and/or Network Transmit Set. The experiment does not isolate those two
conditions further.
