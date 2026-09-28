# Multi-Sphere Support Architecture Audit

## Scope and current shape

The repository already has a device table, but its operational model is
single-selected-device. `state.py:new_state()` stores one mesh network
(NetKey/AppKey, IV Index, provisioner address, sequence cursor, and TID),
plus `devices`, `active_device`, and a `next_unicast` allocator.
`provision.py:provision()` inserts a node record keyed by primary unicast
and advances the allocator past all composition elements. The schema
migration in `migrate_state()` also chooses the last device as active for
older state. Thus provisioning several nodes into one network is partly
anticipated already.

The daemon does not operate that device table as a collection. At startup,
`daemon.py:_active_unicast()` requires one `active_device`; `run_service()`
loads that record's DeviceKey and creates one `ConnectionSupervisor`, one
Mesh Proxy session, and one `SphereRuntime`. `SphereRuntime` stores a single
`_unicast`, serializes every command with one lock, and exposes that single
address in `health()`/`status()`. `status()` reads one top-level
`composition`/`control_model`, which provisioning overwrites each time it
configures a node.

The transport is one physical BLE GATT client at a time in current use.
`transport.py:open_mesh_proxy_session()` creates a MeshNode for a session,
adds one destination DeviceKey/unicast, and persists the network-wide
sequence cursor. The bearer and `ConnectionSupervisor` reconnect the same
chosen target; there is no per-device session manager or simultaneous
connection limit policy. Even if the MeshNode can hold several device
keys, the BLE proxy is the scarce shared link and command dispatch must
still explicitly select a destination.

The provisioning CLI can choose a previously saved record by
`--device-uuid` through `pick_device()`, with active-device preference on
duplicate UUIDs. Provisioning scans multiple beacons but provisions the
first by default and tells the user to pass a UUID to override. The daemon
CLI (`sphere-meshd`) has only `--state`, `--socket`, and `--verbose`; the
client CLI (`sphere-meshctl`) has no device selector or inventory command.
Its requests map directly to methods without device parameters.

The GUI (`gui/backend.py:SphereBackend` and `gui/qml/Main.qml`) has a
single set of power, lighting, effects, and diagnostics properties. It
shows the daemon's one `device` string but offers no selector/list. The
systemd unit starts one daemon with one state file and one control socket.
Packaging in `pyproject.toml`, desktop entry, and icon is application-wide;
multi-device support should not need per-device package artifacts, but
runtime paths and service instance strategy need a decision.

## Cross-cutting design decisions

1. **One shared Mesh network or separate networks?** The current allocator
   and keys naturally support multiple nodes on one subnet. This is the
   least invasive model and allows one proxy bearer to route to all nodes.
   A future multi-network design would require network IDs, keys, IV state,
   source addresses, replay/sequence spaces, and potentially separate
   bearer discovery per network. Keep the first scope to one network unless
   network isolation is an explicit requirement.
2. **One selected connection or concurrent node sessions?** A single BLE
   proxy connection can likely service sequential unicast requests. Keep
   one physical GATT bearer and one network sequence allocator initially;
   model a selected target per request and serialize access. Simultaneous
   GATT connections to multiple spheres should be an evidence-led later
   optimization because radio/proxy connection slots and Sphere advertising
   behavior constrain it.
3. **Identity and selection contract.** Add stable device IDs (primary
   unicast is already unique within this network) and user-facing metadata
   such as name/UUID. Decide whether omitted selection means the persisted
   active device for compatibility, and how stale/deleted selections fail.
   UUID is not sufficient as a key because duplicate UUIDs are explicitly
   tested in `test_provision.py`.
4. **API evolution.** API version is currently 1. Add device inventory and
   selection methods or an explicit `device` parameter on device commands.
   Prefer explicit per-request targeting so independent clients do not
   race by changing global active state. Preserve no-device-parameter
   behavior as an alias for the selected device during a compatibility
   window, and define whether health is network-level or per-device.
5. **Capabilities and persistent node facts.** Move `composition` and
   `control_model` from top-level state into each device record, and define
   optional connection/failure status as volatile runtime information.
   Persist authenticated composition per primary address so GUI/daemon
   startup can report capabilities accurately for every node.

## Required changes by area

### Provisioning and state storage

- Keep network-wide keys, IV Index, sequence cursor, AppKey/NetKey indices,
  provisioner source address, and TID at network scope. Sequence numbers
  and TIDs are sender/network or model transaction concerns; do not create
  per-Sphere network SEQ cursors for nodes on the same subnet.
