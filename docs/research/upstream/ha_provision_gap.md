# Home Assistant provisioning gap

Date: 2026-09-26

## Question and source set

This note compares issue 12 in `ha-bluetooth-mesh` with the
hardware-proven Sphere C flow in `src/sphere_mesh/provision.py`.

Issue 12 is still open and says that HA must:

- discover unprovisioned `0x1827` advertisements;
- drive `Provisioner` through a provisioning `GattBearer`;
- allocate a collision-free unicast range;
- add the node to the stored `.connect` document; and
- add the AppKey and bind every model HA will drive.

The issue text is at:

<https://github.com/dasimon135/ha-bluetooth-mesh/issues/12>

The old audit independently calls provisioning the only substantial
open roadmap item. It says that `provisioner.py` is complete but is
unreachable from HA
(`/local/src/old/ha-bluetooth-mesh/docs/
2026-07-26-audit-debug-and-improvements.md:344-346`).

The important conclusion is that the cryptographic protocol is not
the missing work. The missing work is an asynchronous transaction
around that protocol, plus durable network mutation and HA UX.

## Executive assessment

The Sphere harness proves almost the whole device-side transaction:

1. find a matching unprovisioned beacon;
2. connect to the PB-GATT characteristic pair;
3. run `Provisioner` and save the DeviceKey and element count;
4. reconnect through the new mesh network;
5. add the AppKey;
6. read Composition Data;
7. bind selected SIG and vendor models; and
8. perform the Sphere-specific Network Transmit Set.

The core sequence is visible at
`src/sphere_mesh/provision.py:343-493`, `src/sphere_mesh/provision.py:590-742`, and
`src/sphere_mesh/provision.py:843-904`.

That sequence should be reused as the behavioral reference. It should
not be pasted into an HA config flow. Three reusable pieces should be
promoted into `btmesh`: an asynchronous provisioning-session facade,
a Config Client facade, and contiguous address allocation informed by
the Capabilities PDU. HA should retain discovery, connection policy,
storage, config-flow progress, recovery, and `.connect` compatibility.

## What the old HA integration already provides

### Config-entry lifecycle and stable entities

The import flow already parses and validates the raw export, uses the
network identifier as the config-entry unique ID, and stores the raw
JSON (`config_flow.py:94-125`). The reconfigure flow can replace that
JSON without deleting the entry and explicitly schedules reload
(`config_flow.py:127-159`). Those are the right persistence and reload
mechanics after a node has been committed.

Entity identity is already stable across such a reload. A light unique
ID is the network identifier plus its element unicast address, and its
device ID uses the node's primary unicast address
(`light.py:245-255`). Appending a node and reloading therefore creates
only the new entities; existing IDs and history remain intact.

### HA-native Bluetooth path selection

`mesh_transport.py` already has the correct HA boundary:

- inspect HA's advertisement snapshot;
- ask HA for a connectable `BLEDevice`;
- let HA choose the local adapter or ESPHome proxy path; and
- use `bleak-retry-connector` to establish the link.

The proxy-network implementation is at
`mesh_transport.py:132-188` and `mesh_transport.py:297-331`.
Provisioning can reuse this architecture, including path scoring and
connection cleanup, but not the functions unchanged:

- discovery currently matches only `0x1828` Network ID adverts;
- `async_connect_bearer()` constructs
  `GattBearer(..., provisioning=False)` unconditionally; and
- callback registration currently matches only `PROXY_SERVICE`
  (`mesh_transport.py:334-359`).

HA needs parallel helpers for `PROV_SERVICE`: snapshot enumeration and
push callbacks using `parse_unprovisioned_service_data()`, followed by
a connection helper that constructs the bearer with
`provisioning=True`. This stays HA code because advertisement caches,
scanner selection, retry budgets, and `BluetoothServiceInfoBleak` are
HA concerns.

### Connection ownership and serialization

`MeshCoordinator` already owns one network's connection, controller,
lock, retry policy, backoff, keep-alive, and shutdown
(`coordinator.py:189-285`). It also prevents a queued command from
reconnecting after unload and cleans up partial connections
(`coordinator.py:599-745`). Provisioning must be serialized with this
owner. Otherwise a kept-alive proxy connection, ordinary entity
commands, and the provisioning/configuration job can compete for BLE
slots and sequence state.

The coordinator's persistence rules are also reusable after the node
joins the mesh. It writes SEQ immediately after controller startup and
after commands, including failed commands
(`coordinator.py:729-738`, `coordinator.py:763-801`). Its stored state
is keyed by network identity, not config-entry ID
(`coordinator.py:202-218`). Provisioning should use this same network
cursor once post-provision configuration starts; it must not create an
independent HA sequence cursor.

