# Documentation

## Guides

- [Self-provisioning](guides/self-provisioning.md) records the proven
  factory-reset, provisioning, configuration, and recovery procedure.
- [Key extraction](guides/key-extraction.md) records the phone and app
  investigation path for an existing vendor-owned mesh.
- [Vendor datapoints](guides/vendor-datapoints.md) is the living
  reference for readable and writable Sphere-specific functionality.
- [Local service](guides/local-service.md) documents the daemon, API,
  client, installation, and operational checks.
- [Desktop app](guides/desktop-app.md) covers installation and use of
  the optional Qt/QML control application.

## Analysis

- [Capture analysis](analysis/capture.md) explains the original Android
  Bluetooth snoop capture and the limits of encrypted traffic analysis.
- [Compatibility](analysis/compatibility.md) compares the Sphere C with
  the original Sphere and other relevant devices.
- [Provisioning](analysis/provisioning.md) summarizes Bluetooth Mesh key
  establishment and the available provisioning strategies.
- [Step 6 operational validation](analysis/step6-operational-validation.md)
  records the final API, lifecycle, and live read-only checks.

## Plans

- [Daemon plan](plans/daemon.md) is the staged roadmap for a persistent
  Linux control service.
- [Desktop GUI plan](plans/desktop-gui.md) records the initial design
  and phased scope for the now-implemented desktop client.
- [Project continuation handoff](plans/continuation.md) summarizes the
  current status, safety rules, and next milestones for future work.

## Research

- `research/app/` covers the APK, phone storage, cloud endpoints, and
  vendor provisioning behavior.
- `research/hardware/` covers the original Sphere, Sphere C hardware,
  USB, firmware artifacts, and OTA capture options.
- `research/compatibility/original-sphere.md` gives the current
  original-Sphere compatibility assessment and cautious trial process.
- `research/mesh/` covers Bluetooth Mesh protocol and self-provisioning
  experiments.
- `research/upstream/` covers `ha-bluetooth-mesh`, dependency choices,
  and potential upstream work.
- `research/project/` contains historical project and hygiene audits.

`project/original-task.md` preserves the initial investigation prompt.
