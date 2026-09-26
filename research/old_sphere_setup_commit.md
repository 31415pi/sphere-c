# Sphere setup completion and persistence research

## Question and result

This note asks whether older Olight Sphere or Tuya reverse-
engineering material identifies a vendor-specific post-provisioning
command that commits provisioning state and prevents the Sphere C
from returning to its unprovisioned state.

The source review found **no device-side vendor commit command**.
However, live evidence now proves that the minimal standard sequence
implemented locally is **not sufficient**: after Provisioning
Complete, AppKey Add, Composition Data Get, and a successful Generic
OnOff bind at `0x010c`, a fresh 15-second Bleak scan saw the same
physical device actively advertising service `0x1827` with the same
unprovisioned Device UUID. The node had rolled back.

The leading next experiment is therefore the rest of the standard
device-side flow visible in the Olight APK: Model Publication followed
by Network Transmit configuration. It is also possible that the app
binds a broader model set, performs group/subscription configuration
for this product branch, or merely keeps the session alive long enough
for a delayed NVM checkpoint. The evidence does not yet isolate which
condition matters.

The Olight 3.2.5 APK does have a named `Finale`, but it is app/cloud
bookkeeping. It uploads the node and DeviceKey to Thingclips. It does
not issue a Bluetooth Mesh vendor access message to the Sphere.

## Evidence ranked by usefulness

### 1. Local experiment: minimal standard configuration rolled back

The allocation at `0x010c` completed this ordinary configuration:

- Config AppKey Add, NetKey index 0 and AppKey index 0;
- Config Composition Data Get, page 0;
- Config Model App Bind to the Generic OnOff Server;
- no intentional light-state change.

It did **not** persist. A later fresh 15-second Bleak scan found
`DC:23:52:09:A8:CF` advertising the Mesh Provisioning service
`0x1827` and the same Device UUID used before provisioning. This is
stronger evidence than a failed proxy scan: seeing the same UUID on
`0x1827` positively identifies a return to the unprovisioned state.

The earlier `0x0100` and `0x0106` handshakes also rolled back. Thus
all three locally allocated addresses are historical controller-side
records, not live durable nodes.

Sources:

- [`SELF_PROVISION.md`](../SELF_PROVISION.md), which records the
  attempted `0x010c` flow but whose “successful”/“live” persistence
  wording predates the decisive `0x1827` rescan and is stale;
- [`provision_sphere.py`](../provision_sphere.py), `configure()` and
  the `--no-toggle` path;
- [`mesh-lab/provision_run.log`](../mesh-lab/provision_run.log), lines
  575-657;
- [`mesh-lab/sphere_state.json`](../mesh-lab/sphere_state.json), the
  historical `0x010c` controller record and saved composition;
- `mesh-lab/captures/self-config-fast-20260926.btsnoop`.

The decisive 15-second follow-up scan was reported live after these
artifacts were written and is not present in `provision_run.log`.
Until that scan is saved separately, its exact observation should be
kept with the experiment ledger: MAC `DC:23:52:09:A8:CF`, service
`0x1827`, and the same pre-provision Device UUID.

The local implementation contains no Olight/Tuya vendor setup write.
Its configuration function ends after a successful SIG Config Model
App Status. The rollback proves only that this endpoint is too early;
it does **not** prove that a hidden vendor write is required.

### 2. Olight APK: the post-provision sequence is standard Mesh config

Static inspection of `apk/olight-3.2.5.apk` found these explicit
Thingclips SDK states, in order:

```text
ConfigCompositionState
ConfigAppKeyAddState
ConfigModelBindState
ConfigModelPublicationState
ConfigNetworkTransmitState
```

This was previously summarized in
[`research/tuya_provisioning.md`](tuya_provisioning.md), lines 87-99.
The underlying extracted DEX classes are in `classes6.dex` and were
inspected as smali under the temporary extraction directory:

```text
com/thingclips/sdk/bluetooth/qddppqd.smali   composition
com/thingclips/sdk/bluetooth/qqdpbbd.smali   AppKey Add
com/thingclips/sdk/bluetooth/bbdpqbd.smali   model binding
com/thingclips/sdk/bluetooth/bqpbqbp.smali   publication
com/thingclips/sdk/bluetooth/dqbdpdd.smali   network transmit
com/thingclips/sdk/bluetooth/dddqpqq*.smali  state callbacks
com/thingclips/sdk/bluetooth/qqqppqq.smali   orchestrator
```

