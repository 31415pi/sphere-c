# Stage 2 Bluetooth Mesh protocol risks

## Scope and source baseline

This audit covers the requirements that should be settled before
Stage 2 of `docs/plans/daemon.md`: persistent connection and protocol
state. It focuses on the single, self-provisioned Sphere C that the
initial daemon is intended to control.

The dependency inspected was the installed `btmesh` 0.10.3 source and
the local checkout at the pinned commit:

```text
a8f7bbbebaeba124d7d08cb16122055074f2ef46
```

The dependency's tests at that checkout were also inspected. The
project code and tests were read from `src/`, `tests/`, and the current
guides. No behavior below should be inferred from a moving upstream
branch.

The audit found and corrected a documentation mismatch. The provisioner
module had said that segment acknowledgements were not transmitted.
The pinned dependency sends acknowledgements for segmented unicast
messages addressed to the local node, so that old limitation was not
accurate for the current `MeshNode`.

## Launch boundary

For the initial one-Sphere daemon, "must-have" means required to avoid
silent command loss, nonce or transaction reuse, accepting the wrong
device, false acknowledgement, or an uncontrolled reconnect loop.

The following are must-have before Stage 2 is considered complete:

- reserve transmit sequence numbers durably before transmission;
- reserve each logical Set transaction ID before transmission;
- serialize access to the one provisioner identity and state file;
- reject replayed inbound network messages;
- authenticate the Sphere's Node Identity advertisement;
- configure the proxy filter on every new connection;
- retain protocol cursors across every reconnect;
- detect dead links and reconnect with bounded backoff;
- fail safely on an IV Index mismatch or sequence exhaustion; and
- store keys and mutable protocol state durably and privately.

Full IV Update participation, reliable outbound segmented transfers,
multiple nodes, multiple subnets, key refresh, and recovery from a
damaged state store can be later hardening. The initial daemon still
needs explicit, safe failure behavior for these cases; it must not
guess and continue.

## Transmit sequence numbers

### What exists

`btmesh.network.NetworkContext.next_seq()` allocates either one number
or a consecutive block. It rejects allocation beyond `0xffffff`.
`MeshNode.send_access()` allocates one number for an unsegmented PDU and
one number per segment for segmented traffic. Segment acknowledgements
and proxy configuration messages also consume sequence numbers.

The project passes a stored cursor into `MeshNode`. Its send callback
queues a PDU and then calls `persist_seq(node.ctx.seq)`. The provisioning
harness writes the full JSON state through a temporary file and
renames it. Project tests prove that a normal access send advances and
persists the cursor, and that the resulting file has mode `0600`.

The dependency tests cover single and block allocation, exhaustion,
and the fact that proxy configuration consumes a sequence number.

### Gaps and risks

The current callback performs `pump.put(raw)` before persistence. The
event loop normally cannot drain the queue until the synchronous
callback returns, but this is an implementation-order assumption. A
persistence exception leaves a queued PDU and an ambiguous cursor.

The JSON replacement is atomic as a namespace operation but not fully
durable. Neither the temporary file nor its containing directory is
`fsync`ed. A power loss can therefore restore an older cursor even
though the process previously observed a successful save.

Persisting the exact next number before transmission is sufficient for
normal crash recovery if the write is durable and exclusive. A state
backup rollback, copied state file, filesystem loss, or a second daemon
using the same source address can still reuse a nonce. Peers will also
discard those PDUs as replays.

The higher-level `MeshController.start()` consumes two sequence numbers
when it configures the proxy filter. It puts those PDUs directly on its
pump. A future daemon that reads `controller.seq` only after a user
command would miss numbers consumed merely by connecting or by sending
a segment acknowledgement.

### Required initial design

Use a one-PDU write-ahead reservation, not an after-send checkpoint.
Before a PDU can be queued, atomically store and durably sync the next
cursor. Apply this to access messages, filter configuration, segmented
blocks, and segment acknowledgements. A crash may burn a sequence
number; it must not make one reusable.

An explicitly requested emergency tool may advance from a slightly
uncertain cursor using harmless acknowledged reads until one succeeds.
Each unsuccessful probe may reuse an encryption nonce and a missing
response is ambiguous, so this must not be automatic startup behavior.
For missing or badly stale state, preserve the keys but move the
controller to a fresh, non-overlapping source address. Loss of the keys
requires reset and reprovisioning.

