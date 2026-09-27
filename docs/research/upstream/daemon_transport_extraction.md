# Daemon transport extraction

Research date: 2026-09-26.

## Decision

Extract the proven GATT session and Mesh Proxy wiring into a
project-level `mesh_transport.py`. Keep provisioning policy, scanning
policy, commands, diagnostics, and JSON ownership in
`src/sphere_mesh/provision.py`.

The reusable boundary should be a live `MeshProxySession` containing
the connected client, `GattBearer`, `BearerPump`, and `MeshNode`. Its
opening function should accept decoded keys and addresses plus a
synchronous `persist_seq(next_seq)` callback. The provisioning harness
and daemon can then share identical radio behavior while retaining
different state stores and command lifecycles.

Do not replace this path with `btmesh.controller.MeshController` in
the extraction commit. That controller configures a Proxy accept list,
handles extra Proxy PDU types and beacons, changes timeout semantics to
best-effort returns, uses `Network`, and tracks a TID. Those are useful
later, but they change the frames and observable behavior.

## Current path and ownership

The successful post-provision path is assembled in
`sphere_mesh.provision.mesh_session` at lines 906-990:

1. `_find_proxy_target` scans for service `0x1828` and selects a
   candidate.
2. `open_gatt_session` connects, constructs `GattBearer` in Proxy
   mode, and subscribes with whole-session retry.
3. A `BearerPump` serializes Network PDUs into Proxy SAR frames.
4. A `MeshNode` owns network/transport encryption, request matching,
   lower-transport segmentation, and the next sequence cursor.
5. Incoming `MSG_TYPE_NETWORK_PDU` messages are fed to the node.
6. Outgoing Mesh PDUs are queued, then the new next-to-use sequence
   cursor is immediately written to the JSON state file.
7. The caller runs operations through the exposed `MeshNode`, then
   stops pump, bearer, and client in order.

The lower layers are already reusable and should remain vendored:

- `btmesh.bearer.GattBearer` provides characteristic selection,
  Proxy SAR, delayed-subscribe handling, and raw send/receive.
- `btmesh.pump.BearerPump` provides the required single-consumer send
  queue. Concurrent bearer sends can interleave SAR frames.
- `btmesh.node.MeshNode` provides Mesh Network/Transport/Access
  processing and request correlation.
- `btmesh.bearer.connect_local` and `EsphomeTransport` provide the two
  connection backends.
- `btmesh.bearer.find_proxy_node` provides generic Network ID matching.

This is an orchestration extraction, not a reason to fork those
implementations.

## Exact symbols to move

Move these symbols unchanged from `src/sphere_mesh/provision.py` into
`mesh_transport.py` first:

- `LocalTransport` (currently lines 297-312).
- `_disconnect_quietly` (lines 331-335), exported under the non-private
  name `disconnect_quietly` if both callers need it.
- `GATT_SESSION_ATTEMPTS` (line 338).
- `open_gatt_session` (lines 341-379).

Move the transport-only statements currently embedded in
`mesh_session` (lines 917-961 and 987-990) into the new session/opener:

- the temporary `session_handler` and `dispatch` callback;
- `open_gatt_session(..., provisioning=False, ...)`;
- construction and error hook of `BearerPump`;
- the synchronous `send_network_pdu` bridge;
- construction of `MeshNode` and `add_device`;
- filtering inbound Proxy PDUs to `MSG_TYPE_NETWORK_PDU`;
- starting the pump; and
- orderly pump, bearer, and client shutdown.

Those statements form one lifecycle and have circular late binding:
`dispatch` exists before `node`, while the node's send callback closes
over `node`. Keep the callback indirection so notifications received
during startup are handled exactly as today.

The new module needs these imports:

```python
import asyncio
import logging
from collections.abc import Callable
from typing import Any, Protocol

from btmesh.bearer import BearerError, GattBearer, connect_local
from btmesh.node import MeshNode
from btmesh.proxy_pdu import MSG_TYPE_NETWORK_PDU
from btmesh.pump import BearerPump
```

Keep the `bleak.BleakScanner` import lazy inside
`LocalTransport.scanner`, as it is now. This preserves importability
of tests and non-BLE commands without eagerly loading the platform BLE
stack.

`src/sphere_mesh/provision.py` should import `LocalTransport`,
`open_gatt_session`, and `open_mesh_proxy_session` (or the session
class) from the new module. It still needs `BearerPump` for PB-GATT
provisioning and diagnostic type annotations.

