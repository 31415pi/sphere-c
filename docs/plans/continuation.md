# Project continuation handoff

Last reviewed: 2026-10-04. This is a restart guide for a future
coding agent. It summarizes the checked-in project and its recorded
hardware evidence; it is not permission to perform live writes,
reset a device, install a service, or publish changes.

## Current state

**Verified in repository records:** This is a Linux/Python control
stack for a self-provisioned Olight Sphere C. Provisioning, standard
Mesh OnOff/Lightness/CTL/HSL access, a persistent local daemon and
client, read-only vendor datapoints, guarded effect/countdown
operations, and an optional Qt/QML desktop client exist. The daemon
owns Bluetooth and the private state; GUI and CLI clients use its
Unix socket. The active physical node in the last recorded session
was `0x0118`.

**Hardware evidence:** Authenticated reads, standard controls,
effect selection/restoration, Aurora activation, neutral countdown
zero, daemon restart/reconnect, and ownership recovery are documented
as exercised on the owner's Sphere C. The 2026-09-27 desktop trial
found CTL changes had no obvious visible effect and one Cotton Candy
activation timed out. The desktop client does not repeat a full
activation; the service may retry only its idempotent effect-selection
step once in the documented timeout case. These observations do not
establish behavior on the original magnetic-charge Sphere.

**Current checkpoint:** The code baseline reviewed is `c6043fc`,
following desktop implementation at `ae4241b`. The pre-firmware
report and reference are complete; no runtime changes were made.
The complete suite passed on 2026-10-04: 235 tests in 12.91 seconds.
The Qt and local-socket tests need an environment
allowed to create Unix sockets; an ordinary restricted sandbox run can
fail for that reason without a source regression.

**Fresh device check:** After the owner's reported sleep/wake,
authenticated reads on 2026-10-04 confirmed power on, Flame
selected, battery 100%, both countdowns zero and sleep false.
DP 2 remained absent. Lightness, HSL and CTL Temperature also
answered. No lighting Set or vendor Write was issued. The
temporary daemon was stopped; do not assume one is running now.

Read [STATUS.md](../../STATUS.md) for the latest summary,
[REPORT.md](../../REPORT.md) for the narrative, and
[SPHERE-C-BT.md](../../SPHERE-C-BT.md) for consolidated tables and
evidence. [PLAN.md](../../PLAN.md) records completed checks and
the remaining phone/video dependencies. The owner has requested
a local documentation commit, not a push or new live experiments.

Treat documented hardware results as historical observations. Before
any live operation, confirm device identity, daemon ownership, current
state freshness, and user intent. Never inspect or publish secrets.

## Re-reading inventory, in priority order

1. `STATUS.md`, `README.md`, `SPHERE-C-BT.md`, and `PLAN.md` —
   current checkpoint, commands, consolidated protocol evidence
   and remaining work. `REPORT.md` supplies the narrative.
2. `docs/README.md` — documentation map and which pages are guides,
   evidence, or plans.
3. `docs/plans/daemon.md` — completed stage boundaries and deferrals;
   especially protocol persistence and vendor-control limits.
4. `docs/plans/desktop-gui.md` and
   `docs/guides/desktop-app.md` — intended UX, implemented behavior,
   known GUI mode/brightness limitations, install details, and
   remaining acceptance work. The guide is the implementation-facing
   source; the plan has older proposed language in places.
5. `docs/guides/local-service.md` — daemon/socket/API contract,
   ownership, error and verification semantics, and systemd notes.
6. `docs/guides/vendor-datapoints.md` plus
   `docs/research/mesh/vendor_dp_inventory.md` — public DP meanings
   and evidence by DP; read before changing vendor behavior.
7. `docs/analysis/step6-operational-validation.md` — automated and
   live checks, including trial observations and test count.
8. `docs/research/mesh/stage2_protocol_risks.md` — historical audit
   and design constraints for sequence, TID, IV, replay, and identity.
   Some gap statements predate current code; reconcile them with
   source and recent commits.
9. `docs/guides/self-provisioning.md` and
   `docs/analysis/provisioning.md` — recovery facts, active node
   history, and destructive provisioning boundaries.
10. `docs/research/compatibility/original-sphere.md` — current
    compatibility assessment and passive-first procedure. Sphere C
    mappings are not transferable by assumption.