Hold an exclusive process lock for the provisioner state. Only one
process may transmit with a given `(NetKey, IV Index, source address)`.
Validate the stored cursor range and refuse startup on corruption or
exhaustion. State restoration must be documented as security-sensitive:
restoring an old cursor without changing identity is unsafe.

Tests should crash or inject failure at each boundary: before reserve,
after durable reserve, after enqueue, during write, and after send.
Restart must never allocate a number below the durable next cursor.
Tests must include filter PDUs, access PDUs, segmented blocks, and
outbound segment acknowledgements.

## IV Index and IV Update

### What exists

The dependency parses and authenticates Secure Network Beacons. It
checks beacon type, Network ID, and the beacon authentication value
derived from the NetKey. `MeshController` exposes the last valid beacon
and warns when its IV Index differs from the configured index.

The network codec itself supports exactly one fixed IV Index. It checks
the incoming IVI bit against that index and uses the fixed index for
network and upper-transport nonces. The controller deliberately does
not adopt a beacon's value. The project state has an `iv_index` field,
but the reusable session ignores beacon proxy messages and has no IV
transition logic.

### Gaps and risks

A stale IV Index causes silent bidirectional failure. Merely matching
the one-bit IVI field is insufficient because indices two apart have
the same bit. Blindly copying any authenticated beacon value is also
not a complete IV Update implementation. Transition and recovery rules,
old-index receive handling, and the sequence-number relationship must
be enforced together.

The pinned library has tests for authenticated and tampered beacons,
the IV Update flag, and an explicit IV Index override. It has no state
machine that adopts an index, accepts the permitted old/current pair,
or persists an IV transition atomically with transmit state.

### Required initial design

At launch, receive and authenticate Secure Network Beacons. If the
stored IV Index agrees, operation may continue. If it differs or the IV
Update flag requires behavior not implemented by the daemon, mark the
mesh unavailable with a precise diagnostic. Do not transmit using a
guessed index and do not silently rewrite state.

Persist the IV Index, IV Update phase if supported, and associated
sequence reservation as one state transition. A controlled operator
recovery path may be acceptable for the first one-Sphere release. It
must validate a fresh authenticated beacon and preserve the old state
for diagnosis without ever rolling the live cursor backward.

Later hardening should implement the full IV Update and IV Index
Recovery rules, including valid monotonic movement, permitted receive
indices during transition, timers, restart behavior, and sequence reset
only where the Mesh specification permits it.

## Receive replay protection

### What exists

`MeshNode` explicitly does not implement a Replay Protection List. It
stores only the last sequence number seen per source and rejects only an
immediate equal number. Lower sequence numbers and older non-adjacent
messages are accepted. The source comment describes this as a tradeoff
to tolerate a peer that incorrectly restarts its sequence number.

There are no dependency or project tests for a standards-compliant
receive replay window. The current bounded `received` deque is message
history, not replay protection.

### Risk and required initial design

An old authenticated Status can satisfy a new request because request
matching uses only source address and opcode. That can make an
acknowledged command appear successful with stale state. Accepting
lower sequence numbers also contradicts Mesh replay protection.

Implement a Replay Protection List keyed at least by source address and
IV Index, with the specification's old/current-IV behavior. Persist it
across daemon restarts, or reserve a conservative policy that cannot
accept old traffic after restart. For one Sphere, the storage cost is
small and is launch-critical.

Do not weaken replay protection to accommodate a node that resets its
sequence illegally. Treat that node as unavailable and require an
explicit recovery or reprovisioning path. Tests should cover equal,
lower, out-of-order, old-IV, new-IV, relayed duplicate, and restart
cases. A replayed Status must never resolve a live request.

## Transaction IDs

### What exists

The dependency's `MeshController` holds an eight-bit TID cursor and
increments it once per logical Set. Request retransmissions reuse the
already encoded payload, and therefore reuse the same TID while using
fresh network sequence numbers. That is the correct distinction between
a retransmission and a new transaction.

The constructor accepts a TID seed, and dependency tests cover seeding
and incrementing across commands. The project does not use this
controller. Its feasibility toggle loop starts TID at zero on every run
and increments only after a successful response. Project state has no
TID field and no tests for crash or reconnect behavior.

### Required initial design

