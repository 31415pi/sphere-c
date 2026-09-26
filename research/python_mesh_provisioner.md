# Linux Bluetooth Mesh provisioner options

Research date: 2026-09-26.

## Required capability

For this project, "caller-chosen keys" should mean:

- the caller supplies the NetKey, NetKey index, IV Index, and
  unicast address during provisioning;
- the caller supplies the AppKey during the following Config
  Client exchange; and
- the provisioner returns the newly derived DevKey.

A conforming provisioner does not normally choose an arbitrary
DevKey. The provisioner and device derive it from the ECDH
transcript. A useful implementation must expose that result.

The other required post-provision operation is Config Composition
Data Get, protected with that DevKey.

## Recommendation

Use the MIT-licensed `btmesh` library from
[`ha-bluetooth-mesh`][btmesh] as the first PB-GATT path. Vendor its
small `src/btmesh` package and adapt the repository's
`phase0/provision_and_toggle.py` harness into this project's CLI.

It is the closest existing match to the Sphere C task:

- pure Python, with a direct PB-GATT bearer over Bleak;
- accepts a caller-provided NetKey, key index, IV Index, and
  unicast address;
- exposes the derived DevKey as `Provisioner.device_key`;
- sends Config AppKey Add with a caller-provided AppKey;
- reads and parses Composition Data Page 0;
- works with a normal local Linux adapter, or an ESPHome Bluetooth
  proxy; and
- was built for ThingOS-family lighting and has been validated on
  real Häfele/ThingOS hardware.

This is a code base to adopt, not yet a mature general-purpose
provisioning CLI. It is young, small, and still marked beta. Its
provisioner supports PB-GATT only, the legacy FIPS P-256/CMAC
algorithm, in-band public keys, and No OOB or Static OOB. Those
choices appear to match the Olight evidence already in this repo.

Keep a Zephyr USB-board provisioner as the fallback if the Sphere
only exposes PB-ADV after reset, or if its PB-GATT behavior defeats
the host stack. Zephyr is much more mature and supports both
bearers, but it adds a flashed development board and a serial
control layer.

## Option 1: `btmesh` pure Python

Project: [`dasimon135/ha-bluetooth-mesh`][btmesh]

Status: best fit, with an immaturity warning.

### Evidence

The project describes `src/btmesh` as a Home Assistant-independent,
pure-Python Mesh stack with a provisioner and a GATT bearer over
Bleak. Its README also documents local-adapter and ESPHome-proxy
backends, and says no `bluetooth-meshd` is required.

The implementation is unusually direct evidence, not just a feature
claim:

- `Provisioner(...)` takes `netkey`, `key_index`, `iv_index`, and
  `unicast_addr`.
- It derives `self.device_key` with `k1(..., b"prdk")` before
  sending Provisioning Data.
- `GattBearer(..., provisioning=True)` selects standard Mesh
  Provisioning Data In/Out characteristics.
- The Phase 0 harness saves the DevKey in JSON, reconnects through
  Mesh Proxy, sends Config AppKey Add, fetches Composition Data
  Page 0, parses it, and binds a model.
- The state file contains editable `netkey` and `appkey` fields.

The harness explicitly reports two hardware milestones:
PB-GATT provisioning with a derived DevKey, followed by a
DeviceKey-protected configuration and working control exchange.

### Maintenance and packaging

- Current checkout version: `0.10.3`, dated 2026-09-24.
- Repository history observed: 185 commits.
- PyPI metadata in `pyproject.toml` labels it beta.
- Python requirement: 3.12 or later.
- Runtime dependency: `cryptography>=42`; local BLE adds Bleak and
  `bleak-retry-connector`.
- License: MIT.
- The library is deliberately vendored into the project's own Home
  Assistant integration, which is strong evidence that vendoring is
  an intended deployment pattern.

The package can live in a project virtual environment and use the
host's existing BlueZ service through Bleak. It does not need a Mesh
daemon, root install, or a special radio. Normal Linux Bluetooth
permissions still apply.

### Limits and risks

- PB-GATT only; there is no PB-ADV provisioner bearer.
- The packaged library supplies primitives, not a polished generic
  provisioning command. The complete example is in `phase0/`, which
  the project's source distribution intentionally excludes.
- Only No OOB and Static OOB are implemented. Input, Output, and OOB
  public-key authentication are not.
- It implements the Mesh 1.0-era CMAC suite, not the newer Mesh 1.1
  HMAC-SHA-256 provisioning suite.
- Its author describes provisioning as not yet reachable from the
  Home Assistant UI. That does not affect the library or harness.
- The project has a very small user base. Treat interoperability and
  recovery behavior as something to test on the actual Sphere.
- Composition is obtained only after provisioning by reconnecting to
  the node's Mesh Proxy service. That is appropriate if the reset
  Sphere enables the standard proxy service, but must be verified.

### Adoption shape

Vendor `src/btmesh`, then reduce the Phase 0 harness to:

1. scan service `0x1827` and select the Device UUID;
2. connect PB-GATT and run `Provisioner` with supplied network data;
3. persist the returned DevKey and element count immediately;
4. wait for the node's `0x1828` Mesh Proxy advertisement;
5. add the caller's AppKey, fetch composition, and bind the desired
   SIG or vendor models; and
6. persist the complete state with mode `0600` and crash-safe
   sequence-number handling.

## Option 2: BlueZ `bluetooth-meshd` plus Silvair Python

Projects: [BlueZ Mesh D-Bus API][bluez-api] and
[`python-bluetooth-mesh-bluez`][silvair-bluez].

Status: viable PB-ADV route, but not self-contained.

### Capability

BlueZ's maintained Mesh D-Bus API can:

- create or import a local provisioner identity;
- import a caller-provided NetKey and AppKey;
- scan for unprovisioned devices and call `AddNode`;
- send DeviceKey-protected Config Client traffic;
- export all remote DevKeys with `Management1.ExportKeys`; and
- retain the provisioned remote node in its key database.

The Silvair wrapper provides the provisioner callbacks, subnet and
AppKey import methods, and a typed `get_composition_data()` Config
Client call. Its present `ManagementInterface` does not wrap
`ExportKeys`, even though the BlueZ API exposes it. Adding that one
D-Bus method, or calling it through `dbus-next`, is straightforward.

### Maintenance and packaging

- Silvair released `bluetooth-mesh-bluez` 1.0.1 in June 2026.
- It requires Python 3.14 and is marked beta.
- It is GPL-2.0-only.
- It is pip-installable and can be vendored, but it is only a client
  wrapper. The real Mesh implementation remains `bluetooth-meshd`.

The host must have a compatible BlueZ build with the separate Mesh
daemon, a system D-Bus service, daemon storage, and access to an HCI
adapter. Many distributions do not ship or enable that daemon by
default. Installing or building it, arranging adapter ownership, and
writing its state under `/var/lib/bluetooth/mesh` are system-level
changes. Thus this route does not meet a strict "project-local
install only" requirement.

### Bearer limitation

The current Mesh daemon provisions locally over PB-ADV. BlueZ closed
its PB-GATT daemon request as not planned. PB-GATT belonged to the
older `meshctl` implementation instead. Therefore this is the best
host-side PB-ADV candidate, not a universal GATT/ADV answer.

BlueZ itself is actively maintained, but recent issue reports show
that `bluetooth-meshd` can still have long-running HCI/backend
failure modes. That matters less for a one-shot recovery tool than
for a permanent controller.

## Option 3: Zephyr Bluetooth Mesh shell on a USB board

Project: [Zephyr Bluetooth Mesh shell][zephyr-shell].

Status: most capable and mature fallback; requires radio hardware.

Zephyr's current Mesh stack supports a provisioner over both PB-ADV
and PB-GATT client. The shell exposes commands to:

- create a CDB with an explicit 16-byte NetKey;
- add an explicit AppKey;
- provision by Device UUID over either bearer;
- choose unicast address and NetKey index;
- read Composition Data Page 0; and
- display the CDB.

The CDB API stores a generated DevKey for every provisioned node and
provides `bt_mesh_cdb_node_key_export()` for safely exporting it.
The stock shell documentation does not promise that `mesh cdb show`
prints secret key bytes. A tiny provisioner firmware command should
call the export API and return the DevKey over its USB serial link.

Zephyr is actively maintained, broadly tested, and Apache-2.0
licensed. It is preferable to writing PB-ADV on Linux from scratch.
It does, however, require a supported BLE development board, a Zephyr
build/flash step, and a small serial protocol or shell driver. Once
flashed, the Linux side can remain project-local apart from access to
the USB serial device.

The likely hardware is an nRF52840 DK or dongle. The exact board
choice should be checked against Zephyr's controller and USB-console
support before committing to it.

## Option 4: Nordic Interactive PyACI

Project: [nRF5 SDK for Mesh / Interactive PyACI][pyaci].

Status: functionally useful but obsolete; use only as reference or a
quick PB-ADV experiment with existing Nordic hardware.

PyACI is a Python host talking over serial to Nordic's Mesh Serial
example firmware. Its provisioner:

- loads explicit NetKeys and AppKeys from a JSON database;
- passes the chosen NetKey, index, IV Index, and address to the
  provisioning command;
- receives, logs, and stores the derived `device_key`; and
- includes a Config Client capable of reading composition.

Its Python dependencies are ordinary pip packages and can be kept in
a virtual environment. The repo also includes prebuilt example
binaries. It still requires a compatible nRF52 board and flashing.

The important caveats are decisive for new code:

- the public repository is only a release mirror and rejects pull
  requests;
- the source and copyright trail stop around 2020;
- its documentation targets Python 3.5;
- its provisioner path is PB-ADV; PB-GATT support in that SDK was
  primarily a provisionee/server feature; and