11. `docs/analysis/step4-live-validation.md`,
    `docs/research/mesh/step4_sphere_model_map.md`, and
    `docs/research/mesh/step4_codec_audit.md` — model mapping,
    composition-derived elements, and codec evidence.
12. For any vendor response change, also read
    `docs/research/mesh/vendor_dp_response_codec.md`,
    `docs/research/mesh/vendor_type00_runtime_audit.md`,
    `docs/research/mesh/vendor_dp_live_query.md`, and
    `docs/research/mesh/effect_timeout_runtime_audit.md`.
13. For transport or upstream changes, read
    `docs/research/upstream/btmesh_dependency.md`,
    `docs/research/upstream/daemon_transport_extraction.md`,
    `docs/research/upstream/ha_runtime_key_flow.md`, and
    `docs/research/upstream/ha_provision_gap.md`.
14. For UI/source interpretation, read
    `docs/research/app/desktop_gui_source_audit.md`,
    `docs/research/app/phone_effect_mode_semantics.md`, and
    `docs/research/app/sphere_color_mapping_audit.md`.
15. `git log --oneline --decorate -25` gives the implementation
    sequence. Latest relevant commits are `ae4241b` (desktop controls),
    `0668e3c` (production vendor service), `100e762` (effect library),
    and `24c1101` (neutral countdown validation).

The research folders contain more specialized historical material.
Use `docs/README.md` to route into app, hardware, mesh, upstream, and
project topics. The ignored `old/` and `state/` trees are not source
documentation; do not add their contents to Git.

## Files and entry points

- `src/sphere_mesh/provision.py`: CLI provisioner, device setup,
  direct control harness, and state initialization.
- `src/sphere_mesh/state.py`: private state validation, locking, and
  persistence. Treat cursor/key state as security-critical.
- `src/sphere_mesh/transport.py`: adapter discovery, GATT session,
  reconnect supervision, and shared transport lifecycle.
- `src/sphere_mesh/access.py`: SIG access codecs and request helpers.
- `src/sphere_mesh/lighting.py`: model capability lookup and standard
  lighting Get/Set operations.
- `src/sphere_mesh/vendor.py`: Tuya/Olight DP framing, parsing,
  reads, guarded effect and countdown operations.
- `src/sphere_mesh/daemon.py`: exclusive service, live session,
  reconnect lifecycle, command dispatch, and logs.
- `src/sphere_mesh/api.py`: versioned newline JSON Unix socket server.
- `src/sphere_mesh/client.py`: `sphere-meshctl` command-line client.
- `src/sphere_mesh/gui/backend.py`, `app.py`, `qml/`: desktop client.
- `bin/sphere-mesh`, `bin/sphere-meshd`, `bin/sphere-meshctl`:
  development-tree launchers.
- `packaging/systemd/sphere-mesh.service`: example user unit.
- `tests/`: offline unit, API, service lifecycle, GUI, and packaging
  tests. `test_operational.py` uses fake Bluetooth, not hardware.

## Restart and operational commands

The checked-in dependency is pinned in `requirements.txt` and
`pyproject.toml`. Setup from the README:

```console
python3 -m venv .venv
./.venv/bin/pip install -r requirements-dev.txt
./.venv/bin/pip install -e '.[gui]'
```

The GUI extra is optional if working only on service code. Common
read/control commands are:

```console
./bin/sphere-mesh --init-only
./bin/sphere-mesh --control read
./bin/sphere-meshd
./bin/sphere-meshctl health
./bin/sphere-meshctl status
./bin/sphere-meshctl control read
./bin/sphere-meshctl vendor-read 2 102
./bin/sphere-meshctl effect list
./bin/sphere-meshctl effect get
./bin/sphere-meshctl countdown get
./.venv/bin/sphere-mesh-gui
```

`--init-only` validates/initializes local state without Bluetooth; it
does not make a safe replacement for the existing live state. Before
direct CLI use, stop the daemon. Never run two daemons or a daemon and
direct harness concurrently against the same source identity/state.
The README documents mutating commands; consult the guides before
using them.

The documented offline suite command is `./.venv/bin/pytest -q`.
The 2026-10-04 full run passed all 235 tests in 12.91 seconds.
Rerun after relevant changes in an
environment that permits Unix socket creation before claiming a new
suite status. Focused operational tests can be selected with
`./.venv/bin/python -m pytest -q tests/test_operational.py`.
The live installed-entry-point check in the analysis document was
read-only. The systemd unit was validated, but not installed or
enabled. Do not install/enable it as part of ordinary source work.

