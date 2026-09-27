# Linux Bluetooth Mesh provisioning route

Research date: 2026-09-26.

## Recommendation

Use BlueZ's separate `bluetooth-meshd` daemon and its
`org.bluez.mesh` system-bus API.  Provision locally with PB-ADV.
Use `mesh-cfgclient` first as the reference client and diagnostic
tool.  For repeatable Sphere C support, either automate that tool or
make a small provisioner around the same D-Bus API.

Do not base new work on `meshctl`.  It is BlueZ's old PB-GATT client.
Upstream builds it only inside `if DEPRECATED`, while
`mesh-cfgclient` is the normal Mesh tool.  Debian still packages both,
which can make `meshctl` look current when it is not.

This recommendation assumes that a reset Sphere C emits an
Unprovisioned Device Beacon and accepts PB-ADV.  That must be checked
on the physical light.  BlueZ's current `Management1.AddNode` path
hard-codes `PB_ADV`; it offers no PB-GATT selection.

## Why this route

The current BlueZ daemon owns the difficult parts:

- PB-ADV segmentation, acknowledgements, and retries;
- P-256 provisioning crypto and authentication;
- sequence numbers, IV state, replay protection, and key storage;
- local and remote Device Keys;
- network transmission through kernel MGMT or generic HCI;
- local and remote provisioning, including Mesh 1.1 remote
  provisioning.

Applications use the documented D-Bus API instead of sending raw HCI
or provisioning PDUs.  BlueZ 5.87 source contains `pb-adv.c` in the
daemon and no PB-GATT implementation in that daemon.  Its
`add_node_call()` explicitly invokes the provisioning initiator with
`PB_ADV`.

PB-GATT remains possible through the old `meshctl`, which talks to
`bluetoothd` and contains its own `tools/mesh-gatt` stack.  It is a
reasonable one-off fallback only if the Sphere advertises the Mesh
Provisioning Service UUID `0x1827` but no PB-ADV unprovisioned beacon.
It is not the preferred base for locally maintained automation.

## Host and package expectations

The inspected host has Debian BlueZ `5.87-2`.  `bluez-meshd` is not
installed.  The configured repositories currently offer
`bluez-meshd 5.87-3`.

On Debian, package `bluez-meshd` supplies:

```text
/usr/libexec/bluetooth/bluetooth-meshd
/usr/bin/mesh-cfgclient
/usr/bin/mesh-cfgtest
/usr/bin/meshctl
/usr/lib/systemd/system/bluetooth-mesh.service
/usr/share/dbus-1/system-services/org.bluez.mesh.service
/usr/share/dbus-1/system.d/bluetooth-mesh.conf
/etc/bluetooth/mesh-main.conf
```

Runtime dependencies are `libell0`, `libjson-c5`, D-Bus, GLib,
readline, and libc.  Building BlueZ's tools needs the corresponding
development packages, notably `libell-dev`, `libjson-c-dev`,
`libdbus-1-dev`, `libglib2.0-dev`, and `libreadline-dev`.

The daemon defaults to:

```text
system bus name: org.bluez.mesh
storage:         /var/lib/bluetooth/mesh
configuration:   /etc/bluetooth/mesh-main.conf
controller I/O:  auto
```

`mesh-cfgclient` defaults to
`~/.config/meshcfg/config_db.json`.  Its JSON tracks topology,
composition, and the attachment token; authoritative key material and
live sequence/IV state remain in the daemon storage.  `--config`
should point at a project-specific, permission-restricted database.

`auto` uses the kernel MGMT interface when available and falls back
to generic HCI.  A particular adapter can be selected with
`--io generic:hci0`.  BlueZ documents Linux 4.9 or newer plus kernel
user crypto, AES, CCM, CMAC, AEAD, hash, and skcipher support.  Modern
Debian kernels normally enable these.

The package's D-Bus policy lets ordinary clients call
`org.bluez.mesh`; the daemon itself runs as a system service.  Keep
its storage private and backed up.  It contains live mesh secrets and
state that must not be casually copied between running provisioners.