Persist the next logical TID before a Set may be sent. Reconnects and
retries of that same logical operation must retain its TID; a later
logical command must allocate another. Serialize Sets that could share
the same transaction scope so two callers cannot allocate the same TID.

Because TID is only eight bits, persistence alone cannot prevent reuse
after wrap. Track enough destination, model/operation, and time context
to enforce the Mesh duplicate window, or rate-limit allocation so a TID
cannot wrap into the same transaction scope inside that window. Tests
should cover crash-before-send, crash-after-send, timeout and retry,
reconnect retry, concurrent callers, and wrap from 255 to zero.

A skipped TID after a crash is harmless. Reusing one for a distinct Set
inside the duplicate window can cause the Sphere to ignore the command
as a retransmission, so this is required for initial control reliability.

## Proxy identity and address filter

### Identity discovery

The scan helper accepts only advertisements observed after scan start,
which avoids BlueZ cache staleness. It validates a Network ID advert by
equality with `k3(NetKey)`. It parses Node Identity as an opaque
`Hash || Random` value but does not verify the hash.

The project provisioning flow accepts a sole Node Identity advertiser
and states that it is merely assuming the device is ours. The successful
Sphere run says the device actually advertises Node Identity. No local
dependency helper or test derives the IdentityKey and verifies that
advert against the configured node address.

Before an unattended daemon connects, implement Node Identity
verification using the subnet IdentityKey, advertised random value,
and each configured primary unicast address. Add official-vector tests,
negative address/key/hash tests, and multiple-advertiser tests. Do not
retain the single-candidate fallback.

A matching Network ID proves subnet membership, not a particular node.
For the initial one-node subnet it is a reasonable proxy candidate, as
any proxy on that subnet can carry mesh traffic. If the daemon promises
to connect directly to the Sphere, require verified Node Identity or a
subsequent authenticated node check rather than trusting the BLE address.

### Proxy filter

A new Proxy connection starts with an empty accept list. The dependency
controller correctly sends Set Filter Type followed by Add Addresses
for the local provisioner address. Both use the ordered pump before
access commands. It records Filter Status when one arrives but does not
wait for it, because the tested lamp applies the filter without replying.

The project's `open_mesh_proxy_session()` uses `MeshNode` directly and
does not configure a filter or dispatch Proxy Configuration messages.
Its tests exercise access transmission but do not assert filter setup.

Configure the accept list on every connection and reconnect before
declaring readiness. Persist sequence reservations for both filter
messages. Preserve their ordering ahead of the first request. Because
the Sphere may omit Filter Status, use a small acknowledged read as the
end-to-end readiness check and report filter status when present. Never
carry assumed filter state from an old GATT connection.

## Segmented messages and acknowledgements

### What exists

The pinned dependency can segment outbound access messages and
reassemble inbound ones per source. It supports out-of-order and
duplicate segments, bounded source tracking, acknowledgement and
incomplete timers, immediate completion acknowledgement, and no
acknowledgement for group-addressed traffic. Closing the node cancels
its reassembly timers.

Its tests cover these receive behaviors, concurrent sources, timer
cleanup, block-ack construction, and a segmented controller response.
Every generated acknowledgement consumes a sequence number through the
same node send callback.

Outbound reliability is incomplete. The library records received
Segment Acknowledgements but does not connect them to a transmit state
machine. It sends all outbound segments once, does not wait for block
acknowledgement, and does not selectively retransmit missing segments.

### Launch boundary

Initial on/off and ordinary status messages fit unsegmented access
transport, so complete outbound segmented reliability can follow Stage
2. Inbound segmented handling is already sufficient for composition and
other long unicast responses, provided its acknowledgement sequence
numbers use the new durable allocator.

Stage 2 tests should nevertheless prove that disconnect closes all SAR
timers, a reconnect starts with no stale assembler or ACK state, and an
ACK generated without a user command is durably accounted for.

Before long configuration/vendor messages become daemon features,
implement outbound SAR state, acknowledgement timers, block-ack
matching by peer and transfer, selective retransmission, cancellation,
and terminal failure. A stored ACK indexed only by `SeqZero` is not
enough to correlate concurrent peers or reused sequence-zero values.

## Reconnect behavior

### What exists

`open_gatt_session()` retries initial connect or subscribe setup three
times with a fixed delay and cleans up failed clients. The bearer records
a late subscription failure. The ordered pump records a GATT write
failure and then stops permanently. `MeshProxySession.failure` exposes
these two failure sources.