### Existing runtime controller

`MeshController` already combines a proxy bearer, `BearerPump`, and
`MeshNode` and configures the proxy filter on startup
(`src/btmesh/controller.py:94-183`). It can read Composition Data with
the DeviceKey (`controller.py:362-393`). It cannot yet finish issue 12:

- device keys are copied only from the immutable `Network` in the
  constructor (`controller.py:133-149`);
- the internal `MeshNode` is private;
- there is no AppKey Add method; and
- there is no Model App Bind method.

The HA coordinator therefore cannot insert the newly derived DeviceKey
and configure the node without either rebuilding its network or
dropping below the controller abstraction.

## What the Sphere harness proves and can be reused

### PB-GATT transaction

The harness's `provision()` function contains the missing end-to-end
driver (`src/sphere_mesh/provision.py:343-493`):

- it selects an advertised Device UUID;
- opens a service-specific GATT bearer;
- creates a FIFO `BearerPump` for message type `0x03`;
- routes notifications to `Provisioner.handle_pdu()`;
- converts pump/protocol failure into completion of one waiter;
- enforces an overall timeout;
- stops pump, bearer, and client on every path; and
- saves DeviceKey, UUID, BLE address, and the next free address only
  after Provisioning Complete.

The protocol pieces are already in `btmesh`. `Provisioner` exposes a
synchronous send callback, terminal state, DeviceKey, capabilities,
and `on_done` (`src/btmesh/provisioner.py:141-199` and
`src/btmesh/provisioner.py:390-393`). `BearerPump` is the adapter from
that callback to ordered asynchronous GATT writes
(`src/btmesh/pump.py:24-86`). The harness is consequently a reference
implementation of a missing high-level library object, not evidence
that HA should manually assemble these pieces again.

### Post-provision Config Client sequence

The harness also proves the correct ordering:

- AppKey Add before any application-model use
  (`src/sphere_mesh/provision.py:593-616`);
- Composition Data Get under the DeviceKey
  (`src/sphere_mesh/provision.py:623-658`);
- model selection and binding from actual composition, not guesses
  (`src/sphere_mesh/provision.py:660-720`); and
- only then application-key control
  (`src/sphere_mesh/provision.py:770-840`).

The low-level access codecs for AppKey Add, SIG Model App Bind, vendor
Model App Bind, status parsing, and Composition Data already exist in
`btmesh.access` (`access.py:405-454`, `access.py:659-691`, and
`access.py:812-861`). The missing part is the high-level async Config
Client which sends them with the DeviceKey, checks status codes, and
returns a typed result.

### Sphere-specific successful tail

The Sphere C needs more than the generic issue-12 minimum. The tested
flow binds the app key to an allowlist across all elements, including
the vendor model, and then sends Config Network Transmit Set with count
5 and interval steps 2 (`src/sphere_mesh/provision.py:663-742`). The encoder is
currently project-local because `btmesh` 0.10.3 lacks it
(`src/sphere_mesh/access.py:1-17`).

Network Transmit is a standard Foundation message, so its opcode,
encoder, typed status, parser, and Config Client method belong in
`btmesh`. The choice of values `(5, 2)` and the Sphere model allowlist
are product policy and should stay in HA as a device profile. The HA
integration should not apply those values to all Bluetooth Mesh nodes.

### Transport fixes discovered by Sphere

`sphere_mesh.transport.GattBearer` selects Data In/Out characteristics from
the requested service and chooses deterministically when duplicate
UUIDs are exposed (`src/sphere_mesh/transport.py:34-71`). That fix matters even
more when one physical device can expose provisioning and proxy
services. It should be fixed in upstream `btmesh.GattBearer`, not
copied into the integration.

The Sphere scanner also maintains a fresh discovery set and can return
as soon as a target UUID appears (`src/sphere_mesh/transport.py:74-119`). The
early-exit behavior is useful as semantics for HA discovery, but the
implementation itself is for standalone Bleak scanners. HA should use
its snapshot and callback APIs instead.

## APIs that should be extracted into `btmesh`

### 1. Async provisioning-session facade

Add a class or coroutine with a shape similar to:

```python
result = await provision_over_gatt(
    bearer,
    net_key=network.net_key,
    net_key_index=network.net_key_index,
    iv_index=iv_index,
    allocate_unicast=allocator,
    static_oob=static_oob,
    timeout=60,
)
```

It should own `Provisioner`, `BearerPump`, notification dispatch,
timeout, terminal error propagation, and reverse-order shutdown. Its
result should include DeviceKey, assigned primary unicast, Device UUID
when supplied by the caller, and Capabilities/element count.

