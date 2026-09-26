# Olight Sphere C research and local control

This repository documents and implements local Linux control of the
USB-C Olight Sphere C. It contains capture-analysis tools, a working
Bluetooth Mesh provisioner, and the beginnings of a persistent Linux
control daemon.

The original magnetic-charge Sphere is treated as a separate device.
Research about it is retained under `research/`, but its processor and
firmware image have not been identified conclusively.

## Current result

A factory-reset Sphere C has been provisioned into a locally owned
Bluetooth Mesh network without the Olight app or cloud. The device:

- retains its locally assigned keys across a power cycle;
- advertises the expected Mesh Proxy Network ID;
- accepts standard Bluetooth Mesh configuration;
- returns authenticated Generic OnOff status; and
- exposes standard Lightness, CTL, HSL, and xyL models, plus two
  Olight vendor models.

The current live-device address, keys, sequence cursor, captures, and
phone artifacts are deliberately excluded from Git.

## Important files

- `sphere_mesh` launches the provisioning and diagnostic harness.
- `provision_sphere.py` implements provisioning and configuration.
- `sphere_transport.py` owns reusable BLE and Mesh Proxy sessions and
  the small hardware compatibility layer over upstream `btmesh`.
- `sphere_access.py` implements one standard configuration message
  not yet present in the pinned upstream release.
- `analyze_capture.py` analyzes Android Bluetooth snoop captures.
- `DAEMON_PLAN.md` records the six-stage control-daemon roadmap.
- `SELF_PROVISION.md` records the successful provisioning procedure.
- `research/` contains focused protocol, hardware, APK, and firmware
  notes.
- `requirements.txt` pins the MIT-licensed mesh implementation to an
  exact upstream Git commit.

## Common commands

Create or validate local key state without using Bluetooth:

```console
./sphere_mesh --init-only
```

Read the already provisioned Sphere without changing its light state:

```console
./sphere_mesh --toggle-only --no-toggle
```

Create the environment and install the pinned dependencies:

```console
python3 -m venv .venv
./.venv/bin/pip install -r requirements-dev.txt
```

Run the project unit tests:

```console
./.venv/bin/pytest -q test_*.py
```

## Sensitive local artifacts

The ignored `mesh-lab/sphere_state.json` is authoritative runtime
state. It contains live mesh keys and the next sequence number. Do not
delete, roll back, publish, or edit it while the provisioned Sphere is
in use.

Packet captures, APKs, phone extracts, and downloaded Debian packages
also remain local. Research notes may describe their structure and
hashes without placing the underlying private or oversized artifacts
in repository history.
