# Sphere C self-provisioning experiment

## Purpose and safety boundary

This is a plan, not a record of a completed experiment. It must be
run only against the owner's Sphere C, on an isolated test subnet.
Nothing in this plan requires changing the existing capture.

The experiment is disruptive. A factory reset removes the Sphere
from its current Olight mesh. Without the original NetKey and
DeviceKey, the old membership cannot be restored in place. The lamp
can still be reset and added to Olight again as a new node.

The safest path remains extracting the current Olight keys first.
Only use this plan if that path is exhausted and loss of the current
membership is accepted.

## Stop/go gate before any physical reset

Do not reset the lamp until every item below is true.

- The owner explicitly accepts losing the current Olight membership.
- The Olight app, account, home, and Internet access are available
  for the fallback re-add procedure.
- The official reset and re-add procedure has been tested on another
  device or transcribed from the correct Sphere C manual.
- The lamp is charged and only one target Sphere is powered nearby.
- The existing app entry, room, scenes, schedules, firmware version,
  product ID, and visible identifiers have been recorded.
- A final labeled Olight control capture has been saved if useful.
- `sphere-c.btsnoop` still hashes to:
  `7fa0332b793b9445f6556d0a04dd5bad1846a32cfb2f093265e214e26fb3401b`.
- The provisioner has completed a dry run with a disposable Mesh
  development node.
- Persistent provisioner storage survives a reboot and sequence
  numbers continue rather than restart.
- The fixed keys, key indexes, address plan, IV Index, and output
  directory have been written to a run manifest.
- The phone's Bluetooth is off and other provisioners are absent.

If any item is false, stop before resetting the Sphere.

## Provisioner prerequisites

Use a provisioner with a persistent Configuration Database (CDB),
a Configuration Client model, PB-GATT client support, Mesh Proxy
client support, and durable sequence-number storage. A current Zephyr
Mesh provisioner build is a useful reference implementation. Its
official sample configures a node by adding an AppKey, reading
composition, and binding models.

The host needs:

- a dedicated Bluetooth controller or supported Zephyr board;
- a clean, backed-up provisioner settings/CDB directory;
- HCI capture, preferably `btmon -w`, started before scanning;
- a timestamped terminal log with both commands and responses;
- a second passive scanner if possible;
- an RF-quiet location with the lamp within about one metre;
- no Olight app connection during the run.

Do not use ordinary `bluetoothctl` pairing. Sphere C uses Bluetooth
Mesh provisioning, not an Android-style BLE bond. The expected
provisioning service is `0x1827`; after provisioning, configuration
uses the mesh network, commonly through Proxy service `0x1828`.

The Zephyr shell command names below are a concrete checklist, not a
requirement to use Zephyr. Confirm them against the exact build before
the reset. Required build options include provisioning, CDB,
Configuration Client, PB-GATT, GATT Proxy Client, settings, and enough
transport buffers for segmented configuration messages.

## Fixed disposable test network

Use public, fixed lab values so that the run is reproducible:

```text
NetKey Index:       0x000
NetKey:             <redacted-experimental-net-key>
AppKey Index:       0x000
AppKey:             <redacted-experimental-app-key>
IV Index:           0x00000000
Key Refresh flag:   0
IV Update flag:     0
Provisioner address: 0x0001
Sphere primary:      0x0100
```

These are public test credentials, not production secrets. Keep the
test subnet small and short-lived. Do not join any other device to it.
The Sphere DeviceKey cannot be chosen: both sides derive it during
provisioning. The provisioner must persist the derived DeviceKey.

Reserve `0x0100` through `0x010F` in the run ledger initially. The
Capabilities PDU reports `NumElements`. Before sending Provisioning
Data, verify that the actual occupied range will be:

```text
0x0100 .. 0x0100 + NumElements - 1
```

It must not overlap `0x0001` or another CDB node. The current Tuya
lighting profile says one element, but this must not be assumed for
the physical Sphere C.

## Expected models, with confidence limits

The existing capture proves Tuya company ID `0x07D0` and structurally
matches OnOff, Lightness, CTL Temperature, HSL, and vendor traffic.
It does not contain decrypted Composition Data. The exact baseline
below comes from Tuya's published one-element lighting profile:

| Kind | Company | Model ID | Expected role |
|---|---:|---:|---|
| SIG | n/a | `0x0000` | Configuration Server |
| SIG | n/a | `0x1000` | Generic OnOff Server |
| SIG | n/a | `0x1300` | Light Lightness Server |
| SIG | n/a | `0x1306` | Light CTL Temperature Server |
| SIG | n/a | `0x1307` | Light HSL Server |
| Vendor | `0x07D0` | `0x0004` | Tuya Vendor Server |