- Validate `next_unicast` against the unicast range and every device's
  element span; current `validate_state()` checks keys/cursors and active
  membership but not address overlap, element count, or allocated-range
  exhaustion. Store each record's primary address plus element count/range.
- Move `composition`, `control_model`, and `official_config_complete`
  into the relevant `devices[address]` entry. Ensure configuration failure
  on a second Sphere cannot overwrite the first Sphere's capability data.
- Add a stable display name and an explicit rename mechanism if the GUI is
  expected to present a friendly list. Device UUID can identify a device
  for provisioning but can be duplicate, absent, or changed by reset.
- Define reset/removal semantics. `--reset-node` currently sends Node
  Reset but intentionally keeps keys for recovery; deleting a record must
  not rewind `next_unicast` or sequence state and accidentally reuse
  addresses/nonces.
- Keep schema migration backward compatible: lift old top-level
  composition/configuration fields into the existing active device;
  retain `active_device`; preserve keys and sequence exactly. Add tests for
  legacy empty state, one-device migration, multiple existing records,
  invalid overlap, duplicate UUID, and allocator exhaustion.
- Provisioning CLI: support explicit `--device-uuid` selection as today,
  plus inventory/name/selection commands as appropriate. Make default
  behavior safe and visible when multiple advertisements are found; an
  interactive or require-explicit-choice mode avoids silently provisioning
  the wrong nearby Sphere.

### Transport and daemon

- Replace `run_service()`'s fixed `_active_unicast()` setup with a
  network-level session owner that can open one proxy bearer and address
  messages to any stored node. It must load/add each DeviceKey to the
  protocol node or construct the target context per operation, while
  sharing the single monotonically persisted network SEQ cursor.
- Proxy advertisement matching currently takes `node_addresses=(unicast,)`
  and authenticates Node Identity against that one address. For multiple
  targets, accept any configured address and record which address matched;
  Network ID-only matches identify the network, not the destination.
  Decide whether the daemon reconnects to any proxy for the shared network
  then routes to the requested node, or requires a target-verified identity.
- Reconnection/generation/failure must be represented per selected node
  only if node reachability is actually probed. A connected GATT proxy is
  not evidence that every Sphere is awake/reachable. Prefer separate
  network-link health and per-node last-operation/reachability results.
- A shared `SphereRuntime` needs strict command serialization across all
  destinations to preserve ordered BearerPump/network sequence behavior.
  Include the selected device in interaction logs and error details without
  exposing keys or raw vendor payloads.
- `StateStore` currently locks the entire state file, which is suitable
  and desirable for one process managing the whole mesh. Do not create
  independent per-device daemon locks while retaining one sequence cursor.
- Keep the legacy direct `sphere-mesh` path compatible with the same
  network state lock; do not allow daemon and direct client to send from
  the same source address concurrently.

### Socket API and command-line client

- Add a `devices`/inventory operation returning stable IDs, names, primary
  addresses, saved capabilities, and selected state (never keys). Add an
  explicit selection operation only for default-target UX; device-changing
  control methods should accept a device ID/address parameter for
  race-free targeting.
- Extend `health` with network link status and selected/default device,
  or provide per-device status. Clearly distinguish “proxy connected”
  from “this Sphere answered.” Consider a `status(device=...)` response.
- API version 1 is strict and clients require exact version. Either add
  optional parameters/responses compatibly within version 1 or bump the
  version and negotiate/support old clients; tests should lock the chosen
  behavior. Unknown params are generally rejected by `SphereRuntime`.
- Add `sphere-meshctl devices [list|select|rename]` and a shared
  `--device` option for control, lighting, effects, and countdown. Decide
  accepted syntax (address, UUID, or stable opaque ID), default selection,
  and ambiguity reporting.
- Keep socket permission and ownership behavior unchanged. If multiple
  independent daemons are allowed, define separate `--state`/`--socket`
  pairs and avoid accidental socket/path collisions.

### GUI

- Add a device inventory model and selected-device property to
  `SphereBackend`; request inventory on connection and refresh it after
  provisioning/removal/rename. Include target ID in queued requests so
  selection changes cannot retarget already queued writes.
- Add a compact device picker and per-device connection/last-seen
  presentation. Keep one control panel bound to the selected device;
  distinguish daemon's shared proxy state from node availability.
- Decide whether provisioning/onboarding remains CLI-only. The existing
  GUI communicates only over the private daemon socket and has no BLE or
  key access; a GUI provisioning flow would require new API methods and
  progress/cancellation behavior.
