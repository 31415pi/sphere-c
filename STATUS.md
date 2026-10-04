# Sphere C project status

Snapshot date: 2026-10-04

This is the project status and restart summary. For the detailed
restart guide, read [the continuation handoff][continuation].
The investigation narrative is in [REPORT.md](REPORT.md), and
the consolidated protocol reference is
[SPHERE-C-BT.md](SPHERE-C-BT.md).

[continuation]: docs/plans/continuation.md

## Repository state

- Code baseline reviewed: `c6043fc`; this documentation work
  changes no runtime implementation.
- Latest full test run: **235 passed in 12.91 seconds** on
  2026-10-04, using `./.venv/bin/pytest -q` with Unix-socket access.
- Synthetic message vectors, all 32 preset catalog rows, document
  links, table structure and the OTA manifest were checked.
- The owner requested committing the report, reference, plan and
  reconciled documentation, including this status summary.
  Push permission has not been given.
- Use `git status` and `git log` for current branch/commit state;
  this snapshot does not assert a live remote state.

## Documentation milestone

- [REPORT.md](REPORT.md) explains the path from encrypted capture
  to local provisioning, vendor controls and the desktop app.
- [SPHERE-C-BT.md](SPHERE-C-BT.md) records transport, composition,
  messages, all product DPs and presets, with evidence limits.
- [PLAN.md](PLAN.md) records completed work and the remaining
  phone/video dependencies.
- The [OTA capture plan][ota] now includes exact-device preflight,
  a private artifact manifest and ranked analysis questions.
- Guides have current retry/motion wording. Historical audits
  retain their chronology with pointers to superseding results.

[ota]: docs/research/hardware/original_sphere_ota_capture_plan.md

## What the project now does

This repository locally controls a USB-C Olight Sphere C without the
Olight app or cloud.

Implemented and tested components include:

- PB-GATT provisioning into a locally owned Bluetooth Mesh network;
- standard Mesh configuration and model binding;
- authenticated Generic OnOff, Lightness, CTL, and HSL operations;
- persistent key, sequence-number, and transaction-ID state;
- proxy discovery, authenticated Node Identity matching, reconnects,
  and a supervised persistent Bluetooth session;
- a private Unix-socket daemon API and `sphere-meshctl` client;
- Tuya/Olight vendor datapoint reads;
- the complete named DP 102 preset catalog;
- guarded preset activation and DP 101 countdown operations;
- structured verification and partial-failure reporting;
- optional JSON interaction tracing through daemon debug logging; and
- an optional PySide6/QML desktop application.

The desktop application currently provides:

- power control;
- logarithmic brightness control;
- phone-compatible HSV color control over Mesh HSL;
- a responsive, searchable grid of 25 visible presets, excluding the
  special Color Wheel entry;
- battery display when DP 103 is available; and
- connection, capability, error, and diagnostic information.

Color Wheel is treated internally as DP 102 value `colour_7` and is
used as the solid-color mode rather than appearing in the preset grid.

## Physical-device status

The last recorded active Sphere C node is `0x0118`. Its private keys,
sequence cursor, captures, and other sensitive artifacts are ignored by
Git under `state/` and `old/`.

The following have live evidence on the physical Sphere C:

- local provisioning and configuration;
- ownership persistence across disconnect and button-off/wake;
- authenticated power and standard lighting reads/writes;
- daemon reconnect and restart behavior;
- effect selection and restoration;
- visible Aurora animation;
- visible Red Flash motion;
- neutral countdown value zero;
- restoring the Sphere to the phone app and subsequently taking local
  ownership again; and
- USB charging with no USB data device observed during the limited
  port investigation.

The original magnetic-charge Sphere remains a separate, untested
device. Do not reset or provision it merely to test compatibility.

### Read-only wake check: 2026-10-04

After the owner reported sleeping and waking the Sphere C, the
daemon was started with the existing state. Authenticated reads
confirmed power on, Lightness `1956`, HSL `(4358,19660,65535)`,
and CTL Temperature `18900 K` with delta-UV zero. The vendor
snapshot reported Flame selected, battery 100%, both countdowns
zero and sleep false; DP 2 was omitted again.