The classes use the node's DeviceKey and Configuration Server. No
Olight company opcode appears between these states.

Some exact Olight-app defaults are now the highest-value candidates
for reproducing the missing end of the device-side flow:

- Model Publication configures Generic OnOff Server `0x1000` to group
  address `0xD000`.
- The model-bind state does much more than the local one-model test.
  It walks every composition element and binds every reported match
  from this allowlist: Generic OnOff `0x1000`, Light Lightness
  `0x1300`, Light CTL Temperature `0x1306`, Light HSL `0x1307`, Light
  CTL `0x1303`, and vendor model `0x07D00004`. On the observed Sphere
  C composition this means the primary models plus Generic OnOff on
  each of the five secondary elements.
- The Generic OnOff publication uses AppKey index 0, friendship
  credentials disabled, TTL 3, period steps 24 with resolution 3,
  retransmit count 5, and retransmit interval steps 4.
- Network Transmit is constructed with count `5` and interval steps
  `2` in `dqbdpdd.smali`.
- The publication object also contains standard publication TTL,
  period, and retransmit fields; it is a Config Model Publication Set,
  not a vendor access write.

The callback chain labels success after each standard step with SDK
progress codes `210015`, `210017`, `210019`, `210021`, and `210023`.
After Network Transmit succeeds, the orchestrator enters its custom
`Finale` hook.

### 3. The APK “Finale” is cloud registration, not a lamp command

The relevant path is:

```text
qqqppqq.bdpdqbp(SearchDeviceBean, FinaleData)
  -> qqbppqp.registCustomCodeForFinale(...)
  -> pbdpppp.registCustomCodeForFinale(...)
  -> pbdpppp.registerToCloud(...)
```

In `pbdpppp.smali`, `registerToCloud()` serializes the assigned node
address, MAC, product data, company ID, mesh category, and the derived
DeviceKey. It then invokes the Thingclips mesh join business API. The
concrete API was identified as:

```text
thing.m.device.ble.mesh.join, version 3.0
```

This matches the detailed analysis in
[`research/tuya_provisioning.md`](tuya_provisioning.md), “Cloud
involvement”. This is an HTTPS/cloud operation; no access PDU is sent
to the Sphere by this method.

Consequently, cloud join cannot directly toggle a device-side NVM
commit. It could still correlate with persistence indirectly because
the app reaches it only after the complete Mesh configuration flow and
keeps the activation process alive while the request finishes. A
local experiment should first reproduce all preceding on-mesh steps,
then wait and disconnect cleanly, without attempting cloud join.

There is a conditional “remote control” branch for certain generic
Thingclips categories or product-identifier flags. It obtains or adds
a local group before cloud registration. Even there, the observed
operations are group/subscription setup followed by cloud join, not an
Olight-specific NVM commit command. Nothing ties that optional branch
to Sphere C, whose composition product identifier is `0x0200`.

### 4. Vendor models exist, but they are not in the setup-complete path

The physical Sphere C reports Olight company ID `0x07D0`, product ID
`0x0200`, six elements, and vendor models `0x0004` and `0x0005` on the
primary element. See the composition record in
[`SELF_PROVISION.md`](../SELF_PROVISION.md) and
[`mesh-lab/sphere_state.json`](../mesh-lab/sphere_state.json).

Tuya vendor access opcodes known in this repository are:

```text
WRITE       0xC9D007
WRITE_UNACK 0xCAD007
READ        0xCCD007
DATA        0xCDD007
```

They carry datapoints for effects and product-specific settings. They
are documented in [`COMPATIBILITY.md`](../COMPATIBILITY.md) and
[`research/self_provision_plan.md`](self_provision_plan.md). None is
called by the APK's provisioning/configuration state machine before
the finale. The local `0x010c` attempt sent none of them, but also
rolled back; this absence therefore cannot be used as proof that
vendor traffic is unnecessary. The stronger APK evidence remains that
the normal state chain reaches its finale without an explicit vendor
WRITE.

The public Sphere C profile likewise describes runtime datapoints,
not provisioning completion. It identifies product `d0dpga0h` and
DPs for power, colour, scenes, battery, and sleep:

- [tuya-local Sphere C profile](https://github.com/make-all/tuya-local/blob/main/custom_components/tuya_local/devices/olight_sphere_light.yaml)

The earlier compatibility search found no public implementation that
specifically identifies itself as an original-Sphere provisioning
implementation. See [`COMPATIBILITY.md`](../COMPATIBILITY.md),
“Public-source search boundary”. Recent public SIG Mesh examples also
treat provisioning and binding as separate phases, rather than using
a vendor commit command; for example, Telink's published library says
binding adds the AppKey and binds target models after provisioning:

- [TelinkBleMeshLib networking README](https://github.com/thanhtunguet/TelinkBleMeshLib)

## Current rollback hypotheses

The evidence supports only the first three steps of this behavior:

1. SIG provisioning completes and the node temporarily advertises
   Node Identity / Mesh Proxy.
2. The firmware starts a configuration-completion window.
3. If no meaningful configuration completes, it abandons the new node
   state and returns to its unprovisioned beacon.
4. Some later completion condition marks the node durable.

Step 2 is an inference from the three rollback observations, not a
Bluetooth Mesh requirement. The Mesh specification itself considers
the node provisioned after Provisioning Complete. The vendor firmware
is free to implement an onboarding watchdog.

Candidate completion conditions, in descending order of evidence, are:

1. completing the APK's full standard flow: all selected bindings,
   Model Publication, and Network Transmit;
2. one particular late standard operation within that flow;
3. optional group/subscription setup selected by the Sphere category;
4. a clean disconnect or sufficient delay after the final standard
   Config Status;
5. an unidentified vendor command.

The fifth hypothesis currently has the least source support: no such
command appears in the normal APK state chain. It remains possible
only because the exact live Olight branch has not been captured from
factory reset through durable completion.

## Recommended next experiment

Reproduce the complete standard device-side APK sequence while
changing as little user-visible state as possible:

1. Persist the DeviceKey immediately after Provisioning Complete.
2. Reconnect through Mesh Proxy using Node Identity.
3. Send Config AppKey Add and require Success.
4. Read Composition Data page 0.
5. Reproduce the APK binding allowlist on every matching element:
   `0x1000`, `0x1300`, `0x1306`, `0x1307`, `0x1303`, and vendor
   `0x07D0:0x0004`. For this composition, that includes Generic OnOff
   on the five secondary elements as well as the matching primary
   models.
6. Send standard Config Model Publication Set for Generic OnOff Server
   `0x1000` to `0xD000` with AppKey index 0, credentials disabled,
   TTL 3, period `(resolution 3, steps 24)`, and retransmit
   `(count 5, interval steps 4)`.
7. Send standard Config Network Transmit Set with count `5` and
   interval steps `2`; require Config Network Transmit Status.
8. Leave the link up for a bounded grace period, then disconnect
   cleanly.
9. Run a fresh scan that distinguishes `0x1827` from `0x1828`, then
   power-cycle and repeat the scan.
10. Only call the node durable if it remains on Mesh Proxy/Node
    Identity and no longer advertises the same UUID on `0x1827`.

Do not add a speculative Olight vendor write. It is unsupported by the
evidence and may alter scenes, sleep behavior, or other user-visible
state. Cloud join should not be necessary to test device persistence.
If the complete standard flow still rolls back, instrument or capture
the app's optional group/subscription branch before considering a
vendor WRITE.

## Confidence

- **High:** Provisioning Complete plus AppKey Add, Composition Get,
  and one Generic OnOff bind did not produce a durable node.
- **High:** the Olight APK's main post-provision device traffic is
  standard Configuration Server traffic.
- **High:** the named Finale uploads node metadata and DeviceKey to the
  cloud; it is not a BLE Mesh command to the lamp.
- **Medium:** completing standard Model Publication and Network
  Transmit is the best next anti-rollback experiment because these are
  the missing device-side steps in the APK sequence.
- **Unknown:** whether broader model binding, group/subscription setup,
  or a post-configuration delay is the actual checkpoint.
- **Low / unsupported:** a vendor DP write or cloud join is itself the
  device-side NVM commit.

## Experimental resolution

The full active standard APK sequence was subsequently reproduced at
unicast `0x0112`. The provisioner bound all 11 allowlisted model
instances and sent Network Transmit Set `8024 15`; all statuses were
successful. AppKey-authenticated Generic OnOff Get worked immediately,
after a delayed reconnect, and after a full power cycle. The node
advertised the Network ID derived from our NetKey after disconnect and
reboot.

No Model Publication, group/subscription setup, vendor access command,
or cloud join was sent. The successful result therefore resolves the
main question: ordinary standard Mesh configuration is enough, but the
earlier one-model subset was not. The experiment does not isolate
whether the decisive checkpoint is the complete bind allowlist,
Network Transmit Set, or their combination.