- it depends on the discontinued nRF5 SDK for Mesh/SoftDevice line.

Its permissive BSD-style source is a useful behavioral reference,
especially for JSON state and DevKey capture, but Zephyr supersedes
it as the external-board choice.

## Option 5: legacy BlueZ `meshctl`

Sources: [BlueZ build definition][meshctl-build] and
[Bluetooth SIG's BlueZ 5.50 tutorial][meshctl-tutorial].

Status: technically viable PB-GATT archaeology, not a recommended
dependency.

`meshctl` contains its own GATT Mesh stack and provisions through
the standard `0x1827` service using `bluetoothd`. It persists network
and node data in JSON, including credentials, and can fetch
composition through its Config Client.

It is now built only when BlueZ's deprecated tools are enabled. It is
tightly coupled to BlueZ private source modules and C libraries, so
vendoring it means carrying a fork of an old subsystem rather than a
small standalone binary. Building it also needs BlueZ/GLib/D-Bus,
`json-c`, and Readline development files and may require changes to
the system Bluetooth daemon. It is useful as a packet-flow reference
or last-resort diagnostic, not as the foundation of this project.

## Other candidates considered

### Silicon Labs pyBGAPI

[`pybgapi-examples`][pybgapi] is current and provides Python access
to a Bluetooth Mesh NCP over UART. It requires an EFR32 Wireless
Development Kit flashed with the vendor's Mesh NCP image. The public
Python examples are a host API skeleton; the complete host
provisioner documented in the current SDK is a C example. Host and
target BGAPI/XAPI versions must match.

This is not an open, portable Mesh implementation: the radio stack
and flash workflow depend on Silicon Labs' Gecko SDK and Simplicity
Studio. It offers no advantage over Zephyr for a new open-source
tool.

### Silvair protocol primitives without BlueZ

[`python-bluetooth-mesh-network`][silvair-network] implements Mesh
cryptography, provisioning PDUs, bearer transactions, and test
vectors. It is current, GPL-2.0-only, and requires Python 3.14.
However, it is a protocol library rather than a complete local
PB-GATT or PB-ADV provisioner. Completing its GATT transport and
configuration lifecycle would reproduce work already present in
`btmesh`.

## Decision matrix

| Option | Bearer | Chosen Net/App key | DevKey | Composition |
|---|---|---:|---:|---:|
| `btmesh` | PB-GATT | yes | direct | yes |
| BlueZ + Silvair | PB-ADV | yes | D-Bus export | yes |
| Zephyr board | both | yes | small export hook | yes |
| Nordic PyACI | PB-ADV | yes | direct | yes |
| legacy `meshctl` | PB-GATT | JSON/fork work | stored | yes |

| Option | Extra hardware | System changes | Maintenance |
|---|---:|---:|---|
| `btmesh` | no | no, normal BLE access | active, young |
| BlueZ + Silvair | no | usually yes | active, beta |
| Zephyr board | BLE MCU | flash/serial access | active, mature |
| Nordic PyACI | nRF52 | flash/serial access | frozen |
| legacy `meshctl` | no | usually yes | deprecated |

## Practical go/no-go test

Before integrating any option, factory-reset one Sphere and scan it.

- If it advertises standard service `0x1827`, try the `btmesh`
  Phase 0 harness with an isolated state file and chosen keys.
- Verify the returned element count, derived DevKey, and transition to
  standard Mesh Proxy service `0x1828`.
- Send Composition Data Get before AppKey/model configuration.
- Preserve the raw capabilities, composition, and every key in a
  mode-`0600` state file.
- If no `0x1827` service appears but an unprovisioned beacon is seen,
  move to the Zephyr PB-ADV fallback.
- If neither standard bearer appears, the Olight/Thingclips flow uses
  a vendor transport despite standard Mesh crypto; none of these
  provisioners will work without a transport adapter.

[btmesh]: https://github.com/dasimon135/ha-bluetooth-mesh
[bluez-api]: https://github.com/bluez/bluez/blob/master/doc/mesh-api.txt
[silvair-bluez]: https://github.com/SilvairGit/python-bluetooth-mesh-bluez
[zephyr-shell]: https://github.com/zephyrproject-rtos/zephyr/blob/main/doc/services/connectivity/bluetooth/api/mesh/shell.rst
[pyaci]: https://github.com/NordicSemiconductor/nRF5-SDK-for-Mesh/tree/master/scripts/interactive_pyaci
[meshctl-build]: https://kernel.googlesource.com/pub/scm/bluetooth/bluez/+/master/Makefile.tools
[meshctl-tutorial]: https://www.bluetooth.com/wp-content/uploads/2019/03/Tutorial-How-to-set-up-BlueZ_Part2-3.pdf
[pybgapi]: https://github.com/SiliconLabs/pybgapi-examples
[silvair-network]: https://github.com/SilvairGit/python-bluetooth-mesh-network