## Recommended public boundary

A minimal useful surface is:

```python
class BleTransport(Protocol):
    async def start(self) -> None: ...
    def scanner(self) -> Any: ...
    async def connect(self, device: Any, timeout: float = 30.0) -> Any: ...
    async def stop(self) -> None: ...

class MeshProxySession:
    node: MeshNode
    pump: BearerPump

    @property
    def failure(self) -> BaseException | None: ...

    async def close(self) -> None: ...

async def open_mesh_proxy_session(
    transport: BleTransport,
    device: Any,
    *,
    netkey: bytes,
    appkey: bytes,
    device_key: bytes,
    iv_index: int,
    provisioner_addr: int,
    unicast: int,
    seq: int,
    persist_seq: Callable[[int], None],
) -> MeshProxySession: ...
```

The callback must receive the next sequence number to use, not the
number just transmitted. That is the meaning of `MeshNode.ctx.seq`
and `NetworkContext.seq`.

The harness adapter should remain synchronous and preserve the exact
ordering used today:

```python
def persist_seq(next_seq: int) -> None:
    state["seq"] = next_seq
    save_state(state_path, state)
```

The session must call it immediately after `pump.put(raw)`, just as the
current closure does. Persisting only on `close`, only after a reply,
or from the pump task is not behavior-preserving and can reuse a nonce
after a crash.

Expose `node` and `pump` during this step. The existing `configure`,
`read_control_state`, and `toggle_loop` functions already accept those
objects, so the harness diff stays mechanical. A higher-level daemon
API can wrap them later.

## Symbols that should stay in the harness

Do not move the following into the transport module:

- `_find_proxy_target`: it embeds CLI printing, `Phase0Failure`, the
  risky single-Node-Identity fallback, inspection of the JSON device
  address, and the five-second harness timeout. The daemon needs its
  own reconnect and identity-selection policy.
- `pick_device`: it selects from the harness JSON schema and CLI args.
- `configure`, `toggle_loop`, `read_control_state`, and
  `_full_payload`: these are application/configuration behavior.
- `Phase0Failure`, `_timeout_causes`, and diagnostic strings: these
  are human-facing harness policy. The reusable layer should report
  `BearerError`, `failure`, and ordinary exceptions.
- `load_or_create_state`, `_validate_state`, and `save_state`: the
  daemon must own locking, recovery, and durable storage. Passing a
  callback prevents a dependency from transport back into the CLI.
- `provision`: PB-GATT provisioning uses the common connection helper
  and bearer, but its state machine is a separate workflow.
- Logging setup, scan/provision timeout constants, and CLI parsing.

The scan itself stays outside the session. The harness relies on
`find_proxy_node(..., stop_after=False)`, keeps the scanner active
until GATT is established, then stops it in a `finally`. The opener
should accept an already selected device; it must not hide scanner
ownership or target policy.

## Coupling and behavior risks

### Sequence durability is transport correctness

`MeshNode` advances its cursor before invoking its synchronous send
callback. For segmented messages it allocates the entire SEQ block
first. The callback queues a PDU and saves `node.ctx.seq`; multiple
segments therefore repeatedly save the same already-reserved end
cursor. This is conservative and correct.

The JSON save happens before the asynchronous GATT write completes. A
failed write burns sequence numbers, which is safe. Reversing that
ordering can reuse an encrypted nonce after a process or power failure.

The JSON store is atomic only for one writer. It uses a fixed `.tmp`
filename and no process lock. A daemon and CLI pointed at the same file
can overwrite each other or collide on the temp file. Before enabling
the daemon, give it exclusive ownership/a lock and make the CLI refuse
concurrent use. That is state-store work, not this extraction.

### Pump failure is asynchronous

`BearerPump.put` only enqueues. A later GATT error terminates the pump,
sets `pump.failure`, and leaves queued work undrained. The session must
surface `pump.failure` and the bearer's delayed-subscription `failure`;
a daemon must reconnect rather than continuing to enqueue.

`BearerPump.stop` cancels rather than drains. This matches the current
request/reply workflow. Calling `flush` during shutdown would change
failure and timing semantics. Fire-and-forget daemon commands must
explicitly flush, as `MeshController` already does.

### Lifecycle ordering is observable

Preserve startup order: bearer subscription, node/callback wiring,
then pump start. Preserve shutdown order: pump stop, bearer stop,
client disconnect. The scanner remains caller-owned and must stop
whether opening succeeds or fails.