## Mesh, state, key, and sequence safety

- `state/sphere_state.json` (or the configured state path) contains
  NetKey/AppKey/DeviceKey, IV index, node data, and mutable sequence
  state. It is ignored, private, and authoritative. Never print,
  copy into docs, commit, delete, roll back, or manually edit it.
- The last recorded active node is `0x0118`; older checkpoint/state
  snapshots can have stale address or sequence state. Do not restore
  one verbatim. A stale sequence cursor with the same
  `(NetKey, IV Index, source address)` can reuse an encryption nonce.
- Exactly one process may transmit as a provisioner identity. The
  daemon obtains an exclusive state lock. Never launch a second
  process, another copy of the state, or a systemd service against
  the same identity.
- Each access PDU, SAR segment, proxy filter message, and outbound
  segment acknowledgement consumes a network sequence number. Current
  transport persists the advanced cursor before queueing the PDU;
  state writes fsync the file and containing directory. Audit every
  path and retain failure-injection coverage when changing it. Gaps
  are safe; reuse is not.
- A logical Set has an 8-bit transaction ID. Current state allocates
  and persists a TID before use. Confirm intended transaction-window
  behavior and wrap handling before changing retries; reuse a TID only
  when retrying the same logical Set.
- IV Index is fixed in the current session. Runtime Proxy beacons
  are ignored; network decoding rejects mismatched IVI or failed
  authentication under that fixed index. Beacon-driven IV Update
  detection/recovery is not implemented. Do not copy an advertised
  IV index into state or describe recovery policy as implemented.
- Node Identity advertisements are matched to the active NetKey and
  unicast address in current source. This is stronger than Network ID
  alone. The receive path still lacks a standards-complete persistent
  Replay Protection List, so stale authenticated statuses remain a
  concern.
- Keep the private keys off GUI/client/API responses. API
  verification fields distinguish transport acknowledgment,
  response value, independent readback, and human visual confirmation.
  An empty vendor DATA acknowledgment is not proof of state or effect.
- Provisioning changes durable ownership and model bindings. PB-GATT
  is the tested transport; no OOB authentication is implemented except
  optional static OOB. Reset/reprovision only under an explicit,
  device-specific plan.

## Known limitations and evidence boundaries

- Only Sphere C is provisioned and live-tested. The original
  magnetic-charge Sphere has no authenticated composition or live
  session evidence. Start with passive discovery only.
- The Sphere C has OnOff, Lightness, CTL Temperature, HSL and two
  Olight vendor models. It has no xyL Server; xyL support is generic
  and capability-gated. CTL Temperature produced no obvious visible
  change in the latest GUI trial.
- DP 2 work mode is often omitted from reads. The daemon may only know
  mode from a recent write on the current connection; restart or
  reconnect loses that knowledge. DP 102 selection does not alone
  prove the effect is active.
- Aurora and Red Flash have direct recorded motion evidence. Other
  visible effect names/values come from the saved phone picker and
  offline encoding tests. One Cotton Candy activation timed out in a
  GUI trial; later mode switching worked, so treat the failure as
  intermittent rather than proof that the preset is unsupported.
- DP 101 zero has live acknowledgment/readback. Positive countdowns
  and delayed off behavior are untested. DP 7 is a separate schema
  countdown with live zero reads, not a proven alias for DP 101;
  it has no dedicated local write operation.
- Opaque or behavior-unknown writes remain unavailable for DPs
  33, 37, 38, 104, 105 and 107. DP 104 does have live read evidence.
  DP 106 is a factory-lighting action. DP 8 music requires
  continuous streaming and is deferred. Vendor client model
  `0x07d0:0x0005` is not a control target.
- Effect persistence over a true battery-off cycle is unknown. The
  documented button-off/wake cycle is not a verified hard power cycle.
- Provisioning lacks PB-ADV, output/input OOB, reliable outbound
  segmented retransmission, full IV Update/recovery, and durable RPL.
  Incoming segmented unicast messages are acknowledged/reassembled;
  immediate-duplicate suppression is not durable replay protection.
  Node Identity matching and write-ahead sequence/TID persistence are
  present in current source. See the protocol risk audit for detail,
  while accounting for its older gap statements.
