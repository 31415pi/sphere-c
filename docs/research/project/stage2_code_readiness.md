# Stage 2 code-readiness audit

Audit date: 2026-09-27.

## Scope and conclusion

Stage 2 requires a long-running Mesh Proxy connection, automatic
reconnection after radio or node loss, and crash-safe sequence and
transaction-ID cursors (`docs/plans/daemon.md:13-17`).

The Stage 1 transport seam is a useful base, but the repository is not
yet ready to add a reconnect loop around it unchanged. The main
blocking correctness issues are:

- outgoing sequence state is queued before durable reservation, so a
  persistence failure is not fail-closed;
- a live connection has no positive disconnect notification or waitable
  failure signal;
- closing a session does not close its `MeshNode`, so lower-transport
  timers can outlive a connection;
- TID state is reset to zero on every harness run and is absent from
  the state schema; and
- the JSON writer is atomic for one cooperative writer, but is neither
  durable across a host crash nor safe against concurrent processes.

Stage 2 should introduce a state store and a connection supervisor as
separate units. The existing harness should consume those units only
after their failure behavior is covered by tests.

## Reusable pieces already present

### Transport and discovery

`LocalTransport` provides a small backend boundary with lazy Bleak
imports (`src/sphere_mesh/transport.py:167-182`). Keeping the adapter
alive for the daemon lifetime is compatible with the current API.

`find_proxy_node()` already:

- scans only fresh advertisements;
- selects a cryptographic Network ID match;
- removes its detection callback in `finally`; and
- optionally leaves scanner stop ownership to the caller
  (`src/sphere_mesh/transport.py:122-164`).

Those properties are appropriate for every reconnect attempt. The
daemon should require an exact Network ID match. The harness fallback
that assumes a lone Node Identity advertiser is the target is explicitly
unverified and must not become daemon policy
(`src/sphere_mesh/provision.py:534-547`).

`open_gatt_session()` provides bounded retries over the whole
connect-and-subscribe operation. It disconnects a client whose
subscription fails and preserves cancellation by catching only
`BearerError` (`src/sphere_mesh/transport.py:193-229`). This is useful
inside one supervisor attempt. Its fixed three attempts and two-second
delay are not a long-running reconnect policy.

### One-session protocol wiring

`open_mesh_proxy_session()` constructs the useful Stage 1 boundary:

- one subscribed `GattBearer`;
- one ordered, single-consumer `BearerPump`;
- one `MeshNode` with key and address state; and
- an inbound dispatcher that tolerates notifications arriving during
  construction (`src/sphere_mesh/transport.py:255-318`).

The session exposes both asynchronous failure sources through
`failure`: pump write failure and late subscription failure
(`src/sphere_mesh/transport.py:242-244`). Its idempotent close order is
also correct: stop the pump, stop notifications, then disconnect
(`src/sphere_mesh/transport.py:246-252`).

The upstream `MeshNode` is worth retaining. It owns Network and
Transport encryption, allocates a complete sequence block before a
segmented send, correlates acknowledged responses, and handles Segment
Acks. Its `ctx.seq` is already the next-to-use cursor. The pinned
upstream implementation also has `MeshNode.close()` for cancelling SAR
timers; the project session simply does not call it yet.

### Existing state behavior

`save_state()` writes a temporary file, applies mode `0600`, and uses
an atomic same-directory replacement
(`src/sphere_mesh/provision.py:274-280`). This is a useful primitive for
the future store, and it prevents readers from observing partially
written JSON during an ordinary process run.

`load_or_create_state()` gives malformed JSON a user-facing
diagnostic instead of exposing a traceback
(`src/sphere_mesh/provision.py:235-271`).
The existing state holds the keys, IV Index, provisioner source, node
DeviceKey, and next sequence cursor needed to rebuild a session.

### Test fixtures

The project already has useful fake boundaries:

- `FakeClient`, `FakeTransport`, and `bearer_factory` script initial
  connection outcomes (`tests/test_transport.py:11-40`);
- `FakeProxyBearer` records writes and supports injected failure
  (`tests/test_transport.py:123-130`); and
- the `tmp_path` state tests establish mode and replacement checks
  (`tests/test_provision.py:11-28`).

These should be promoted into shared fixtures once the Stage 2 test
surface spans state, session, and supervisor modules.

## Coupling to extract before the daemon loop

### State ownership is still in the provisioning CLI

The schema, validation, creation, and persistence functions all live in
`provision.py` (`src/sphere_mesh/provision.py:211-280`). The session
correctly accepts a persistence callback instead of importing the
writer, but a daemon would still have to import CLI policy to load the
same file.