The connected Bleak client and its acquisition remain outside this
API. That keeps the library independent of HA while letting HA choose
the adapter/proxy and guaranteeing client disconnect on its side.

### 2. Capability-aware contiguous address allocation

This is a correctness blocker. The current `Provisioner` constructor
requires a primary unicast address before it has received Capabilities
(`provisioner.py:145-178`). The number of elements is learned later
(`provisioner.py:269-273`). The Sphere harness works because it owns a
monotonically increasing `next_unicast` and advances it after success
(`src/sphere_mesh/provision.py:412-421` and `src/sphere_mesh/provision.py:489-492`).

That is not safe for an arbitrary imported network. The selected range
`base .. base + num_elements - 1` may overlap an existing secondary
element even when `base` itself is free.

The library API should defer final address choice until Capabilities
arrives. For example, `Provisioner` can accept an allocator callback
and invoke it with `caps.num_elements` before building Provisioning
Data. The network model also needs a method such as
`free_unicast_range(count, preferred=...)` which checks all element
addresses and address-space bounds. Existing `Network.free_unicast()`
checks only one address and is explicitly designed for HA's source
address (`network_model.py:264-291`); it is not a node allocator.

### 3. High-level Config Client

Add a `ConfigClient` facade over `MeshNode`, or extend
`MeshController` with explicit configuration methods:

- `add_app_key(node, net_idx, app_idx, key)`;
- `get_composition(node, page=0)`;
- `bind_sig_model(element, app_idx, model)`;
- `bind_vendor_model(element, app_idx, cid, model)`; and
- `set_network_transmit(node, count, interval_steps)`.

Each method should use the target DeviceKey, parse the expected status,
and raise a typed status error rather than returning a raw
`ReceivedMessage`. A generic `configure_node()` can orchestrate the
common Add → Composition → Bind sequence if its model-selection policy
is provided by the caller.

HA or a product profile must decide which models to bind. The Sphere
allowlist is valid hardware evidence, not a cross-vendor standard.

### 4. Runtime insertion of a freshly provisioned node

The controller needs either a public `add_device(unicast, device_key)`
method or a configuration-session constructor which accepts one
provisional node independently of `Network.nodes`. Without it, HA must
serialize a half-described node merely to get its DeviceKey into the
runtime.

The latter is cleaner for crash recovery: a pending provisioning
record can create a Config Client, obtain composition, then become a
full static `Node` only when the configuration transaction succeeds.

### 5. Composition-to-model conversion

`CompositionData` and static `Node`/`Element`/`Model` types already
exist, but no public conversion joins them. Add a helper that combines:

- Device UUID;
- assigned primary unicast;
- DeviceKey;
- Composition Data; and
- the AppKey indexes actually bound by the configuration transaction.

It should produce the vendor-neutral `btmesh.network_model.Node`.
Serialization into ThingOS `.connect` JSON should remain HA-side: that
format contains vendor fields and unknown fields which HA must
preserve. The library should not pretend that a ThingOS export is the
Bluetooth Mesh configuration database format.

## Work that still belongs in Home Assistant

### Discovery and UI

Add a provisioning route that reads both cached and live `0x1827`
service data, deduplicates by Device UUID, shows UUID/OOB information,
and lets the user choose the target and destination mesh. It should
register a `PROV_SERVICE` active-scan callback like the existing proxy
callback, but parse unprovisioned data.

This cannot simply create another normal config entry: the integration
has one entry per network, and `async_step_user()` aborts when that
network identity already exists (`config_flow.py:107-118`). The node
discovery flow should select and mutate an existing network entry, or
be an explicit “add node” flow associated with that entry. Completion
should update/reload that same entry.

The current `Provisioner` supports No OOB and Static OOB only. It
warns and attempts No OOB when only output/input OOB is advertised
(`provisioner.py:279-301`). HA can initially expose an optional Static
OOB step, but numeric/output/input OOB requires a protocol API extension
and interactive flow resumption; the original design's general OOB UX
does not exist yet.

### One owner and a visible job

Provisioning and configuration may take well beyond one form submit.
HA needs a cancellable background job with progress states such as
discovering, connecting, provisioning, waiting for proxy advert,
reading composition, binding, and committing. Config-flow pages can
poll/show that job; they should not hold BLE resources in ephemeral
flow objects.

The job must acquire the network coordinator's exclusive operation
lock, temporarily release its held proxy link, and keep ordinary entity
commands out until the transaction ends. It should use HA task APIs so
unload and shutdown cancel it and run cleanup.

