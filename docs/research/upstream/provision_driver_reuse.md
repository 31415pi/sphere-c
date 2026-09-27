# Provisioning driver reuse for upstream issue #12

Date: 2026-09-26

## Scope and source baseline

This compares the Sphere harness at repository commit `048f972` with
the upstream checkout at:

```text
/local/src/old/ha-bluetooth-mesh
commit a8f7bbbebaeba124d7d08cb16122055074f2ef46
```

The latter is also the exact revision pinned by `requirements.txt`.
The relevant upstream files are:

- `src/btmesh/provisioner.py`
- `src/btmesh/pump.py`
- `src/btmesh/bearer.py`
- `phase0/provision_and_toggle.py`

[Issue #12](https://github.com/dasimon135/ha-bluetooth-mesh/issues/12)
is titled “Provisioning is unreachable from Home Assistant.” Its body
asks for five things:

1. scan for unprovisioned `0x1827` advertisements;
2. drive `Provisioner` through a provisioning `GattBearer`;
3. allocate a collision-free unicast range;
4. add the resulting node to the stored `.connect` document; and
5. perform AppKey Add and Model App Bind for driven models.

The issue is not asking for a new cryptographic implementation. The
state machine is already complete and tested; the missing part is a
callable asynchronous session driver plus Home Assistant workflow and
storage integration.

## Bottom line

The Sphere repository does now contain the first four layers needed to
run a real PB-GATT provisioning exchange:

1. `sphere_mesh.transport.scan_unprovisioned()` discovers a fresh target.
2. `sphere_mesh.transport.open_gatt_session()` connects, creates a
   provisioning bearer, and subscribes with retry.
3. upstream `BearerPump` serializes synchronous state-machine output
   onto the asynchronous bearer.
4. upstream `Provisioner` executes Invite through Complete and derives
   the DeviceKey.

The function that composes those layers into a complete live exchange
is `sphere_mesh.provision.provision()` at lines 343-493. It is still a CLI
harness function, not an upstream-quality library API. The reusable
core is concentrated in lines 393-473. Scanning, console output,
Sphere state mutation, and diagnostics wrap that core.

For issue #12, the main upstreamable result is therefore not
`Provisioner` itself. It is an extracted asynchronous provisioning
session object/function based on:

- `sphere_mesh.transport.GattBearer` (lines 34-71);
- `sphere_mesh.transport.disconnect_quietly` (lines 185-190);
- `sphere_mesh.transport.open_gatt_session` (lines 193-229); and
- the dispatcher, pump, completion/error wait, and teardown in
  `sphere_mesh.provision.provision` (lines 393-479).

The revised scan behavior in `sphere_transport` is also independently
worth upstreaming. The post-provisioning configuration code is useful
as a behavioral prototype, but only part of it is generic.

## Exact live provisioning call path

The end-to-end call path in the Sphere CLI is:

```text
async_main()                                      src/sphere_mesh/provision.py:910
  -> transport.start()                           src/sphere_mesh/provision.py:929
  -> provision()                                 src/sphere_mesh/provision.py:343
     -> transport.scanner()                      src/sphere_mesh/provision.py:346
     -> scan_unprovisioned()                     src/sphere_mesh/provision.py:350
        -> _fresh_discoveries()                  src/sphere_mesh/transport.py:74
        -> upstream advertisement parser
     -> open_gatt_session(provisioning=True)      src/sphere_mesh/provision.py:405
        -> transport.connect()
        -> local GattBearer()                    src/sphere_mesh/transport.py:216
        -> GattBearer.start(dispatch)
     -> BearerPump(... PROVISIONING_PDU)          src/sphere_mesh/provision.py:410
     -> Provisioner(... send=pump.put)            src/sphere_mesh/provision.py:414
     -> pump.start()                              src/sphere_mesh/provision.py:444
     -> Provisioner.start()                      src/sphere_mesh/provision.py:445
     -> GATT notify -> dispatch -> on_message
        -> Provisioner.handle_pdu()               src/sphere_mesh/provision.py:438
     -> Provisioner.on_done -> asyncio.Event      src/sphere_mesh/provision.py:430
     -> wait for Complete/error/timeout           src/sphere_mesh/provision.py:447
     -> stop pump, bearer, and client             src/sphere_mesh/provision.py:468
     -> persist DeviceKey and advance address     src/sphere_mesh/provision.py:473
```

The actual protocol conversation remains entirely upstream in
`Provisioner`:

- constructor and state dispatch:
  `src/btmesh/provisioner.py:141-213`;
- Invite:
  `src/btmesh/provisioner.py:232-238`;
- PDU decode, state validation, and fatal failure transition:
  `src/btmesh/provisioner.py:242-265`;
- capabilities/auth selection and Start/PublicKey:
  `src/btmesh/provisioner.py:269-313`;
- ECDH and Confirmation:
  `src/btmesh/provisioner.py:315-346`;
- confirmation/random validation and encrypted provisioning data:
  `src/btmesh/provisioner.py:348-388`; and
- terminal Complete callback:
  `src/btmesh/provisioner.py:390-393`.

`BearerPump` is likewise already upstream. Its queue is essential: a
synchronous `Provisioner` can emit multiple PDUs while
`GattBearer.send()` is asynchronous, and concurrent sends would
interleave SAR frames. See `src/btmesh/pump.py:24-68`.

## What changed from the old Phase 0 harness

`sphere_mesh.provision.provision()` is almost the same protocol driver as
`phase0/provision_and_toggle.py:373-517`. Its meaningful differences
are lifecycle and extraction improvements rather than a new protocol:

- `LocalTransport`, quiet disconnect, and GATT session setup moved out
  of the script into `src/sphere_mesh/transport.py`.
- scanning can return without stopping the scanner. The caller keeps
  it running until the connection attempt finishes, then stops it in a
  `finally` block (`src/sphere_mesh/provision.py:404-409`). This preserves the
  just-observed target during the short Telink connectability window.
- only advertisements delivered during the current scan are accepted,
  instead of blindly reusing a scanner cache
  (`src/sphere_mesh/transport.py:74-89,92-119`).
- local `GattBearer` chooses characteristic objects within the intended
  service, avoiding ambiguous duplicate characteristic UUIDs
  (`src/sphere_mesh/transport.py:34-71`).
- state is stored privately and atomically, under `state/`, and the
  first assigned address is Sphere policy `0x0100`. These are not
  protocol-driver changes.

The old harness already contained `open_gatt_session()` at lines
308-349 and essentially the same provisioning composition at lines
422-495. The Sphere work makes these pieces more reusable and fixes
scanner/characteristic behavior, but it has not yet made the
provisioning composition itself a library abstraction.

## Generic reusable logic to upstream

### 1. Service-scoped GATT characteristic selection

`sphere_mesh.transport._characteristic_for_service()` and its
`GattBearer` subclass (`src/sphere_mesh/transport.py:34-71`) are generic BLE
Mesh behavior, not Sphere behavior.

Upstream `GattBearer.__init__()` stores UUID strings only
(`src/btmesh/bearer.py:117-123`). If a backend or device exposes the
same characteristic UUID more than once, UUID-only lookup can bind the
wrong instance. The local subclass searches only the selected Mesh
Provisioning or Mesh Proxy service and passes the characteristic object
to bleak. It deterministically chooses the lowest handle if a service
itself has duplicates.

This should be folded into upstream `GattBearer`, not retained as a
Sphere subclass. It benefits both provisioning and runtime proxy
connections. Local coverage exists at
`tests/test_provision.py:51-78`, although that test currently exercises
the proxy pair rather than the provisioning pair.

### 2. Fresh-scan collection and caller-owned scan lifetime

`sphere_mesh.transport._fresh_discoveries()`
(`src/sphere_mesh/transport.py:74-89`) and the revised
`scan_unprovisioned()` (`src/sphere_mesh/transport.py:92-119`) are generic.

Two changes matter:

- a detection callback builds a per-scan discovery set, so stale cached
  advertisements cannot be mistaken for a currently connectable node;
- `stop_after=False` lets a caller keep scanning until it has connected,
  then explicitly stop the scanner.

Upstream `scan_unprovisioned()` reads the scanner's possibly stale
cache and always stops before returning
(`src/btmesh/bearer.py:361-396`). The local behavior is covered by
`tests/test_transport.py:195-257`.

The same fresh-scan change appears in local `find_proxy_node()`
(`src/sphere_mesh/transport.py:122-164`), with coverage at
`tests/test_transport.py:259-279`. It is reusable, but proxy discovery
is adjacent to issue #12 rather than the PB-GATT driver itself.

An upstream API should preferably express scan ownership as a context
manager/session rather than a boolean whose caller must remember to
stop. At minimum, document that `stop_after=False` transfers cleanup
responsibility to the caller.

### 3. Retrying connect-and-subscribe as one operation

`sphere_mesh.transport.open_gatt_session()`
(`src/sphere_mesh/transport.py:193-229`) is generic. It treats connection plus
notification subscription as the unit of success. On a connect or
subscribe failure it establishes a fresh link and retries, disconnecting
the failed client first.

This was already proven in the old harness at
`phase0/provision_and_toggle.py:308-349`; extracting it is the useful
change. Tests cover connect retry, subscribe cleanup, exhaustion, and
invalid attempt count at `tests/test_transport.py:43-120`.

The function is useful for provisioning and proxy sessions. It belongs
either beside `GattBearer` or behind a small transport/session
abstraction in `btmesh`, not in a device-specific script.

### 4. An asynchronous provisioning-session driver

The code in `sphere_mesh.provision.provision()` lines 393-479 should be
extracted instead of upstreaming the entire function.

Its generic responsibilities are:

- install a temporary receive dispatcher before subscription;
- create a provisioning `GattBearer`;
- create `BearerPump` with `MSG_TYPE_PROVISIONING_PDU`;
- construct `Provisioner` from the network parameters;
- route only provisioning messages to `Provisioner.handle_pdu()`;
- turn synchronous `on_done` and pump failure callbacks into an
  awaitable result;
- apply a bounded timeout; and
- always stop the pump, unsubscribe, and disconnect.

A reasonable upstream surface would return a value such as:

```text
ProvisioningResult(
    device_key,
    capabilities,
    unicast_address,
)
```

and accept explicit inputs such as a connected device/transport,
NetKey, NetKey index, IV Index, address selection, static OOB, and a
timeout. It should raise typed provisioning, bearer, and timeout errors.
It should not know about argparse, JSON, console messages, Sphere model
IDs, or `.connect` persistence.

The current callback list (`session_handler`) is a small race-avoidance
shim: subscription must be installed before `Provisioner` exists, but
incoming notifications need to reach it once constructed. This pattern
is reusable, though a session class can represent it more clearly.

There is no direct unit test for this orchestration today. Upstreaming
should add a fake-bearer transcript test that proves successful
Invite-to-Complete flow plus connect failure, pump failure, protocol
failure, timeout, cancellation, and cleanup.

### 5. Proxy session lifecycle for post-provisioning configuration

`sphere_mesh.transport.MeshProxySession` and
`open_mesh_proxy_session()` (`src/sphere_mesh/transport.py:232-318`) are generic
session-lifecycle code. They compose `GattBearer`, `BearerPump`, and
`MeshNode`, register the new DeviceKey, persist the sequence cursor
after every send, expose link failure, and provide idempotent cleanup.

This is not the missing PB-GATT driver, and upstream already has a
higher-level `MeshController`. For issue #12 it is best treated as a
reference for how the freshly provisioned DeviceKey can immediately be
used for Config Client traffic. Do not add a competing public session
API without first reconciling it with `MeshController`.

Tests at `tests/test_transport.py:123-193` prove sequence persistence,
idempotent close, and cleanup after construction failure.

### 6. Generic portion of post-provisioning configuration

`sphere_mesh.provision.configure()` (`src/sphere_mesh/provision.py:590-742`) contains
three broadly useful steps:

- DeviceKey-encrypted AppKey Add, lines 593-616;
- Composition Data Get and parse, lines 623-658; and
- one Model App Bind request per selected element/model, lines 682-720.

These operations cover the last bullet of issue #12, but the selection
policy must live above the generic Config Client primitives. Home
Assistant should bind the models for which it will create entities, as
the issue says, rather than copy the Sphere allowlist.

## Sphere-specific behavior that should not be upstreamed as generic

The following belongs in the Sphere harness or in an explicit HA policy
layer, not in a generic PB-GATT session driver.

### CLI, logging, and diagnostics

- `Phase0Failure`, `_provisioning_causes()`, and printed milestone text
  (`src/sphere_mesh/provision.py:106-113,322-340,479-482`).
- argument parsing and modes such as `--provision-only`, `--no-toggle`,
  and `--reset-node` (`src/sphere_mesh/provision.py:118-187`).
- the frame-log location, environment-based 60-second scan duration,
  and command-line transport selection.

Typed upstream exceptions are useful; this exact user-facing triage is
not.

### JSON state and address policy

- `load_or_create_state()` / `save_state()` and their schema
  (`src/sphere_mesh/provision.py:211-280`);
- default provisioner address `0x0001` and first device address
  `0x0100` (`src/sphere_mesh/provision.py:264-267`);
- the `devices` mapping and the decision to identify it by formatted
  address strings; and
- storing the last BLE address alongside UUID and DeviceKey
  (`src/sphere_mesh/provision.py:484-492`).

The private, atomic write is a good general security property, but
issue #12 explicitly needs HA's stored `.connect` document, not this
parallel JSON format.

### Target selection policy

The first unprovisioned beacon is chosen unless `--device-uuid` is
supplied (`src/sphere_mesh/provision.py:374-391`). Home Assistant instead needs a
discovery/config-flow choice with stable identity and user confirmation.

### Sphere post-provisioning recipe

The bind allowlists at `src/sphere_mesh/provision.py:663-680` are copied from the
Thingclips/Olight 3.2.5 behavior:

- SIG models `0x1000`, `0x1300`, `0x1303`, `0x1306`, `0x1307`; and
- vendor model `0x07d0:0x0004`.

The Config Network Transmit Set to count 5 / interval steps 2 at lines
722-742 is also Sphere/vendor setup policy. Neither is required for a
standards-compliant generic provisioning driver.

The choice between Generic OnOff and Light Lightness in
`_pick_control_model()` (`src/sphere_mesh/provision.py:751-767`) and all toggle
or readback behavior (`src/sphere_mesh/provision.py:770-840`) are validation
actions, not provisioning.

### Proxy-target fallback

`_find_proxy_target()` accepts a sole Node Identity advertiser without
cryptographically verifying it (`src/sphere_mesh/provision.py:520-587`). This is
a Phase 0 convenience and must not become generic identity behavior.
Home Assistant should verify Node Identity or use a known Network ID.

## Coverage of issue #12 and remaining gaps

### Unprovisioned-device scan

Status: implemented and improved locally.

Reusable code:
`sphere_mesh.transport._fresh_discoveries()` and
`scan_unprovisioned()` (`src/sphere_mesh/transport.py:74-119`).

Still needed in HA: discovery registration, a config-flow list, user
selection, and flow deduplication by Device UUID.

### Drive `Provisioner` over provisioning `GattBearer`

Status: works in the local CLI, but is not a reusable upstream entry
point.

Reusable code:
`sphere_mesh.transport.GattBearer`, `open_gatt_session()`, and the session
core in `sphere_mesh.provision.provision()`.

Still needed: extract and test a public asynchronous driver, then call
it from HA without blocking config-flow progress handling.

### Collision-free unicast allocation

Status: not solved for issue #12.

The local harness merely takes `state["next_unicast"]` before the
session (`src/sphere_mesh/provision.py:412`) and advances it by
`Capabilities.num_elements` after completion
(`src/sphere_mesh/provision.py:489-492`). That is safe only relative to nodes
previously added through this same monotonically maintained file. It
does not inspect arbitrary occupied ranges from an imported `.connect`
document.

There is also an API timing problem worth addressing upstream:
`Provisioner` receives the unicast address in its constructor, before
Capabilities reveals `num_elements`. The address is not transmitted
until the Data PDU, so a higher-level driver could select/reserve the
range after Capabilities if `Provisioner` exposed an address-selection
hook. Otherwise HA must precompute a free range conservatively from the
entire imported inventory.

At a minimum, allocation must treat each existing node as an occupied
range from primary address through `element_count - 1`, reject unknown
or incomplete layouts, and reserve the new range atomically against
concurrent flows.

### Persist into `.connect`

Status: not implemented locally.

The Sphere CLI writes a separate custom JSON record. It demonstrates
which results must survive—DeviceKey, Device UUID, primary address,
element count/address cursor—but it cannot be copied into HA storage.
The integration needs a schema-preserving `.connect` mutation path and
must keep entity IDs stable on reload.

Composition data should also be persisted after the proxy/config phase;
the Sphere CLI's representation at `src/sphere_mesh/provision.py:639-658` is a
useful inventory checklist, not necessarily the upstream schema.

### AppKey Add and Model App Bind

Status: demonstrated locally, partially generic.

The requests and response checks in `configure()` prove the procedure.
The Sphere allowlist and Network Transmit Set must be replaced with HA's
entity/model selection policy. The newly provisioned node must first be
found through its proxy advertisement or reached through another known
proxy; the local `mesh_session()` demonstrates that handoff at
`src/sphere_mesh/provision.py:843-904`.

## Risks to fix while extracting

1. The 60-second `asyncio.wait_for()` is a whole-session deadline, not
   “no progress” as the error text says
   (`src/sphere_mesh/provision.py:447-452`). A reusable driver should either call
   it a total timeout or reset a progress timeout on valid PDUs.

2. `open_gatt_session()` and the provisioning wait observe pump errors,
   but a notification subscription that fails after upstream
   `GattBearer.start()`'s grace period is recorded in `bearer.failure`,
   not sent to the provisioning `done` event. The current result is a
   misleading timeout. A session driver should monitor both pump and
   bearer failures.

3. The callback catches `BaseException`
   (`src/sphere_mesh/provision.py:438-440`). Library code should preserve task
   cancellation/system-exit semantics and catch protocol/application
   exceptions deliberately.

4. `prov.capabilities.num_elements` is dereferenced after success
   (`src/sphere_mesh/provision.py:491`) even though the public type is optional.
   Complete implies Capabilities in the current state machine, but a
   result object should enforce that invariant rather than rely on it.

5. The orchestration has transport-level unit tests but no transcript
   test of the live driver. This is the largest confidence gap before
   moving it into HA.

6. Only No OOB and Static OOB are supported by `Provisioner`. Issue #12
   may initially scope the config flow accordingly, but it must clearly
   reject or defer input/output OOB instead of presenting all devices as
   provisionable.

## Recommended upstream split

### Library change

1. Fold service-scoped characteristic resolution into upstream
   `GattBearer`.
2. update scan helpers to use fresh discoveries and explicit scan
   lifetime ownership;
3. add a tested async `ProvisioningSession` or `provision_gatt()` built
   from the core of `sphere_mesh.provision.provision()`; and
4. reuse the existing controller/config-client path for AppKey,
   composition, and binding rather than upstreaming the Sphere proxy
   session wholesale.

### Home Assistant integration change

1. add unprovisioned-device discovery and config-flow UX;
2. implement collision-free, atomic address reservation over the full
   imported document;
3. invoke the new library driver and handle supported OOB methods;
4. persist the node and DeviceKey into `.connect` storage;
5. connect through a known proxy, read composition, add AppKey, and bind
   exactly the models represented by entities; and
6. roll back the address reservation/storage mutation on failure while
   retaining enough diagnostic state to recover a node that received
   provisioning data but failed during HA persistence or configuration.

That split keeps Bluetooth Mesh protocol and session mechanics in
`btmesh`, while device choice, storage, entity policy, progress UI, and
rollback remain Home Assistant concerns.