- GUI known constraints include absent countdown/music/service
  management, uncertain mode after daemon restart, opportunistic
  battery refresh, and menu launcher/icon files that need packaging
  outside editable install. GUI guide and current source take
  precedence over earlier plan wording.

## User priorities at pause

The owner deliberately curtailed the project after the desktop app
became usable, then requested the completed report/reference work.
Do not silently revive every deferred experiment.

- First product priority was the complete built-in preset library and
  useful previews in the GUI.
- The owner planned one long video in the displayed effect order so it
  could be segmented and used to distinguish static **Patterns** from
  **Animations**.
- Music mode, positive countdowns, CTL Temperature, and hard power-off
  persistence were explicitly deferred as low-value work.
- Persisting work mode across daemon restarts was acknowledged as a
  limitation and explicitly deferred.
- A preferred future simplification is to ignore missing DP 2 at GUI
  startup: while power is on, treat readable DP 102 as the active
  preset, mapping `colour_7`/Color Wheel to solid-color mode and every
  other visible value to effect mode. This deliberately assumes no
  external controller has left the device in `white`, `scene`, or
  `music`; revisit the rule if those modes become supported.
- Multi-Sphere support was requested as an architecture plan only. Do
  not begin it without a new request.
- Restoring ownership to the phone app and taking it back from the app
  were demonstrated successfully, reducing recovery concern, but this
  is not blanket permission for resets or new physical experiments.

## Deferred work by track

### Resume first: preset capture and GUI previews

- Accept the owner's single continuous video in current grid order.
  Preserve the raw capture under an ignored path such as
  `old/effect-captures/raw/`; do not commit a large private original.
- Prefer a fixed camera, framing, exposure, white balance, and Sphere
  brightness. A useful capture has about 5 seconds of lead-in and
  tail, 20–30 seconds per preset, and visible selection transitions.
- Segment the recording by its known row-major preset order. Confirm
  uncertain cuts with the owner instead of guessing names.
- Classify each entry as a static Pattern or Animation and update
  `docs/research/app/effect_motion_classification.md`.
- Add derived preview assets only after choosing an explicit open
  license for the owner's media and recording attribution. Do not use
  proprietary thumbnails or other assets extracted from the phone app.
- Split the current Effects presentation into **Patterns** and
  **Animations**, retaining search, keyboard access, the responsive
  grid, and a reasonable no-preview fallback.

### Protocol safety before broadening operation

- Retain failure-injection coverage for sequence reservation across
  every PDU path, including proxy configuration and segment ACKs.
- Review TID transaction-window and wrap behavior; allocation and
  persistence before use already exist.
- Implement durable replay protection so stale Status cannot resolve
  a current request.
- Implement IV Update/recovery or retain explicit operator recovery;
  design authenticated-beacon handling and retain rejection of
  messages that cannot authenticate under the current state.
- Add crash/failure injection at reserve, enqueue, send, reconnect,
  and restart boundaries. Test hard power-loss assumptions separately.

### Desktop quality and packaging

- If startup mode reporting becomes important, persist the last
  acknowledged mode per device and expose its source/age. When no
  direct or persisted mode exists, show “preset selected · mode
  unknown” rather than treating unknown as ordinary Light mode.
- Complete GUI accessibility, keyboard, screen reader, theme, and
  HiDPI validation called out in the original acceptance criteria.
- Resolve per-user/distribution installation of `.desktop` entry and
  icon; editable installs put these under the virtual environment.
- Catalog counts are reconciled in `SPHERE-C-BT.md`: 32 schema
  values, 26 panel choices and 25 desktop-grid entries. Confirm
  current UI behavior against source before extending it.
- If the intermittent Cotton Candy timeout recurs, diagnose it through
  readback and structured logs. Do not infer state from acknowledgment
  absence or repeat the whole activation automatically.

### Targeted protocol features

- Evaluate positive countdown only as a deliberately timed live
  experiment with explicit cancellation and readback.
- Test effect behavior after a true battery-off cycle only if there is
  a user-relevant need and a safe restore plan.
- Consider music mode in its own design: continuous phone-microphone
  stream, disconnect handling, and recovery policy.
- Investigate host-side animation only after confirming required
  vendor payload semantics, segmentation reliability, and whether it
  serves an actual requirement.