## Key control and persistence

The three key types have different rules.

### NetKey

`Network1.CreateNetwork` creates the primary NetKey at index `0x000`
inside the daemon.  The caller cannot choose that value.

For a new, independent test mesh this is fine.  Run
`mesh-cfgclient`, use `create`, then use the generated primary key.

To reproduce an Olight/Thingclips mesh with a chosen primary NetKey,
bootstrap the local provisioner with `Network1.Import`, not
`CreateNetwork`.  `Import` accepts the local UUID, local DevKey,
chosen NetKey and index, flags, IV Index, and local unicast address.
The application must expose the required `Application1`,
`Provisioner1`, `ProvisionAgent1`, `Element1`, and ObjectManager
objects.

Stock `mesh-cfgclient` has no command for this local import.  Its
`subnet-import` command cannot replace the already-created primary
key.  For this C-oriented project, the lowest-risk custom route is a
small patch to current `mesh-cfgclient` that adds an `import-local`
bootstrap command and reuses its existing `JoinComplete`, token,
database, provisioner, and Configuration Client code.  A wholly new
D-Bus application must recreate that object hierarchy and the
Configuration Client message/database logic.

### AppKey

The local key database can accept an exact 16-byte AppKey:

```text
appkey-import <net_idx> <app_idx> <32-hex-digit-key>
```

The direct D-Bus equivalent is
`Management1.ImportAppKey(net_index, app_index, app_key)`.

Importing only changes the local daemon database.  After provisioning
the Sphere, send Config AppKey Add to the node, then bind that AppKey
to each model that needs application messages.

### DevKey

The target Sphere's DevKey cannot be selected during standards-based
provisioning.  It is derived from the provisioning ECDH secret and
transcript.  BlueZ derives it and saves it in its private keyring.

After provisioning, `Management1.ExportKeys` returns NetKeys,
AppKeys, and remote Device Keys to the owning provisioner application.
`mesh-cfgclient` exposes this as `export-db` and writes
`export_db.json` beside its configured database.  Treat that export
as a secret.

`node-import <uuid> <net_idx> <primary> <ele_count> <dev_key>` and
`Management1.ImportRemoteNode` let the caller supply a known DevKey
for a node provisioned elsewhere.  They do not provision the physical
device and cannot impose that DevKey on it.

The local provisioner's own DevKey may be supplied to
`Network1.Import`; that is distinct from the Sphere's derived DevKey.

## Provision and configure sequence

For a clean, unprovisioned Sphere C:

1. Start `bluetooth-meshd` and attach a provisioner application.
2. Create a fresh network, or import a local provisioner with the
   desired primary NetKey and IV Index.
3. Import or create the AppKey before remote configuration.
4. Call `Management1.UnprovisionedScan`.
5. Record the 16-byte Device UUID from `Provisioner1.ScanResult`.
6. Call `Management1.AddNode(uuid, {})`.
7. In `Provisioner1.RequestProvData`, choose the subnet index and a
   free contiguous unicast range for the reported element count.
8. Handle `ProvisionAgent1` callbacks.  The current Sphere research
   predicts No OOB, but the physical Capabilities PDU is authoritative.
9. Wait for `AddNodeComplete`; do not treat the initial method return
   as completion.
10. Send Config Composition Data Get, normally page `0`.
11. Send Config AppKey Add for the chosen AppKey index.
12. Send Config Model App Bind for the required SIG and vendor models.
13. Configure publications/subscriptions only after composition is
    known.
14. Export and protect the database and keys.

With `mesh-cfgclient`, the equivalent interactive flow is:

```text
create
appkey-import 0000 0000 <APPKEY>
discover-unprovisioned on 60
list-unprovisioned
provision <32-hex-digit-device-uuid>
menu config
target <primary-unicast>
composition-get 0
appkey-add 0000
bind <element-address> 0000 <model-id> [vendor-id]
export-db
```