Extract a typed state-store boundary that owns:

- schema versioning and migration;
- exclusive process ownership;
- fail-closed load behavior for an initialized installation;
- durable cursor reservation;
- secret-file permissions; and
- replacement and recovery policy.

Keep key generation and the harness's create-if-missing behavior in a
provisioning-facing adapter. A daemon must not interpret a missing
authoritative file as permission to generate new keys and reset SEQ to
zero.

### Target selection is mixed with CLI output

`_find_proxy_target()` combines scanning, prints, `Phase0Failure`, a
five-second timeout, unsafe Node Identity fallback, and inspection of
the harness state dictionary
(`src/sphere_mesh/provision.py:520-587`).

The supervisor needs a small resolver that returns only a verified
`BLEDevice` or a typed not-found result. Logging, attempt backoff, and
availability state belong above it. Scanner ownership must remain
explicit because the current working path leaves scanning active until
the GATT connection has been established
(`src/sphere_mesh/provision.py:852-873`).

### Failure is pollable, not waitable

`MeshProxySession.failure` is a snapshot property. Nothing wakes a
supervisor when it changes, and the GATT client has no registered
disconnect callback. A remote power loss may therefore remain invisible
until a write fails. A receive-only idle link may appear healthy
indefinitely.

Add a waitable session-lost event whose first cause wins. Feed it from:

- the Bleak disconnection callback;
- `BearerPump.on_error`;
- late `GattBearer.failure`; and
- deliberate shutdown, distinguished from failure.

Backends that cannot install a callback need an `is_connected` fallback
or bounded health check. The supervisor should not busy-poll
`session.failure`.

### Command lifetime and connection lifetime are not coordinated

`MeshNode.request()` can remain asleep until its timeout after the
connection dies. Its retry then queues into a dead pump, consuming more
SEQ values. `BearerPump.put()` accepts work even after its consumer has
failed, so the queue can grow without bound.

The supervisor needs a generation-scoped command gate. On link loss it
must:

1. stop admitting commands to the failed generation;
2. cancel or fail its outstanding commands;
3. close that session completely; and
4. only then publish the replacement as available.

A single lock around session replacement and command admission is the
simplest initial policy. It also avoids ambiguous concurrent requests
for the same source and response opcode; upstream response matching is
FIFO rather than correlation-ID based.

### TID allocation is embedded in commands

The harness-local toggle counter starts at zero for every run
(`src/sphere_mesh/provision.py:770-806`). A restart within the model's
transaction window can therefore make a genuine Set look like a
retransmission.

The pinned upstream `MeshController` has useful high-level model
methods and a cursor seeded by its `tid` constructor argument. However,
its `_next_tid()` increments only in memory before constructing a Set;
it has no synchronous persistence hook. Adopting it unchanged would not
meet Stage 2's crash-safety requirement.

Extract a persistent transaction allocator below all Set operations,
or extend the controller boundary so allocation durably advances the
cursor before the payload can be queued. Use one cursor for the current
single provisioner source; do not give each reconnect a fresh cursor.

## Correctness findings

### 1. Sequence reservation is not fail-closed

The outgoing callback currently does this
(`src/sphere_mesh/transport.py:286-289`):

```python
pump.put(raw)
persist_seq(node.ctx.seq)
```

In the normal event loop, the synchronous save finishes before the pump
task can perform the GATT write. That explains why the existing happy
path is normally safe. It is not a sufficient storage contract:

- if persistence raises, a PDU is already present in a live queue;
- a caller can catch the exception and yield, allowing that PDU to be
  transmitted with an uncommitted cursor; and
- a process restart can then reuse the same sequence number and nonce.

Durably reserve the advanced cursor before exposing the PDU to the
pump. If persistence fails, do not queue it. Failed or uncertain sends
must burn reserved sequence numbers; storage failures must prevent
transmission.

Segmented messages need special coverage. `MeshNode` allocates the
whole SEQ block before its first callback, so that first callback can
reserve the final next-to-use cursor for every segment. Subsequent
segment callbacks should not cause redundant disk commits.

### 2. Atomic replacement is not durable persistence

`save_state()` does not flush and `fsync` the temporary file before
rename, and does not `fsync` the parent directory after rename
(`src/sphere_mesh/provision.py:274-280`). A host crash can therefore
lose either file contents or the directory entry despite the atomic
replacement. The existing test proves only final contents and removal
of the temporary name after a successful call
(`tests/test_provision.py:21-28`).

