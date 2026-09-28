# Effect activation timeout runtime audit

## Scope and evidence

This began as a source and saved-record audit of a reported timeout
after changing from Flame to Cotton Candy. Later live trials exercised
the transaction but did not reproduce that particular failure. The
repository has no decoded trace or daemon journal for the failing
transition, so its exact cause is not established.

The effect names resolve to DP 102 enum values `00` and `01`.
`effect.activate cotton-candy` sends two sequential acknowledged Tuya
vendor writes to model `0x07d0:0x0004`:

```text
c9 d0 07 01 02 04 01   # DP 2 = colour work mode
c9 d0 07 01 66 04 01   # DP 102 = Cotton Candy
```

The service holds one command lock across both operations, so its own
vendor requests do not overlap. Each write asks `MeshNode.request()`
for DATA opcode `0xcdd007`, with a five-second per-attempt timeout.
The node waiter matches only source element and opcode, not the
command-type byte or DP ID. The vendor controller then waits 100 ms
and examines all newly received DATA from that element. See
`src/sphere_mesh/daemon.py` (`handle`, `_effect`),
`src/sphere_mesh/vendor.py` (`TuyaVendorController._request`), and
`old/vendor/ha-bluetooth-mesh/src/btmesh/node.py` (`_deliver`,
`request`).

The live activation record in `effect_write_experiment_plan.md`
confirmed the DP 2 write and a subsequent DP 102 write for Color
Wheel; both returned exact typed DATA echoes. A separate DP 102
selection of Aurora also echoed, read back, and visibly animated. That
validates the general wire encoding and sequential activation path,
though it does not prove how the Sphere responds to every effect or
every repeated same-value DP 2 write.

## What the timeout can mean

If the service returned `unavailable` with `Sphere element ... did
not answer vendor DP Write`, a `MeshNode.request()` received no DATA
with the expected source and opcode within five seconds. The write
may still have reached and changed the Sphere; a missing application
reply, a lost mesh/GATT reply, or a disconnected session cannot be
distinguished from that error alone. The original code disabled
automatic write retransmission.

The error details now distinguish both steps. If `failed_step` is
`effect` and `completed_steps` contains
`mode_transport_acknowledged`, the first write received DATA and the
second write timed out. If `failed_step` is `mode`, DP 102 was not
attempted. An independent `effect.get` afterward can reveal DP 102's
final value. It cannot necessarily verify DP 2 because the observed
read snapshots omit it.

If the desktop app instead displays `Service did not respond in time;
the result is uncertain`, that is its separate 30-second socket
deadline for `effect.activate`. It can include daemon queueing or a
stalled session, and cannot identify which write was active. An app
readback after a returned mutation error is also a separate request;
its failure should not be conflated with the activation error. See
`src/sphere_mesh/gui/backend.py` (`_mutate`, `_on_timeout`,
`_read_effect`).

## Relation to DATA command type `00`

The recent `vendor_type00_runtime_audit.md` and
`apk_vendor_response_race.md` identify a real response-selection
problem: the first DATA from the element can complete a pending
request even when its first parameter byte is `00`. The APK dispatches
ordinary DP DATA types `01` and `05` to the typed DP parser and routes
`00` to a generic status. Our controller instead parses all DATA in
the burst as typed DPs, so a `00` message raises `protocol_error`.

That problem does not directly explain a **vendor five-second
timeout**: a `00` DATA with matching source and opcode would resolve
the waiter. It could explain a protocol error on the write or on the
subsequent readback. A `00` from another source, with a different
opcode, or outside the response window would not resolve this waiter.
The existing saved live read had an empty type-`01` DATA followed by a
typed snapshot; no captured `00` sequence establishes that it occurs
during the reported effect transition.

## Current mitigation

The controller now filters vendor DATA to command type `01` for writes
and accepts only an empty acknowledgment or a typed response mentioning
the DP being written. It waits within the original deadline when a late
echo for a previous write arrives, so that echo cannot satisfy the next
operation.

After DP 2 acknowledges, `effect.activate` waits another 250 ms before
sending DP 102. If that DP 102 write alone receives no response, it is
repeated once. Repeating the same one-byte effect selection is
idempotent; DP 2 is never repeated, and a contradictory typed response
is reported rather than retried. Trace records include the attempt
number, while returned failure details report the total DP 102 attempts.

The root agent subsequently exercised twelve live effect transitions,
including three solid-color-to-effect changes, without reproducing the
failure. This verifies the normal transaction path but not the cause of
the earlier intermittent timeout; the running daemon did not yet
contain this mitigation during those trials.

## Response-selection rationale

Keep writes and reads distinct. For reads, filter to ordinary
command types `01`/`05` within a bounded receive window, then parse
and merge typed records; preserve the empty type-`01` acknowledgment
while waiting briefly for a snapshot. An unsupported type should not
turn an otherwise valid later snapshot into a protocol error. If no
ordinary DATA arrives, report no usable DP response. Do not globally
teach the DP parser to decode type `00` as ordinary records.

For writes, an unrelated type-`00` DATA is not confirmation. A type-`01`
response is eligible when empty or when it mentions the requested DP.
An empty response remains only a transport acknowledgment. A typed echo
of a different DP does not prove that the current write succeeded.
Because the protocol has no vendor transaction ID, exact state is still
best supported by a separate readback.

To localize the reported timeout, inspect the exact API error code,
message, and `details`, then record a timestamped decoded trace of
the two outgoing WRITE payloads and incoming DATA messages (source,
opcode, full parameters) plus connection state. A read-only
`effect.get` is the safest state check after an uncertain result. The
bounded DP 102 retry is intentionally narrower than repeating the whole
activation.