Tuya also writes its packed vendor server identifier as
`0x07D00004`, with client `0x07D00005`. For a configuration command,
pass model ID `0x0004` and Company ID `0x07D0` separately unless the
tool explicitly asks for the packed value.

The repository's vendor opcodes are:

```text
WRITE       0xC9D007
WRITE_UNACK 0xCAD007
READ        0xCCD007
DATA        0xCDD007
```

Tuya's current document labels `0xCBD007` as a reserved STATUS value
and `0xCDD007` as DATA. That distinction should replace any informal
use of “status” for `0xCDD007` in earlier notes.

Tuya's general SDK initialization page also lists a Light CTL model,
whose SIG server ID is `0x1303`. Its one-element lighting access
profile does not list `0x1303`, and the repo only inferred CTL
Temperature traffic. Therefore `0x1303` is a discovery candidate,
not a Sphere C binding target unless Composition Data reports it.
The same rule applies to setup, level, hue, and saturation models.

Composition Data is authoritative. Never bind an inferred model that
is absent, and never assume every model is on the primary element.

## Phase 0: prepare and prove the provisioner

1. Start with empty, dedicated CDB storage. Do not reuse an Olight or
   household mesh database.
2. Start HCI capture and a timestamped console transcript.
3. Initialize the provisioner and create the CDB with the exact
   NetKey. Add the exact AppKey to the CDB.
4. Provision the local provisioner at `0x0001`, IV Index zero.
5. Reboot it. Confirm the CDB, keys, IV Index, local address, and
   monotonically advancing sequence state all survive.
6. Confirm scanning and PB-GATT provisioning against a disposable
   test node. Reset that node cleanly afterward.

A current Zephyr shell expresses the setup approximately as:

```text
mesh init
mesh cdb create <redacted-experimental-net-key>
mesh cdb app-key-add 0 0 <redacted-experimental-app-key>
mesh prov local 0 0x0001 0
mesh cdb show
```

Do not omit the key arguments: the shell otherwise permits defaults
or random keys. Export a protected copy of the CDB after local setup.

## Phase 1: baseline and reset

1. Record the lamp's current physical state and whether Olight can
   still control it.
2. Start a new capture and log file before touching the lamp.
3. Scan for the existing Proxy advertisement and record its Network
   ID. The old capture used `1d497fd760450169`; a change is useful
   evidence but not itself an error.
4. Re-run the stop/go gate.
5. Perform the documented physical factory reset once.
6. Do not repeat reset gestures while diagnosing later failures.
7. Confirm an unprovisioned beacon or PB-GATT advertisement carrying
   the 16-byte Device UUID and service `0x1827`.
8. Record the UUID, Bluetooth address and type, RSSI, service data,
   and the exact visible lamp indication.

If no unprovisioned beacon appears, stop. Power-cycle once and scan
again. Do not improvise more reset sequences. The device may not have
reset, may have a delayed window, or may use PB-ADV instead.

## Phase 2: provision only

1. Listen for unprovisioned beacons and select by the recorded UUID,
   not by the random BLE address or Olight `0xFCF1` address.
2. Request PB-GATT with NetKey Index 0, primary address `0x0100`,
   and a five-second attention value.
3. Capture and log the full Capabilities PDU before choosing an
   authentication method.
4. Verify `NumElements` fits the reserved address range.
5. Prefer the strongest advertised and usable OOB method. The APK
   path suggests in-band public keys and No OOB, but the physical
   Sphere's capabilities have not been captured.
6. Select No OOB only if the device advertises it and the user has
   visually confirmed the sole target at close range. Abort if the
   capabilities require an unknown Static OOB value or an unusable
   input/output action.
7. Complete provisioning once. Require a Provisioning Complete PDU.
8. Immediately persist the UUID, address range, element count,
   DeviceKey, NetKey index, IV Index, and provisioner sequence state.
9. Back up the CDB before attempting post-provision configuration.

For the current Zephyr shell, the action is approximately:

```text
mesh prov beacon-listen on
mesh prov auth-method none
mesh prov remote-gatt <device-uuid> 0 0x0100 5
mesh cdb show
```

Do not run the last two lines until Capabilities and authentication
policy are observable in the chosen build. If the tool automatically
selects No OOB before exposing Capabilities, change or instrument it
before using it on the Sphere.

