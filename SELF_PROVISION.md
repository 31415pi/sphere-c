# Sphere C self-provisioning runbook

This workflow provisions a factory-reset Sphere C into a new,
locally controlled Bluetooth Mesh network. It does not involve the
vendor app or cloud.

## Successful persistent run: 2026-09-26

The Sphere C is now provisioned persistently at unicast address
`0x0112`. It advertised GATT Proxy using Node Identity, accepted our
AppKey, returned Composition Data page 0, completed the Olight app's
full standard model-bind sequence, and accepted Network Transmit Set
with count 5 and interval steps 2. An AppKey-authenticated Generic
OnOff Get returned `present=0` without changing the light.

After disconnect, a fresh 20-second scan found a Mesh Proxy advert
whose Network ID exactly matched our NetKey. A second, delayed GATT
Proxy connection then returned another authenticated Generic OnOff
Status. This proves retained network membership, AppKey, and model
binding across disconnect and beyond the earlier rollback window.

The final durability test fully powered the Sphere off for about 10
seconds and turned it back on. After reboot it again advertised our
exact Network ID and answered a read-only Generic OnOff Get with
`present=1`. The chosen NetKey, DeviceKey, AppKey, unicast address,
and model configuration therefore survived a real power cycle in
nonvolatile storage.

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

The successful persistent capture is
`mesh-lab/captures/self-config-full-20260926.btsnoop`. The key-bearing
state is `mesh-lab/sphere_state.json` and remains mode `0600`.

The generated NetKey and AppKey, the derived Device Key, the node
address, and the transmit sequence number live in
`mesh-lab/sphere_state.json`. Keep that file. Losing or rolling it
back can make the node unreachable without another factory reset.

The entries at `0x0100`, `0x0106`, and `0x010c` record completed
provisioning handshakes whose nodes subsequently rolled back. The live
allocation is `0x0112`; do not select an older entry for control.

The difference between the rollback at `0x010c` and persistence at
`0x0112` is the remainder of the APK's unconditional standard setup:
the persistent run bound all 11 allowlisted model instances and sent
Config Network Transmit Set (`0x8024`, parameter `0x15`). Both are
standard SIG Mesh configuration. The experiment does not yet isolate
which individual late operation is the firmware's commit checkpoint.

## Prepared stages

Create the key state without touching Bluetooth:

```console
./sphere_mesh --init-only
```

After a factory reset, identify unprovisioned Mesh beacons:

```console
./.venv/bin/python scan_mesh.py
```

Provision the selected Device UUID, but stop before configuration:

```console
./sphere_mesh --device-uuid DEVICE_UUID --provision-only
```

Read composition data, add the AppKey, and bind a standard control
model without changing the light state:

```console
./sphere_mesh --toggle-only --no-toggle
```

Only after reviewing the composition and saved state, perform a
short standard-model control test:

```console
./sphere_mesh --toggle-only
```

## Capture

Run `btmon` for the complete reset/provision/configure session and
write its output under `mesh-lab/captures/`. Bluetooth monitor
access requires elevated privileges on this host, so Codex will
request approval immediately before starting that capture.

On Olight app 3.2.5, reset the Sphere with **Remove Device** followed
by **Clear**. **Restore Factory Colors** is unrelated and only resets
the lighting palette. After the reset, close the app and disable the
phone's Bluetooth. The phone will not know the locally chosen mesh
keys and will no longer control the Sphere until it is reset and
provisioned again.

With the phone out of the picture, three quick presses of the bottom
button put the reset Sphere into green-flashing pairing mode. Incomplete
self-provisioning attempts returned to an unprovisioned beacon, providing
a physical recovery path. A provisioned node can also be reset using a
device-key-authenticated Config Node Reset; `./sphere_mesh --reset-node`
implements that path while retaining the recovery keys locally.

On this Linux host, reliable operation also required loading a 5-second
LE supervision timeout with `set_le_conn_params.py`. BlueZ caches old
advertisements and removes device objects when discovery stops, so the
scanner now accepts only advertisements received after scan start and
keeps discovery active through each GATT connection handoff.

## Current limitations

- Provisioning uses PB-GATT, not PB-ADV.
- No output/input OOB authentication is implemented.
- Static OOB is supported with `--static-oob HEX` if required.
- Segmented responses are assembled, but segment acknowledgements
  are not transmitted by this feasibility harness.
- IV Index recovery and a receive replay cache are not implemented.
