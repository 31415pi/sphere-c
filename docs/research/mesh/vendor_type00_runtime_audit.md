# Tuya vendor DATA command-type `00` runtime audit

## Question

Why could a Tuya/Olight vendor read surface DATA command type `0x00`
even though the earlier live DP 103 battery read succeeded, and could
that `0x00` be an unrelated/intermediate response?

## Finding

The code has a real exposure to a matching but semantically unrelated
DATA message ending a request. The pinned `MeshNode.request()` waiter
matches only response source and access opcode. It has no predicate
for vendor command type or requested DP IDs. Thus the first
`0xCDD007` DATA from the target vendor element resolves the waiter,
even if its first parameter byte is not an ordinary DP command type.

The controller then waits 100 ms and gathers every newly received DATA
message from that element, without first filtering command type. It
passes the gathered messages to `parse_tuya_dp_data()`. That parser
accepts only command types `0x01` and `0x05`, based on the Olight APK's
ordinary DP DATA dispatcher. A leading `0x00` therefore raises
`AccessError("unsupported vendor DATA command type 0x00")`; it does
not mean that a datapoint had Tuya type RAW. For a valid ordinary DATA
message, command type is `params[0]`, while a record's DP type is the
second byte of that record (`DP ID`, then `DP type`).

Consequently, a leading `0x00` could be an unrelated/special vendor
DATA status which happens to use the same opcode, or another earlier
vendor message delivered during the request window. If it precedes a
valid `0x01`/`0x05` DATA snapshot, the current gather-and-parse logic
will fail the whole read before using that later snapshot. This is a
plausible mechanism, not a confirmed explanation for a particular
runtime observation: the repository contains no authenticated capture
of a command-type-`0x00` Sphere DATA message.

## Evidence

- `docs/research/mesh/vendor_dp_response_codec.md` documents the
  `cc d0 07` READ / `cd d0 07` DATA pair and the body grammar. The Olight
  3.2.5 APK dispatches DATA command types `01` and `05` to its ordinary
  DP status parser; other command types select special vendor status
  classes or generic pass-through parsing. Its ordinary DATA parser
  starts record parsing after the first command-type byte.
- `docs/research/mesh/vendor_dp_live_query.md` records a successful
  read-only live query. The response began with empty DATA parameters
  `01`, followed by a snapshot beginning with `01`, and that snapshot
  contained DP 103 as `67 02 00000064` (100%). The report therefore
  demonstrates successful command type `01` and DP value type `02`;
  it does not demonstrate command type `00` or a reply to a requested
  DP 103 specifically (the firmware returned a broader snapshot).
- `src/sphere_mesh/vendor.py` sends a request expecting only
  `OP_TUYA_DP_DATA`, snapshots `node.received`, waits 100 ms after the
  request returns, and gathers new messages with matching source and
  opcode. It parses all of them before checking consistency. There is
  no `message.params[0] in (1, 5)` candidate filter in the controller.
- The vendored `old/vendor/ha-bluetooth-mesh/src/btmesh/node.py`
  `_deliver()` resolves the oldest waiter matching only `(src, opcode)`.
  Its `request()` documentation explicitly describes that matching
  rule and lack of correlation ID. A DATA command-type-`00` response
  can therefore wake the request just like an ordinary DP DATA reply.
- `tests/test_vendor.py` verifies normal DATA, empty type-`01`
  acknowledgments, snapshot bursts, and malformed/unsupported command
  types. It does not exercise an initial `00` message followed by a
  valid `01`/`05` snapshot or validate read candidate filtering.
- The tracked `docs/research/mesh/vendor_dp_live_query.md` is the
  repository's saved live-read record; no raw current-session capture
  establishes whether an observed `00` was an intermediate response.

## Safest evidence-based handling

For read operations, treat only DATA messages with command type `01`
or `05` as ordinary DP-state candidates. Ignore unsupported command
types while collecting within a bounded response window, then parse
and merge the supported messages. If no supported DP DATA arrives,
return a clear unsupported/no-DP-response error rather than decoding
the special status as datapoints or reporting it as a successful
snapshot. Keep source and opcode checks, and keep requests serialized
per node because the vendor payload has no transaction identifier.

Do not globally make the DATA decoder accept `00` as a third DP
command type: the saved APK evidence says command type selects among
different status parsers, and adding it to the ordinary DP grammar
could reinterpret special vendor payloads as DP records. Also avoid
discarding every empty DATA acknowledgment: the successful live read
itself used an empty `01` message before the snapshot, and writes
currently rely on empty `01` acknowledgments.

To confirm the runtime hypothesis, preserve one authenticated decoded
message trace with timestamp, source, destination, opcode, full
parameters, and request timing. A trace showing `cd d0 07 00 ...`
followed by `cd d0 07 01 ...` would establish the intermediate-message
case. Until then, filtering `00` from ordinary read candidates is a
defensive fix grounded in the existing APK dispatcher, not a claim
about the Sphere's exact `00` semantics.

## Implementation update

The controller now applies that defensive filter. Reads collect only
types `01` and `05`; writes accept only type `01`. A special status can
no longer poison a later ordinary response in the same bounded receive
window. If the window contains no supported DATA, the operation reports
that no supported response arrived. The ordinary parser remains strict,
and writes are still never retried automatically.