### Crash-safe pending-node journal

Provisioning Complete is an irreversible boundary from HA's point of
view: the node has left `0x1827` and now knows the NetKey, while HA is
the only holder of its derived DeviceKey. Losing that key before the
`.connect` update strands the node until factory reset.

Immediately after Complete, HA must atomically persist a pending record
containing at least Device UUID, DeviceKey, primary unicast, element
count, destination network ID, and current stage. A config-flow object
or in-memory coordinator field is insufficient.

On restart, setup should detect this journal and resume post-provision
configuration. It should not allocate the reserved address range to a
second device. Remove the journal only after the updated network
document is durable. If configuration fails, expose Retry and Config
Node Reset recovery rather than silently deleting the secret.

### Reconnect and configure

After PB-GATT disconnects, HA must wait for the node to reboot and for
a matching `0x1828` network advert. It may configure through the new
node or any existing proxy node on the subnet; mesh routing makes the
latter valid. It must insert the pending DeviceKey into the Config
Client before sending device-keyed traffic.

The Sphere harness already demonstrates this phase boundary by closing
the provisioning session, scanning for the Network ID, and opening a
new proxy session (`src/sphere_mesh/provision.py:468-493`,
`src/sphere_mesh/provision.py:520-587`, and `src/sphere_mesh/provision.py:843-875`). HA
should add retry/backoff around the reboot window instead of assuming
the first advert is immediate.

### Generic binding versus device profiles

For ordinary issue-12 support, bind the AppKey only to server models
that the integration can expose: Generic OnOff, Light Lightness,
Light CTL, and the relevant temperature/HSL models. Always derive
elements from Composition Data.

Sphere C needs a profile layered over that generic rule. Its proven
allowlist includes `0x1000`, `0x1300`, `0x1303`, `0x1306`, `0x1307`,
and vendor model `0x07D0:0x0004`, followed by Network Transmit Set
`(5, 2)` (`src/sphere_mesh/provision.py:663-742`). Identification of this
profile needs a defensible composition fingerprint such as CID/PID,
not a name or BLE address.

### Updating the stored export

The integration stores the source JSON verbatim
(`const.py:7-9`). To preserve vendor extensions, HA should load that
object, append one synthesized node object to `nodes`, and leave every
unknown top-level and existing-node field untouched. The new node must
include:

- UUID, primary address, DeviceKey, and composition identifiers;
- one element object per reported element;
- all SIG and vendor model IDs; and
- `bind: [app_key_index]` exactly for models whose bind succeeded.

Before updating the config entry, parse the candidate JSON back through
`Network.from_connect()` and assert that the new node, every element,
and every binding round-trip. Then use the existing update-and-reload
pattern. This is also where a user-supplied friendly name can be placed
in the ThingOS-compatible fields.

### Failure and recovery UX

At minimum HA needs translated errors for:

- target disappeared before connect;
- no connectable Bluetooth path or free proxy slot;
- unsupported/OOB-required authentication;
- provisioning protocol failure or timeout;
- no collision-free contiguous address range;
- node never appeared on the provisioned network;
- Composition Data timeout;
- AppKey or model-bind status failure; and
- durable-commit failure after Provisioning Complete.

The last four must retain the pending journal and offer retry. A Reset
action is useful but must be explicit because it changes the physical
node back to unprovisioned state.

## Suggested implementation order

1. Upstream the service-specific `GattBearer` characteristic fix and
   the Network Transmit codec.
2. Add capability-aware address allocation and an async provisioning
   session to `btmesh`, with transcript-driven tests.
3. Add a typed Config Client and composition-to-node conversion, with
   mock-bearer tests for partial status failures.
4. Add HA `0x1827` snapshot/callback discovery and a provisioning
   connection helper.
5. Add the pending-node store and recovery behavior before exposing a
   UI button.
6. Add the serialized HA job and existing-entry config-flow UX.
7. Implement generic binding and `.connect` round-trip commit.
8. Add the Sphere profile and hardware-test its complete sequence.

## Bottom line

Roughly half of issue 12 already exists, but at the wrong abstraction
levels. HA has mature connection ownership and persistence for an
already described network. The Sphere harness has a working one-shot
transaction. `btmesh` has the cryptography, PDU codecs, pump, and raw
Config Client messages.

The shortest safe path is to make the harness's two protocol
transactions first-class library APIs, then let HA provide discovery,
durability, serialization, and UX. Copying the harness directly into a
config flow would work on the happy path but would leave the two most
dangerous gaps unresolved: contiguous address allocation after the
element count is known, and recovery after Provisioning Complete but
before the new node is committed to storage.