For an imported primary NetKey, replace `create` with the proposed
local-import bootstrap.  `mesh-cfgclient` supports `--config`,
`--address-start`, `--net-index`, `--timeout`, and `--init-script`.
Use a dedicated config path and an init script for repeatability.
Interactive/piped commands are useful for bring-up; direct D-Bus
state handling is preferable for a finished controller.

BlueZ's official Rust binding, BlueR, exposes a Mesh feature, but its
three Mesh examples are currently checked in with `.disabled`
suffixes.  Treat it as useful API scaffolding rather than the most
proven end-to-end provisioner.  The C `mesh-cfgclient` remains the
closest maintained executable reference for configuration behavior.

The Configuration Client submenu also includes NetKey and AppKey
get/add/delete, bind/unbind, model-AppKey queries, publication,
subscription, TTL, proxy, relay, heartbeat, and node-reset commands.
Composition responses are decoded into CID, PID, VID, CRPL, feature
bits, elements, SIG model IDs, and vendor company/model IDs, and are
stored in the client database.

## Sphere-specific blockers and checks

- An already-provisioned Sphere will normally not emit an
  unprovisioned beacon.  Factory-reset it before this flow, or import
  its existing unicast address and exact DevKey instead.
- NetKey and AppKey alone are insufficient to configure an existing
  node.  Configuration messages require its DevKey.
- Factory reset should invalidate the old DevKey and membership.  Do
  not expect the Olight app to keep controlling the node afterward.
- Confirm bearer support before coding.  In reset mode, look for an
  unprovisioned beacon (AD type `0x2b`) for PB-ADV and/or Mesh
  Provisioning Service `0x1827` for PB-GATT.
- Keep only one provisioning attempt active.  `AddNode` cancels the
  caller's outstanding scan, and BlueZ serializes the pending
  provisioning procedure.
- Allocate unicast addresses by element count, not one address per
  product.  Overlap corrupts the provisioner's view of the network.
- Do not edit `/var/lib/bluetooth/mesh` JSON/keyring files directly.
  Use `Import`, `ImportSubnet`, `ImportAppKey`,
  `ImportRemoteNode`, and `ExportKeys`.

## PB-GATT fallback boundary

Use `meshctl` only if a reset-device capture proves that PB-GATT is
the only bearer the Sphere accepts.  It depends on `bluetoothd`,
connects to the Mesh Provisioning GATT service, and keeps its own
`prov_db.json`-style database.  It is not an API-compatible front end
to `bluetooth-meshd`, and its database must not be confused with the
daemon's keyring or `mesh-cfgclient` database.

A more maintainable PB-GATT fallback would need a userspace GATT
provisioning implementation over BlueZ's regular GATT D-Bus API.
That means owning provisioning segmentation and cryptography, so it
is materially more code and risk than the PB-ADV daemon route.

## Sources

- [BlueZ Mesh D-Bus API][mesh-api]
- [BlueZ `Management1.AddNode` PB-ADV implementation][manager]
- [BlueZ Mesh daemon source list][make-mesh]
- [BlueZ tool build rules showing deprecated `meshctl`][make-tools]
- [BlueZ `mesh-cfgclient` source and command table][cfgclient]
- [Debian `bluez-meshd` package file list][debian-files]
- [Debian `bluetooth-meshd(8)`][meshd-man]
- [Bluetooth SIG Mesh Protocol 1.1][mesh-spec]

[mesh-api]: https://github.com/bluez/bluez/blob/master/doc/mesh-api.txt
[manager]: https://github.com/bluez/bluez/blob/master/mesh/manager.c#L388-L473
[make-mesh]: https://github.com/bluez/bluez/blob/master/Makefile.mesh
[make-tools]: https://github.com/bluez/bluez/blob/master/Makefile.tools#L339-L384
[cfgclient]: https://github.com/bluez/bluez/blob/master/tools/mesh-cfgclient.c
[debian-files]: https://packages.debian.org/trixie/amd64/bluez-meshd/filelist
[meshd-man]: https://manpages.debian.org/unstable/bluez/bluetooth-meshd.8.en.html
[mesh-spec]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html