The check made no lighting Set, vendor Write, reset or provisioning
request. Proxy setup and normal sequence persistence occurred.
The temporary daemon was stopped cleanly afterward. This proves
reachability after the reported wake, not hard power-loss
persistence, active work mode or visual Flame behavior.

## Important protocol facts

The Sphere C exposes standard OnOff, Lightness, CTL, and HSL models,
plus Olight/Tuya vendor models. It does not expose a standard xyL
Server.

Relevant vendor datapoints include:

- DP 2: `work_mode` (`white`, `colour`, `scene`, or `music`);
- DP 5: static HSV color data, translated by the app stack to HSL;
- DP 6: generic Tuya `scene_data`;
- DP 8: microphone-driven `music_data` stream;
- DP 101: the countdown used by the product panel;
- DP 102: the built-in preset selector;
- DP 103: battery percentage; and
- DP 104: sleep flag.

DP 102, not DP 6, selects the built-in Sphere presets. The saved Sphere
panel has no discovered DP 6 call site. `scene` mode and DP 6 have not
been live-tested and may be unused generic Tuya functionality.

DP 2 is frequently omitted from vendor read responses, including
focused reads requesting DP 2 alone. DP 102 is reliably readable, but
the current daemon cannot always prove whether that remembered preset
is active after a daemon restart.

The user proposed the following future GUI rule, which is documented
but **not implemented**:

- when the Sphere is on, treat readable DP 102 as the active state;
- map Color Wheel to solid-color mode; and
- map every other visible DP 102 value to its active preset.

This deliberately assumes that no external controller has left the
Sphere in `white`, `scene`, or `music` mode. It is considered reasonable
for the current single-controller feature set.

## Current limitations

- After a daemon restart, missing DP 2 can cause the GUI to display
  ordinary “Light on” even though a DP 102 preset is visibly active.
- The proposed power-plus-DP-102 startup rule is deferred.
- Only Aurora and Red Flash currently have direct motion
  classification evidence. The other presets still need observation.
- One Cotton Candy activation timed out during a GUI trial, while later
  mode switches worked. Treat this as intermittent, not unsupported.
- CTL Temperature produced no obvious visible effect and is omitted
  from the GUI.
- Positive countdown values and delayed-off behavior are untested.
- Music mode was deliberately deferred because the phone implementation
  continuously streams microphone-derived data and performs poorly.
- DP 6 scenes, opaque timer/state datapoints, factory-light behavior,
  and per-LED addressing are unsupported or unverified.
- There is no known individual-LED or pixel-buffer command.
- Provisioning lacks PB-ADV and output/input OOB support.
- Outbound segmented retransmission, full IV Update recovery, and a
  durable Replay Protection List remain incomplete.
- Incoming segmented unicast messages are reassembled and
  acknowledged by the pinned dependency. Its immediate-duplicate
  suppression is not durable replay protection.
- Runtime Proxy beacons are ignored. Network decoding checks IVI
  and authenticates using a fixed IV Index; it does not implement
  beacon-driven IV Update detection/recovery.
- Editable GUI installation does not automatically place the desktop
  launcher and icon in the user's application menu.
- Multi-Sphere operation is designed but not implemented.

## Remaining priorities

The owner curtailed feature work after reaching a usable desktop
application, then requested the pre-firmware documentation now
completed. The next product priority remains the visual library.

If work resumes, the first product priority is the full visual preset
library:

1. Accept one long user-recorded video in the current preset-grid order.
2. Segment it into individual preset clips.
3. Classify each preset as a static **Pattern** or an **Animation**.
4. Establish an explicit open license and attribution for the user's
   derived preview media.
5. Add previews to the GUI.
6. Split the present Effects view into Patterns and Animations while
   retaining the responsive grid, search, keyboard access, and fallback
   behavior.

Raw video should remain in an ignored location such as
`old/effect-captures/raw/`. Do not use proprietary images or animations
extracted from the phone app.

Firmware acquisition preparation is documented, but runtime hooks
and phone-to-host export checks are not tested. They need a
prepared phone and an agreed device/account workflow. Preserve
the original Sphere's possible one-time update opportunity until
that preparation is complete.

The following were explicitly deferred or deprioritized:

- persisting or otherwise repairing startup work-mode knowledge;
- Music mode;
- positive countdown experiments;
- true battery-off persistence tests;
- CTL Temperature;
- speculative opaque vendor datapoints;
- original-Sphere active experiments; and
- implementing multi-Sphere support.

## Multi-Sphere status

Only an architecture audit exists. The recommended eventual design is:

- one shared Mesh network;
- one daemon and one globally owned sequence-number stream;
- one shared proxy connection initially;
- per-device composition, capabilities, identity, and friendly name;
- explicit per-request device targeting in the API; and
- a GUI device picker bound to stable device IDs.

Do not split a shared Mesh network across copied state files or multiple
daemons. That could reuse sequence numbers and encryption nonces. Read
`docs/research/architecture/multi-sphere-support.md` before doing any
multi-device work.

## Safety rules

- `state/sphere_state.json` is authoritative and contains live secrets.
  Never print, publish, commit, delete, roll back, or manually edit it.
- Never run two transmitters using the same provisioner identity and
  state. Stop the daemon before using the direct control harness.
- Sequence-number gaps are safe; reuse is not.
- Do not restore an old state snapshot with a stale sequence cursor.
- Do not infer visible success from an empty vendor acknowledgment.
- Distinguish transport acknowledgment, reported value, independent
  readback, and human visual verification.
- Device reset, re-provisioning, factory actions, original-Sphere work,
  and uncertain live writes require a new explicit plan with the user.
- Do not commit anything unless the user explicitly requests it, and
  never push without explicit permission.

## How to run it

Install the development and GUI dependencies:

```console
python3 -m venv .venv
./.venv/bin/pip install -r requirements-dev.txt
./.venv/bin/pip install -e '.[gui]'
```

Start the daemon and GUI from the checkout:

```console
./bin/sphere-meshd
./.venv/bin/sphere-mesh-gui
```

Useful read-only service commands include:

```console
./bin/sphere-meshctl health
./bin/sphere-meshctl status
./bin/sphere-meshctl control read
./bin/sphere-meshctl vendor-read 2 102
./bin/sphere-meshctl effect list
./bin/sphere-meshctl effect get
./bin/sphere-meshctl countdown get
```

Run the complete test suite with:

```console
./.venv/bin/pytest -q
```

Qt and API tests require permission to create local Unix sockets.

## Read these first

For a future implementation session, read in this order:

1. `STATUS.md`
2. `README.md`
3. `SPHERE-C-BT.md` and `REPORT.md`
4. `PLAN.md` and `docs/plans/continuation.md`
5. `docs/guides/desktop-app.md`
6. `docs/guides/local-service.md`
7. `docs/guides/vendor-datapoints.md`
8. `docs/research/app/effect_motion_classification.md`
9. `docs/research/app/phone_effect_mode_semantics.md`
10. `docs/research/mesh/vendor_dp_inventory.md`
11. `docs/research/mesh/vendor_dp_response_codec.md`

Then select specialized research through `docs/README.md`.

Principal source entry points are:

- `src/sphere_mesh/daemon.py`
- `src/sphere_mesh/transport.py`
- `src/sphere_mesh/state.py`
- `src/sphere_mesh/vendor.py`
- `src/sphere_mesh/lighting.py`
- `src/sphere_mesh/provision.py`
- `src/sphere_mesh/gui/backend.py`
- `src/sphere_mesh/gui/qml/Main.qml`

## Related checkout

The related checkout `/local/src/old/ha-bluetooth-mesh` previously had
three local feature branches:

- `fix/gatt-characteristic-selection` at `15878d5`;
- `fix/fresh-ble-scans` at `4bc80a2`; and
- `feat/config-network-transmit` at `a15c857`.

Its `main` branch also had an untracked `PROVISION.md` describing a
future Home Assistant provisioning rewrite. Reinspect that checkout
before relying on this snapshot. Do not overwrite or commit its
untracked document without direction.

## Historical implementation and handoff commits

- `c6043fc` — documented deferred startup mode rule
- `1949cd1` — documented project continuation
- `ae4241b` — implemented desktop Sphere controls
- `0668e3c` — completed production vendor service
- `100e762` — completed named effect library

No additional runtime implementation is in progress. The report
and protocol-reference milestone is documentation-only.
