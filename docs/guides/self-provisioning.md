# Sphere C self-provisioning runbook

This workflow provisions a factory-reset Sphere C into a new,
locally controlled Bluetooth Mesh network. It does not involve the
vendor app or cloud.

## Initial persistent run: 2026-09-26

The Sphere C was provisioned persistently at unicast address `0x0112`.
It advertised GATT Proxy using Node Identity, accepted our
AppKey, returned Composition Data page 0, completed the Olight app's
full standard model-bind sequence, and accepted Network Transmit Set
with count 5 and interval steps 2. An AppKey-authenticated Generic
OnOff Get returned `present=0` without changing the light.

After disconnect, a fresh 20-second scan found a Mesh Proxy advert
whose Network ID exactly matched our NetKey. A second, delayed GATT
Proxy connection then returned another authenticated Generic OnOff
Status. This proves retained network membership, AppKey, and model
binding across disconnect and beyond the earlier rollback window.

The final durability test turned the Sphere off with its button for
about 10 seconds and turned it back on. After waking it again
advertised our exact Network ID and answered a read-only Generic OnOff
Get with `present=1`. The chosen NetKey, DeviceKey, AppKey, unicast
address, and model configuration therefore survived a button-off/wake
cycle. This was not a verified hard power cycle: the internal battery
is not removable, and turning the light output off does not establish
that the controller lost power.

The authoritative composition identifies the device as:

- company ID `0x07d0` (Olight)
- product ID `0x0200`
- version ID `0x3530`
- six elements
- Relay, Proxy, and Friend features (`0x0007`)
- a primary element with standard Generic OnOff, Generic Level,
  Light Lightness, CTL, HSL, and xyL models
- two Olight vendor models, `0x07d0:0x0004` and
  `0x07d0:0x0005`
- five secondary elements, each with a Generic OnOff Server

This proves that ordinary Bluetooth Mesh tooling can control the
standard lighting surface. No hidden vendor activation command was
needed. The vendor models are still likely needed for Olight-specific
effects and settings.

The successful persistent capture is under
`old/captures/self-provisioning/`. The key-bearing state is
`state/sphere_state.json` and remains mode `0600`.

The generated NetKey and AppKey, the derived Device Key, the node
address, and the transmit sequence number live in
`state/sphere_state.json`. Keep that file. Do not restore an older copy
with the same controller source address: that can reuse sequence
numbers and encryption nonces.

The entries at `0x0100`, `0x0106`, and `0x010c` record completed
provisioning handshakes whose nodes subsequently rolled back. Address
`0x0112` records the first persistent local run.

The difference between the rollback at `0x010c` and persistence at
`0x0112` is the remainder of the APK's unconditional standard setup:
the persistent run bound all 11 allowlisted model instances and sent
Config Network Transmit Set (`0x8024`, parameter `0x15`). Both are
standard SIG Mesh configuration. The experiment does not yet isolate
which individual late operation is the firmware's commit checkpoint.

## Ownership round trip: 2026-09-27

Linux first authenticated to the node at `0x0112`, then sent Config
Node Reset. The Olight app claimed the unprovisioned Sphere and
controlled it successfully. With phone Bluetooth disabled, three quick
button presses put the app-owned Sphere into green pairing mode. It
then advertised only the Mesh Provisioning service with the same
Device UUID, proving that the physical gesture erased its provisioned
mesh state without requiring the former controller.

Linux provisioned the Sphere back onto the existing local NetKey and
AppKey at `0x0118`, repeated all 11 model bindings and Network Transmit
Set, and received an AppKey-authenticated Generic OnOff Status. A
second independent GATT connection returned another authenticated
status. The active allocation is now `0x0118`; the older entries are
historical and must not be selected for control.

The pre-app checkpoint under `state/checkpoints/` is forensic evidence,
not a state file that can safely be restored verbatim. Its keys remain
useful for recovery, but its controller address and sequence cursor can
be stale. Recovery with badly stale sequence state must use a fresh,
non-overlapping controller source address or reprovision the node.

## Prepared stages

Create the key state without touching Bluetooth:

```console
./bin/sphere-mesh --init-only
```

After a factory reset, identify unprovisioned Mesh beacons:

```console
./.venv/bin/python scripts/scan_mesh.py
```

Provision the selected Device UUID, but stop before configuration:

```console
./bin/sphere-mesh --device-uuid DEVICE_UUID --provision-only
```

Read composition data, add the AppKey, and bind a standard control
model without changing the light state:

```console
./bin/sphere-mesh --toggle-only --no-toggle
```

Only after reviewing the composition and saved state, perform a
short standard-model control test:

```console
./bin/sphere-mesh --toggle-only
```

## Capture

Run `btmon` for the complete reset/provision/configure session and
write its new output under `state/captures/`. Historical experiment
captures are archived under `old/captures/self-provisioning/`.
Bluetooth monitor
access requires elevated privileges on this host, so Codex will
request approval immediately before starting that capture.

On Olight app 3.2.5, reset the Sphere with **Remove Device** followed
by **Clear**. **Restore Factory Colors** is unrelated and only resets
the lighting palette. After the reset, close the app and disable the
phone's Bluetooth. The phone will not know the locally chosen mesh
keys and will no longer control the Sphere until it is reset and
provisioned again.

With the phone out of the picture, three quick presses of the bottom
button put even an app-owned Sphere into green-flashing pairing mode
and return it to an unprovisioned beacon. This provides a physical
recovery path without the former controller. A provisioned node can
also be reset using a device-key-authenticated Config Node Reset;
`./bin/sphere-mesh --reset-node` implements that path while retaining
the recovery keys locally.

On this Linux host, reliable operation also required loading a 5-second
LE supervision timeout with `scripts/set_le_conn_params.py`. BlueZ
caches old advertisements and removes device objects when discovery
stops, so the scanner now accepts only advertisements received after
scan start and keeps discovery active through each GATT connection
handoff.

## Current limitations

- Provisioning uses PB-GATT, not PB-ADV.
- No output/input OOB authentication is implemented.
- Static OOB is supported with `--static-oob HEX` if required.
- Segmented unicast responses are assembled and acknowledged by the
  pinned `btmesh` dependency. Reliable retransmission of missing
  outbound segments is not implemented.
- IV Index recovery and a receive replay cache are not implemented.