### Provisioning failure checkpoint

- Failure before Provisioning Data: the node should remain
  unprovisioned. Stop the bearer, retain the capture, and retry only
  after diagnosing the transcript.
- Timeout after Provisioning Data: status is ambiguous. First scan
  for the old UUID, new Mesh Proxy or Network ID, then inspect the
  CDB. Do not reset or blindly reprovision.
- Provisioning Complete but no saved DeviceKey: stop. Recover the
  provisioner's CDB or logs. If the key is truly lost, only a physical
  reset can recover the lamp.
- A UUID or element-count mismatch means the wrong target. Stop.

## Phase 3: read authoritative composition

Provisioning does not configure application models. First reconnect
through Mesh Proxy if needed, target node `0x0100` on NetKey Index 0,
and send a DeviceKey-protected Composition Data Get for page 0.

```text
mesh target dst 0x0100
mesh target net 0
mesh models cfg get-comp 0
```

Save the raw response and parse:

- Company ID, Product ID, Version ID, CRPL, and feature bits;
- every element's location and unicast address;
- every SIG model ID;
- every `(Company ID, vendor model ID)` pair.

Require the returned number of elements to equal Capabilities. Compare
the result with the expected model table, but do not reject a valid
response merely because the Sphere firmware has additional models.

If this read times out, retry with a longer Configuration Client
timeout and verify Proxy connectivity, destination, NetKey index,
address, IV Index, and CDB DeviceKey. Do not factory-reset: no AppKey
or model binding is required to read composition.

## Phase 4: add the AppKey

Send Config AppKey Add under the node's DeviceKey:

```text
mesh models cfg appkey add 0 0 \
  <redacted-experimental-app-key>
mesh models cfg appkey get 0
```

Require both a successful status and an AppKey list containing index
0. Save both responses. If an add times out, query the list before
retrying; the node may have committed the key before the response was
lost. Re-adding the same key/index is recoverable. Never substitute a
different key at index 0 without first establishing node state.

## Phase 5: bind only reported application servers

For each reported element, bind AppKey Index 0 to the application
server models that actually appear there. Do not bind the
Configuration Server: foundation models use the DeviceKey.

For the expected one-element profile at `0x0100`, the candidate
commands are:

```text
mesh models cfg model app-bind 0x0100 0 0x1000
mesh models cfg model app-bind 0x0100 0 0x1300
mesh models cfg model app-bind 0x0100 0 0x1306
mesh models cfg model app-bind 0x0100 0 0x1307
mesh models cfg model app-bind 0x0100 0 0x0004 0x07D0
```

Run only the lines supported by Composition Data. If `0x1303` or
other application servers are present, bind them deliberately after
checking their role. Query each binding with Config SIG/Vendor Model
App Get. Treat a timeout as ambiguous and query before retrying.

Do not configure group subscriptions, publication, relay, friend,
proxy, network transmit, scenes, schedules, or vendor writes in this
first experiment. They are unnecessary for unicast validation and
increase the rollback surface.

## Phase 6: staged validation

Validate from least to most state-changing:

1. Re-read Composition Data page 0 and all AppKey bindings.
2. Send acknowledged Get messages by unicast for Generic OnOff,
   Light Lightness, CTL Temperature, and HSL where their server models
   exist. Require a source address in the Sphere's assigned range and
   the expected Status opcode.
3. Send the Tuya vendor READ only after its framing is independently
   implemented. Validate that replies use company `0x07D0` and DATA
   opcode `0xCDD007`; do not send a vendor WRITE yet.
4. Record current OnOff and light values.
5. Send one acknowledged Generic OnOff Set to the current value with
   a fresh transaction ID. This exercises write handling without an
   intentional visible change.
6. Send one acknowledged, visible OnOff change, observe the lamp and
   Status, then restore the recorded state immediately.
7. Only then test Lightness, CTL Temperature, and HSL. Use conservative
   mid-range values and restore every recorded value after its test.
8. Power-cycle the lamp. Repeat the Get tests to prove that keys,
   address, bindings, IV state, and provisioner sequence state persist.
9. Reboot the provisioner and repeat once more.

Every acknowledged request needs a bounded timeout and retry count.
Never replay captured Network PDUs; Mesh replay protection and
sequence numbers make that both invalid and diagnostically confusing.

Success requires:

- stable Mesh Proxy reconnection after both power cycles;
- authenticated status replies under the chosen AppKey;
- correct source and destination addresses;
- no replay or sequence rollback errors;
- CDB contents matching the saved manifest;
- restored user-visible light state at the end.