For a crash-safe store, write the complete new state, flush and sync it,
set private permissions, replace the old file, then sync the directory.
Document whether “crash” includes power loss; Stage 2 should assume it
does because SEQ reuse is a cryptographic nonce-reuse issue.

### 3. Multiple writers can roll the cursor back

The temporary filename is fixed and there is no file lock. Two harness
or daemon processes can overwrite each other's temporary file or commit
states based on stale in-memory dictionaries. The last writer can move
`seq` backwards.

Use a daemon-lifetime exclusive lock, and make CLI operations that use
the same provisioner identity fail clearly while that lock is held.
Atomic rename alone does not prevent lost updates.

### 4. Session close leaves protocol timers alive

`MeshProxySession.close()` stops transport objects but never calls
`self.node.close()` (`src/sphere_mesh/transport.py:246-252`). A pending
Segment Ack timer from the old node can fire after reconnect, allocate
and persist a sequence number, and enqueue onto its stopped pump. It
will never be transmitted, and the old object graph remains live until
the timer expires.

Call `node.close()` as part of session shutdown, with an idempotence and
timer-cancellation test. Outstanding request futures also need explicit
supervisor cancellation because `MeshNode.close()` only handles SAR
state.

### 5. Link-loss detection is incomplete

Pump failure detects GATT write errors, and bearer failure detects only
a subscription that fails after its startup grace period. Neither
detects an otherwise idle client disconnect. There is also no `closed`
property: a deliberately closed session can still report
`failure is None`.

Expose explicit connection states such as `connecting`, `ready`,
`backoff`, `stopping`, and `stopped`. Availability must require the
current session to be ready, not merely free of a recorded exception.

### 6. Proxy filtering must be an explicit decision

The project session ignores every Proxy PDU type except Network PDU and
does not initialize the Proxy accept list
(`src/sphere_mesh/transport.py:301-309`). The pinned upstream
`MeshController.start()` sends Filter Type and Add Address messages,
which consume SEQ values and change the on-air transcript.

The current Sphere has answered without that setup, but a fresh proxy
connection resets filter state. Stage 2 should test the hardware and
either configure the filter on every reconnect or document reliance on
the device's behavior. If enabled, those proxy configuration PDUs must
use the same ordered pump and durable SEQ reservation path.

### 7. Existing state validation is too weak for a daemon

`_validate_state()` checks presence and broad Python types only
(`src/sphere_mesh/provision.py:211-232`). It does not validate:

- SEQ, IV Index, key-index, or address ranges;
- the 16-byte DeviceKeys and shape of device entries;
- consistency between dictionary address keys and element ranges;
- a 24-bit SEQ exhaustion sentinel; or
- booleans masquerading as integers.

Bad state may consequently fail after a connection is already open or,
worse, be saved again by the unconditional final save
(`src/sphere_mesh/provision.py:964-969`). Validate and migrate before
starting Bluetooth.

### 8. The session accepts only one DeviceKey

`open_mesh_proxy_session()` registers one primary node DeviceKey
(`src/sphere_mesh/transport.py:299`). This is enough for the current
Sphere and Stage 2, but it is coupled to the harness's selected-device
workflow. Keep the daemon state/store able to register every known node
if the persistent proxy later serves more than one target.

## State migration design

Adding a required top-level `tid` field would make every existing live
state fail validation. The file has no schema version, so there is no
way to distinguish a legacy file from a partially written future one.

Introduce a versioned schema with an explicit migration from the
current unversioned form. A minimal logical layout is:

```text
schema_version
network keys, key indexes, IV Index
provisioner source address
protocol.seq_next
protocol.tid_next
nodes and composition
```

Migration rules should be conservative:

- preserve every existing key, address, node, and composition field;
- copy the existing top-level `seq` exactly as the next-to-use cursor;
- initialize a missing TID cursor once, commit it before daemon start,
  and never reinitialize it on reconnect;
- retain unknown fields until an intentional schema decision removes
  them; and
- make a recoverable backup before the first format rewrite.

A legacy file cannot reveal the last TID used by previous harness runs.
On first migration, either wait beyond the transaction deduplication
window before accepting Sets, or start with a randomized cursor and
still enforce that quiet period. The quiet period is the only rigorous
way to avoid colliding with an unknown recently used TID.

The next SEQ value can legally advance past `0xffffff` after consuming
the final value. Represent that as an explicit exhausted state or allow
the `0x100000` next-cursor sentinel. Do not wrap to zero under the same
IV Index. IV Update handling is outside the present implementation and
must fail clearly rather than reset the cursor.