There is no long-running connection supervisor, disconnect callback,
automatic reconnect, exponential backoff, readiness state, or project
test for an unexpected drop. The session cleanup also does not call
`MeshNode.close()`, so pending segmented-message timers can outlive that
session until the object is collected.

### Required initial design

Use one supervisor and one connection attempt at a time. Detect the
Bleak disconnect callback, bearer subscription failure, pump failure,
and failed health/read probes. On loss, immediately stop accepting new
operations, fail or classify pending requests, close the node and all
timers, stop the pump and bearer, and disconnect the client.

Reconnect using fresh advertisements, bounded exponential backoff with
jitter, and a cap. Advertisement callbacks and periodic probes must
share the same backoff gate so they cannot create a storm. Shutdown and
intentional disconnect must suppress reconnect. A successful,
authenticated, filter-configured, acknowledged probe should reset the
backoff and mark the daemon ready.

Carry the durable sequence and TID allocators across sessions. Do not
reconstruct either cursor from the value used at the previous connect.
Filter setup and other reconnect traffic also spend sequence numbers.

For a Set interrupted by disconnect, retain its logical TID if it is
retried. Because delivery may have occurred before the link failed,
prefer an acknowledged retry with the same transaction identity or a
read-back policy. Never silently turn the uncertainty into a new Set
with a new TID and report it as exactly-once execution.

Tests should cover a drop during idle, request, filter setup, outbound
write, inbound segmented transfer, backoff, recovery, and shutdown.
They should also cover an unplugged Sphere advertising again later and
many rapid advertisements while a connection attempt is failing.

## Secure and durable state

The current single JSON file mixes NetKey, AppKey, device keys,
addresses, IV Index, and sequence state. Mode `0600` is tested, and
temporary replacement prevents readers from seeing partial JSON.

For the daemon, create the directory privately as well, reject symlinks
and unexpected ownership where practical, and preserve `0600` on every
replacement. Durably sync the file and directory. State parsing should
validate integer ranges, key lengths, address classes, uniqueness, and
schema version, not only Python value types.

Mutable security state should include at least:

- a schema/version marker and provisioner identity;
- IV Index and any supported IV Update phase;
- the durable next transmit sequence cursor;
- the next TID plus duplicate-window context; and
- the receive Replay Protection List.

Secrets should never appear in normal logs, diagnostics, process
arguments, or API responses. Backups must be treated as credentials and
must not be restored independently of monotonic protocol state. If
secrets and fast-changing counters are separated later, bind their
generation/identity so a counter file cannot be paired with the wrong
keys.

Failure to persist a reservation must prevent transmission. A corrupt,
missing, rolled-back, or permission-unsafe state store should make the
daemon unavailable with recovery instructions, not cause it to create
new keys or reset counters automatically.

## Test and documentation gaps

The existing project tests are good Stage 1 transport tests but do not
yet establish Stage 2 protocol safety. Add tests for:

- durable SEQ and TID reservation across injected crashes;
- allocation by proxy filter and segment-ack traffic;
- single-writer locking and concurrent commands;
- Replay Protection List behavior and persistence;
- authenticated Node Identity selection;
- beacon mismatch and controlled IV migration;
- per-connection filter setup and readiness probing;
- disconnect detection, cleanup, backoff, and reconnect; and
- uncertainty and same-TID retry for an interrupted Set.

Update the Phase 0 limitation text to reflect the pinned dependency's
inbound segment acknowledgements. Keep the remaining limitations clear:
no full IV Update state machine, no standards-compliant RPL today, and
no reliable outbound segmented-transfer state machine.

## Recommended Stage 2 order

1. Define a versioned, locked, durable state store and reservation API.
2. Route every sequence consumer and logical TID allocation through it.
3. Add a standards-compliant receive Replay Protection List.
4. Verify Node Identity and make proxy candidate selection fail closed.
5. Configure the proxy filter and prove readiness on every connection.
6. Add the supervised reconnect lifecycle and interruption semantics.
7. Authenticate beacons and fail safely on unsupported IV transitions.
8. Exercise crash and reconnect boundaries before exposing control APIs.

Full IV Update participation and outbound segmented-message reliability
can follow as explicit hardening milestones, provided the initial daemon
detects those conditions and refuses unsafe operation.