## Failure handling before any second physical reset

Use this order. A physical reset is the last step, not the first.

1. Stop state-changing commands and preserve capture, console log,
   CDB, and settings exactly as they are.
2. Classify the last acknowledged stage: provisioning, composition,
   AppKey add, individual binding, or access-model validation.
3. Check radio connection, Proxy advertisement, Network ID, target
   address, NetKey/AppKey indexes, IV Index, and local sequence state.
4. Reconnect the proxy and issue a DeviceKey-protected Composition
   Data Get. Success proves the node and DeviceKey are recoverable.
5. Query AppKey and model-binding state. Retry only missing,
   idempotent configuration operations with the same values.
6. Restore the backed-up CDB if only local storage was damaged. Never
   create a new node record with a guessed DeviceKey.
7. If access messages fail but configuration succeeds, inspect AppKey
   value, AID, binding, element address, opcode, and TID. Do not reset.
8. If configuration fails, verify the persisted DeviceKey and IV/
   sequence state against the immediate post-provision backup.
9. Only if the DeviceKey is irretrievable, or the user chooses to
   abandon the lab mesh, proceed to the recovery section.

## Clean rollback and recovery

### Roll back while the DeviceKey is available

1. Restore the lamp's recorded light state.
2. Send Config Node Reset to `0x0100` under its DeviceKey.
3. Wait for the acknowledged reset status or disconnect, then confirm
   the original Device UUID advertises as unprovisioned.
4. Only after that confirmation, delete the node record from the CDB.
5. Archive the capture, transcript, manifest, composition, and CDB
   backup. Treat the archive as secret because it contains keys.
6. Either end the experiment or factory-reset/re-add through Olight.

Deleting the CDB entry before resetting the node loses the DeviceKey
needed for a clean remote reset and can strand the node.

### Return to Olight

The disposable keys cannot recreate the old Olight membership. Once
the lamp is unprovisioned, use the documented Olight add-device flow.
If a stale cloud entry remains, remove it only through the app after
preserving screenshots and only when the app's workflow requires it.
Expect a new DeviceKey and possibly a new unicast address.

After Olight control works, capture a simple labeled Get/Set sequence
and compare its Network ID with the pre-reset capture. Do not upload
the disposable CDB or keys to Olight.

### If remote reset is impossible

Keep every artifact and try recovery with the saved CDB first. If the
DeviceKey is genuinely lost, repeat only the documented physical
factory reset. Confirm the unprovisioned UUID before starting either
the lab or Olight provisioning again.

## Run artifact checklist

For each attempt, retain:

- UTC and local timestamps and software/firmware versions;
- controller identity and provisioner build configuration;
- SHA-256 hashes for capture, log, manifest, and CDB backup;
- Device UUID, Capabilities, selected bearer and OOB method;
- NetKey/AppKey indexes, IV Index, flags, and address allocation;
- derived DeviceKey in protected storage, not a public report;
- raw Composition Data and parsed model table;
- every configuration request, status code, and retry;
- each validation request, response, TID, and observed lamp state;
- final recovery state: lab mesh, unprovisioned, or Olight re-added.

Redact all keys and DeviceKey before publishing logs.

## Sources

Repository evidence:

- `ANALYSIS.md`
- `COMPATIBILITY.md`
- `PROVISIONING.md`
- `research/mesh_provisioning.md`
- `research/tuya_provisioning.md`

Primary and implementation references:

- Bluetooth SIG, Mesh Protocol 1.1:
  <https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html>
- Bluetooth SIG, Mesh Model 1.1:
  <https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MMDL_v1.1/out/en/index-en.html>
- Bluetooth SIG Assigned Numbers:
  <https://www.bluetooth.com/specifications/assigned-numbers/>
- Tuya one-element lighting access profile:
  <https://developer.tuya.com/cn/docs/iot-device-dev/tuya-sigmesh-light-access-standard?id=K9pieafepp3q7>
- Tuya vendor-model pass-through profile:
  <https://developer.tuya.com/en/docs/iot-device-dev/tuya-sigmesh-device-vendor-model-access-standard?id=K9pikwhoo3gux>
- Zephyr Bluetooth Mesh shell:
  <https://docs.zephyrproject.org/latest/services/connectivity/bluetooth/api/mesh/shell.html>
- Zephyr Configuration Database guidance:
  <https://docs.zephyrproject.org/latest/services/connectivity/bluetooth/api/mesh/cdb_usage.html>