- Tests: backend parsing, switching selection while requests are pending,
  inventory refresh, disconnected/unavailable target, and QML smoke or
  packaging checks as applicable.

### systemd and packaging

- A single daemon/state/socket per mesh is simplest; no additional package
  files are required. Document `sphere-meshd` as managing all nodes in its
  state file and expose per-device selection at API/CLI level.
- If multiple mesh networks or isolated daemon instances are required,
  add a systemd template unit (`sphere-mesh@.service`) with instance-scoped
  state/runtime paths and GUI/client socket selection. Avoid multiple
  instances pointing at the same state file (existing exclusive lock
  rejects this, and separate copies risk sequence nonce reuse).
- Update `docs/guides/local-service.md`, `desktop-app.md`, provisioning
  guide, README examples, and daemon plan to explain device inventory,
  selection, default behavior, and one shared state/network.
- Keep packaging tests checking the executable/desktop assets; add unit
  configuration tests only if a template or new files are introduced.

## Test and documentation map

- `tests/test_state.py`: schema migration, per-device metadata,
  address-span validation, allocator rules, stable selection.
- `tests/test_provision.py`: multiple beacon choices, repeated provisioning
  with non-overlapping ranges, UUID ambiguity, active/default selection,
  per-device composition writes.
- `tests/test_transport.py`: proxy matching against multiple configured
  addresses, shared network sequence persistence, request routing to the
  chosen unicast and element address.
- `tests/test_daemon.py` and `tests/test_operational.py`: one network
  supervisor, serialized requests to distinct destinations, inventory and
  targeting API, proxy loss vs node failure semantics, state lock retained.
- `tests/test_api.py`/`test_client.py`: version/parameter contract,
  inventory/selection CLI parsing, ambiguity/unknown-device errors.
- `tests/test_gui_backend.py`/`test_gui_app.py`: inventory and selection
  model, queued-target stability, status and control for two fake devices.
- `tests/test_gui_packaging.py`: retain checks for unchanged desktop
  packaging; add only if paths/assets change.
- Documentation requiring updates: `README.md`,
  `docs/guides/self-provisioning.md`, `docs/guides/local-service.md`,
  `docs/guides/desktop-app.md`, `docs/plans/daemon.md`, and optionally
  `docs/plans/desktop-gui.md`.

## Suggested migration phases

1. **Clarify model and schema.** Commit to one shared network, one BLE
   proxy session, explicit per-request target IDs, and compatibility
   defaults. Add versioned schema migration and per-device composition/
   config metadata with validation. Keep runtime behavior single-target.
2. **Multi-device provisioning and inventory.** Add safe selection among
   beacons, explicit inventory/list/rename, robust unicast span allocation,
   and migrate existing state without touching network key/SEQ/TID values.
3. **Network transport and API routing.** Make daemon load all device keys,
   match the shared network proxy, route each serialized API operation to
   the requested unicast, and report link and node status separately.
   Preserve old default-target calls during API transition.
4. **CLI and GUI.** Add device selection/listing and explicit target flags;
   bind GUI controls to a stable selected ID and guard queued writes.
5. **systemd/docs/operational coverage.** Keep a single systemd daemon per
   network unless isolation is required; document migration and recovery,
   and validate two-device operation with fakes before live hardware.

## Main risks and unresolved choices

- The most consequential protocol risk is SEQ reuse: all nodes on one
  network share one provisioner source and one global SEQ stream. Any
  per-device state split or second daemon with copied keys/cursor can
  reuse a nonce. Retain one locked network state owner.
- Existing records are keyed by address strings with no explicit element
  count. Address collision checks need trustworthy Composition Data and
  must reserve the provisioned element span even before configuration
  succeeds.
- The service currently treats a proxy connection as service readiness.
  Multi-device UI/API semantics must not imply all devices are connected
  merely because one shared Mesh Proxy bearer is up.
- Provisioned Sphere availability and proxy slot count may limit which
  nodes are concurrently reachable. Sequential targeting is safer than
  concurrent GATT sessions and should be the initial acceptance target.
- A device reset/reprovision can produce duplicate UUIDs or a new unicast
  while the old state record remains. Decide whether reset marks a record
  stale, removes it after confirmation, or leaves explicit recovery state.
- Decisions required before implementation: one network vs many; shared
  proxy vs per-device connection supervisors; stable public device ID;
  default target when multiple nodes exist; API versioning strategy;
  whether GUI provisioning is in scope; and whether systemd templates for
  multiple networks are needed.
