# Olight Sphere C research and local control
Bart Massey and Codex 2026

This repository documents and implements local Linux control of the
USB-C Olight Sphere C. It contains capture-analysis tools, a working
Bluetooth Mesh provisioner, and the beginnings of a persistent Linux
control daemon.

The original magnetic-charge Sphere is treated as a separate device.
Research about it is retained under `docs/research/hardware/`, but its
processor and firmware image have not been identified conclusively.

## Current result

A factory-reset Sphere C has been provisioned into a locally owned
Bluetooth Mesh network without the Olight app or cloud. The device:

- retains its locally assigned keys across disconnect and a
  button-off/wake cycle;
- advertises the expected Mesh Proxy Network ID;
- accepts standard Bluetooth Mesh configuration;
- returns authenticated Generic OnOff status; and
- exposes standard Lightness, CTL, and HSL models, plus two Olight
  vendor models; and
- does not expose a standard xyL Server, despite an earlier model-ID
  misidentification.

The current device address, keys, sequence cursor, captures, and phone
artifacts are deliberately excluded from Git.

## Repository layout

- `src/sphere_mesh/` contains the provisioner, transport compatibility
  layer, capture analysis, and Sphere-specific access messages.
- `bin/sphere-mesh` launches the provisioning and diagnostic harness.
- `scripts/` contains focused inspection and diagnostic utilities.
- `tests/` contains the project unit tests.
- `docs/` contains guides, analysis, plans, and categorized research.
- `config/` contains portable configuration templates.
- `state/` contains live runtime state, logs, and captures. It is
  ignored because it contains active secrets.
- `old/` contains acquired and archival inputs such as the original
  capture, APK, phone export, downloaded packages, and source snapshots.
  The entire directory is ignored.

See [the documentation index](docs/README.md) for the main guides and
research map.

## Setup

Create the environment and install the pinned dependencies:

```console
python3 -m venv .venv
./.venv/bin/pip install -r requirements-dev.txt
```

`requirements.txt` pins the MIT-licensed `btmesh` implementation to the
exact upstream revision used for the hardware work.

## Common commands

Create or validate local key state without using Bluetooth:

```console
./bin/sphere-mesh --init-only
```

Read the already provisioned Sphere without changing its light state:

```console
./bin/sphere-mesh --control read
```

Run several acknowledged commands over one connection:

```console
./bin/sphere-mesh --control read --control on --control off
```

Available commands are `read`, `on`, `off`, and `toggle`. Each command
reports the state acknowledged by the Sphere. Repeating `--control`
keeps the batch on one GATT Proxy connection.

Run the persistent local service and use its separate client:

```console
./bin/sphere-meshd
./bin/sphere-meshctl health
./bin/sphere-meshctl control read
./bin/sphere-meshctl vendor-read 2 102
./bin/sphere-meshctl effect list
./bin/sphere-meshctl effect get
./bin/sphere-meshctl countdown get
```

The daemon owns the state file and Bluetooth connection exclusively,
reconnects automatically, and serves a versioned JSON API on the
mode-`0600` `state/sphere.sock`. See
[`docs/guides/local-service.md`](docs/guides/local-service.md) for the
protocol and systemd user-service setup.

The vendor read shown above queries work mode and color mode without
changing either one. Its result includes decoded enum values and raw
bytes for continued protocol research.

To choose a built-in effect by its schema name, use
`effect select colour_2` (or `effect set colour_2`). This sends only
DP 102 and leaves the current work mode as reported by the device.
`effect activate colour_2` first sets DP 2 to colour mode, then selects
DP 102. Both operations require a DATA acknowledgment for each write;
the response may be an empty acknowledgment, so it does not by itself
prove that the light visibly changed. Only effects shown in the saved
Sphere panel are accepted. A live DP 102 experiment successfully
selected Aurora, read it back, and restored Color Wheel. Visual output
was not observed during that unattended experiment.

The panel countdown uses DP 101. Use `countdown get` to read its
remaining seconds, `countdown set SECONDS` to start or change it, and
`countdown cancel` to set it to zero. `SECONDS` must be an integer from
0 through 43200. A nonzero countdown can turn the lamp off later.
Writes require a vendor acknowledgment. A neutral DP 101=`0` write was
acknowledged and independently read back; positive durations remain
untested.

Read the standard lighting states without changing the output:

```console
./bin/sphere-mesh --lighting lightness \
  --lighting temperature --lighting hsl
```

Set raw Bluetooth Mesh lighting values and read back the acknowledged
state:

```console
./bin/sphere-mesh --lighting lightness=32768
./bin/sphere-mesh --lighting temperature=5000
./bin/sphere-mesh --lighting hsl=32768,0,65535
```

The HSL order is lightness, hue, saturation. Values are unsigned
16-bit integers; temperature is in kelvin. The CLI also understands
xyL messages for nodes that advertise an xyL Server, but reports the
operation as unsupported for the observed Sphere C composition.

Run the project unit tests:

```console
./.venv/bin/pytest -q
```

Analyze the original Android Bluetooth capture:

```console
PYTHONPATH=src ./.venv/bin/python -m sphere_mesh.capture --no-pdus
```

Scan for relevant BLE advertisements:

```console
./.venv/bin/python scripts/scan_mesh.py --seconds 10
```

## Sensitive local artifacts

The ignored `state/sphere_state.json` is authoritative runtime state.
It contains live mesh keys and the next sequence number. Do not delete,
roll back, publish, or edit it while the provisioned Sphere is in use.

`old/` contains private or oversized research inputs. Documentation may
describe their structure and hashes, but the underlying files must not
enter repository history.

## License

This work is made available under the "MIT License". See the
file `LICENSE.txt` in this distribution for license terms.