### Portability and multi-device

- Original Sphere compatibility: passive BLE/GATT discovery first;
  use that device's own credentials and composition. No reset or
  provisioning as a discovery shortcut.
- Multi-Sphere support needs address/key/state isolation and a clear
  identity model. See `docs/research/architecture/multi-sphere-support.md`.
- Upstream reusable fixes or Home Assistant integration are separate
  workstreams; inspect the upstream research inventory first.

### Firmware acquisition preparation

The [OTA capture plan][ota-plan] now has an exact-device checklist,
private manifest template and ranked analysis questions. Runtime
hooks and export validation still need a prepared phone and an
agreed workflow. Do not consume the boxed original Sphere's
possible update offer during tooling preparation. Any resulting
original-Sphere image remains evidence for that generation.

[ota-plan]: ../research/hardware/original_sphere_ota_capture_plan.md

## Suggested milestones

1. Ingest and segment the planned effect video, classify all presets,
   establish licensing/attribution, and add preview assets plus the
   Patterns/Animations split.
2. Rerun the complete suite in an environment where Unix socket
   creation is permitted. Record genuine failures separately from
   sandbox restrictions.
3. Finish desktop accessibility and installation validation using a
   fake daemon for automated flows; use the real device only for
   already-validated, reversible controls.
4. For protocol expansion, close the relevant state gap with focused
   design notes, code, and fault-injection tests. Do not use the live
   Sphere as a substitute for deterministic tests.
5. Choose one user-valued live experiment, write a preflight/restore
   plan, capture before/after reads, acknowledgment, independent
   readback, and visible result, then update the evidence ledger.
6. Reassess original-device and multi-device work only after the
   single-device safety and identity boundaries are clear.

## Live experiments requiring the user

These require the owner's physical presence, an explicit action plan,
or user acceptance of a device-visible effect. Do not infer consent
from access to this checkout.

- Positive DP 101 countdown: can switch the lamp off later. Agree on
  a short duration, immediate cancellation path, and observation
  window first.
- True battery-off effect persistence: requires a real power cycle
  and a before/after state plan; previous evidence is only button-off/
  wake.
- Additional effect trials or the Cotton Candy timeout diagnosis:
  inspect and read back state first, and restore
  the intended effect only if the owner approves.
- Hidden/opaque datapoint, music, factory-light reset, node reset,
  re-provisioning, or any original-Sphere operation: all require a
  new narrowly scoped plan. DP 106 and reset operations are especially
  consequential and should remain out of routine experiments.
- Rooted phone/app key extraction or network/account operations:
  only on an explicitly authorized disposable device/account
  environment. The APK/app research points to possible paths but does
  not establish live possession of credentials.

## Related external work

The dependency is pinned to `ha-bluetooth-mesh` commit
`a8f7bbbebaeba124d7d08cb16122055074f2ef46`; see
`docs/research/upstream/` before proposing changes.

There is also a related checkout at
`/local/src/old/ha-bluetooth-mesh`. On 2026-09-27 it was on `main`
tracking `origin/main`, with local branches
`fix/gatt-characteristic-selection` (`15878d5`),
`fix/fresh-ble-scans` (`4bc80a2`), and
`feat/config-network-transmit` (`a15c857`). These commits have not
been pushed by this project. Its untracked `PROVISION.md` is a
Home Assistant provisioning rewrite handoff that describes issue #12,
those experimental branches, and the separate Sphere C hardware
reference. Treat it as valuable but untracked user work: read it before
upstream/integration tasks and do not overwrite or commit it without
direction. The checkout also has remote `upstream` branches; inspect
its current state before relying on these observations.

The project research also references Home Assistant's
`ha-bluetooth-mesh` integration and Tuya Local's public Sphere C
profile. These are reference/integration leads, not verified
compatibility claims. External URLs and sources are recorded in
`docs/research/compatibility/original-sphere.md` and
`docs/research/upstream/`.

## Evidence labels for future edits

Mark conclusions as **Verified** only when supported by a test or a
recorded live observation. Use **Inference** for conclusions derived
from schema, APK code, or protocol expectations. Mark hardware changes
and UI effects **Untested** until directly observed. For every live
write, record request bytes/operation, acknowledgment, independent
readback, visible result, and restoration separately. Update the
relevant guide and evidence ledger together.