If `MeshNode.close()` is added for timer cleanup, do it in a separate
follow-up with a test. The harness does not call it today. It is likely
right for a daemon, but is not a literal extraction.

Make session close idempotent for daemon use, but test that this does
not produce extra `stop_notify` or disconnect calls in the harness.

### Early notifications require deferred dispatch

`GattBearer.start` installs notifications before `MeshNode` exists.
The list-backed handler tolerates a PDU in that window. Directly
binding `node.handle_network_pdu` either reorders creation or refers to
an uninitialized node. Retain a dispatcher or safely drop/log until
the session's node is assigned.

### Do not accidentally enable Proxy configuration

The harness accepts only Network Proxy PDUs and sends no Proxy Filter
Configuration PDUs. `MeshController.start` sends two such messages and
consumes sequence numbers. Substitution would alter the transcript,
initial SEQ, and potentially whether statuses arrive. Keep that
migration separate and hardware-validate it.

### Backend and error boundaries

`EsphomeTransport.start` may install a process-global habluetooth
manager. A daemon should start one backend and reuse it, not start/stop
it per command. The session owns one GATT connection only.

`open_gatt_session` retries only `BearerError`. Catching all exceptions
would retry programming/configuration errors too. Preserve that
boundary, the attempt count, and the two-second retry delay.

Give the new module its own logger. Tests should assert outcomes and
call order rather than logger names; the logger name is the only
harmless visible difference expected from a direct move.

## Tests required before changing the harness import

Add project-level `test_mesh_transport.py` tests using synthetic
clients and transports; no adapter is needed.

### Connection and retry

- Connect fails once then succeeds: verify two connects and one live
  session.
- Subscribe fails once: verify disconnect before a fresh connection.
- All attempts fail: verify `BearerError`, exact count, and no leak.
- Use `attempts=1` or patch sleep to avoid production backoff in tests.
- Disconnect failure during retry/close stays ignored.
- `LocalTransport` start/stop are no-ops and connect delegates its
  device and timeout unchanged to `connect_local`.

### Proxy session wiring

- Opening uses `GattBearer(..., provisioning=False)`.
- A Network PDU notification reaches `MeshNode`; another type is
  ignored.
- Outgoing traffic reaches one shared `BearerPump` in order.
- The supplied DeviceKey is registered for `unicast`, proved by a
  device-keyed request or tightly scoped fake node.
- Pump starts only after receive handler and node installation.
- Close order is pump, bearer, disconnect; repeated close is safe if
  idempotence is promised.
- Pump and bearer failures are observable from the session.

### Sequence persistence

- Seed a known `seq`, send one unsegmented message, and assert the
  callback synchronously receives `seq + 1`.
- Send a segmented message and assert the final value skips the full
  reserved segment count.
- Fail the fake bearer write after enqueue and assert the advanced
  cursor was still persisted: uncertain sends burn sequence numbers.
- Assert persistence occurs after queueing but before the async bearer
  write runs.
- Assert no transport code imports or calls `save_state`.

### Harness regression

- Keep existing state, mode, default transport, and characteristic
  tests green.
- Test that `mesh_session` passes decoded NetKey, AppKey, DeviceKey,
  IV Index, source address, unicast, and current SEQ to the opener,
  then takes the same configuration/control branch.
- Test the harness persistence adapter updates the same dictionary and
  calls `save_state` for every outgoing allocation.
- Test that scanner stop runs when session opening fails.
- Run vendored `test_bearer.py`, `test_pump.py`, `test_node.py`, and
  `test_network.py` suites for SAR, ordering, failures, and SEQ rules.

Finally, run an import-only smoke test without `bleak` loaded. Moving
`LocalTransport` must not make `--init-only` or test collection load
the platform BLE stack.

## Safe implementation sequence

1. Add `mesh_transport.py` and isolated fake-based tests.
2. Move the four standalone plumbing symbols without changing bodies,
   retry constants, or exception handling.
3. Add the session/opener by transplanting only the wiring from
   `mesh_session`.
4. Replace harness wiring with an opener call, a synchronous JSON
   persistence adapter, and `session.close()` in the existing
   `finally`.
5. Run project and vendored transport/node tests.
6. Compare a verbose hardware run with the prior log: same scan
   choice, retries, PDU types, SEQ progression, command order, and
   shutdown behavior.

Defer daemon reconnection loops, state locking, signal handling, Proxy
filter setup, beacon/IV adoption, TID persistence, and controller API
design until this behavior-preserving seam is established.