The initial one-Sphere daemon should reserve one PDU at a time. Persist
and sync the next cursor before queueing the PDU. This trades write
amplification for a simple invariant suitable for the expected command
rate: anything that might have reached the air is below the durable
cursor. Range reservations remain a later optimization if profiling
shows that synchronous storage is a problem.

Roll-forward probing from a slightly uncertain cursor is a manual
recovery option, not the normal persistence policy. It can recover
availability but may reuse encryption nonces and cannot distinguish
replay rejection from radio loss. Missing or badly stale cursor state
should instead move the controller to a fresh source address while
retaining the keys.

## Test gaps and required cases

The current 25 project tests pass with `./.venv/bin/pytest -q`.
Existing coverage proves:

- initial connect retry and subscribe-failure cleanup
  (`tests/test_transport.py:43-86`);
- retry exhaustion, invalid attempt counts, and tolerant disconnect
  (`tests/test_transport.py:89-120`);
- one unsegmented send advances a callback from SEQ 10 to 11, plus
  idempotent session close (`tests/test_transport.py:133-165`);
- invalid-key construction cleanup
  (`tests/test_transport.py:168-193`); and
- successful state replacement and private file mode
  (`tests/test_provision.py:11-28`).

No current test covers:

- a connection dropping after a successful open;
- a waitable pump, bearer, or client-disconnect signal;
- cleanup-before-reopen, backoff, cancellation, or repeated failure;
- the invariant that at most one session generation is live;
- command behavior while connecting or disconnected;
- persistence failure preventing queueing and transmission;
- ordering between durable reservation and pump insertion;
- segmented-message sequence reservation;
- restart monotonicity across two store/session instances;
- 24-bit sequence exhaustion or invalid cursor values;
- TID persistence, restart behavior, wrap, or migration;
- writer exclusion and stale-state lost updates;
- interrupted write/replace recovery and directory durability;
- `MeshNode` timer cleanup at session close; or
- scanner cleanup when target resolution or session opening fails.

Use deterministic fakes rather than Bluetooth hardware for the failure
matrix. Add a scripted connection that can open, trigger a disconnect,
record close ordering, and later permit a replacement. Inject a fake
clock/backoff sleeper so tests never wait in real time.

## Recommended implementation and test order

1. Define the versioned state model and migration in a standalone
   module. Add legacy migration, range validation, missing/corrupt file,
   permissions, backup, and unknown-field tests.

2. Implement an exclusive, durable state writer. Add injected failures
   at write, sync, replace, and directory-sync points, plus a two-writer
   exclusion test.

3. Implement cursor allocators independently of Bluetooth. Prove
   persist-before-use, storage-failure fail-closed behavior, SEQ range
   reservation, restart monotonicity, exhaustion, TID advance, TID
   wrap, and legacy TID quiet-period behavior.

4. Tighten the one-session lifecycle. Add `node.close()`, explicit
   `closed` state, a waitable lost event, client-disconnect integration,
   and rejection of sends after close or pump failure. Test every
   startup and shutdown failure point.

5. Add the connection supervisor around verified Network ID discovery
   and the existing session opener. Test initial success, scan failure,
   connect failure, post-connect loss, cleanup-before-reconnect,
   exponential backoff with jitter bounds, cancellation during each
   state, and the single-live-session invariant.

6. Add generation-scoped command admission and cancellation. Test a
   request interrupted by disconnect, a command during backoff, and a
   command racing with session replacement. Serialize same-response
   requests initially.

7. Route every network PDU, including Segment Acks and optional Proxy
   Filter Configuration, through the durable sequence allocator. Route
   every acknowledged and unacknowledged Set through the durable TID
   allocator.

8. Add an integration-style fake test: start, send, disconnect,
   reconnect, send, terminate, reconstruct from disk, and send again.
   Assert that neither cursor regresses or reuses a value and that only
   the active generation transmits.

9. Adapt the provisioning harness to the new store/session contracts
   while preserving its command transcript. Re-run the full unit suite,
   then perform a read-only hardware reconnect soak before enabling Set
   operations.

## Stage 2 readiness gate

Stage 2 is ready for hardware validation when all of these statements
are true:

- storage failure prevents transmission;
- a process or host crash cannot cause SEQ or TID rollback;
- only one process owns the provisioner identity and mutable cursors;
- remote disconnect wakes the supervisor without needing a command;
- the failed session is fully closed before its replacement is ready;
- old commands and timers cannot transmit through an old generation;
- every reconnect selects a verified target and restores any required
  Proxy Filter state; and
- restart, reconnect, segmented traffic, cursor boundaries, and
  cancellation are covered without a physical adapter.
