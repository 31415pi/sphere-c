# Host-side animation over Mesh Proxy/GATT

## Recommendation

Host-driven animation is feasible for modest-rate, single-Sphere effects,
but the current controller API is not an animation transport. Start with
unacknowledged HSL or Lightness keyframes at a configurable 10 Hz, allow an
experimental ceiling around 20 Hz, and periodically sample state with a Get.
Use latest-value-wins backpressure so a slow link drops stale frames rather
than playing an ever-growing backlog.

Do not initially promise video-rate animation, exact per-frame delivery, or
long unattended continuous playback. A 30 Hz experiment may work on the
direct GATT link, especially at the captured ATT MTU, but it is not established
by the present code or captures. Rates at or above 43 new transactions per
second are invalid for one source/destination/model transaction scope because
the eight-bit TID wraps inside the six-second duplicate window.

For fades and other simple motion, adding the standard transition-time fields
is preferable to streaming every intermediate value. Firmware interpolation
reduces traffic, storage writes, and jitter. Complex effects can use streamed
unacknowledged keyframes.

## Current data path

The runtime path is:

```text
lighting/control codec
  -> MeshNode.send_access() or MeshNode.request()
  -> encrypted Network PDU
  -> BearerPump's unbounded, single-consumer queue
  -> GattBearer Proxy-SAR framing
  -> GATT Write Without Response to Mesh Proxy Data In
```

`BearerPump` correctly serializes all writes, so Proxy SAR frames from two
messages cannot interleave. `GattBearer` sizes each frame from
`client.mtu_size - 3`, and falls back to a 20-byte GATT value at the minimum
23-byte ATT MTU. Every write uses `response=False`; completion of the Bleak
call is not an application-level delivery acknowledgment.

For direct Linux connections, `LocalTransport` asks Bleak's BlueZ backend
to acquire the negotiated ATT MTU before the bearer reads `mtu_size`. If
BlueZ cannot acquire it, the backend uses the conservative 23-byte
minimum and logs a warning. ESPHome proxy connections use their own
backend's MTU reporting.

The captured phone connection used an 18.75 ms connection interval, latency
zero, and a five-second supervision timeout. All captured Proxy messages had
Proxy SAR `Complete`, including 29-byte Network PDUs. A 29-byte Network PDU
plus the one-byte Proxy header requires a 30-byte GATT value and therefore an
ATT MTU of at least 33. The current bearer will adapt if a new connection
negotiates less, but each lighting frame then costs more GATT writes.

Local sources:

- `src/sphere_mesh/transport.py`
- installed pinned dependency `btmesh/bearer.py`, `btmesh/pump.py`,
  `btmesh/proxy_pdu.py`, and `btmesh/node.py`
- `docs/analysis/capture.md`
- `old/analysis/proxy_pdus.csv`

## Message sizes

For these unsegmented, AppKey-authenticated access messages with a 32-bit
TransMIC, the Network PDU is 18 bytes plus the access payload. The lower
transport permits up to 11 access bytes without Mesh lower-transport
segmentation (`15` upper-transport bytes minus the four-byte TransMIC).

| Set, no transition | Access bytes | Network PDU | Proxy value | GATT writes at ATT MTU 23 | Mesh lower-transport |
|---|---:|---:|---:|---:|---|
| Generic OnOff | 4 | 22 | 23 | 2 | unsegmented |
| Light Lightness | 5 | 23 | 24 | 2 | unsegmented |
| CTL Temperature | 7 | 25 | 26 | 2 | unsegmented |
| HSL | 9 | 27 | 28 | 2 | unsegmented |

“Proxy value” includes the one-byte Proxy SAR/type header. At ATT MTU 31 or
larger, the current nine-byte HSL Set fits in one GATT write. The observed
capture's effective MTU was sufficient for all of these in one write.

Adding the optional transition-time and delay bytes makes HSL 11 access bytes,
a 29-byte Network PDU, and a 30-byte Proxy value. It still fits one
unsegmented Mesh access message and fits one GATT write at ATT MTU 33 or
larger. No proposed standard lighting animation message needs the dependency's
incomplete outbound Mesh segmentation support.

Acknowledged and unacknowledged Set encodings have equal length: only the
two-byte opcode changes. Status traffic is the material difference.

## Acknowledged versus unacknowledged frames

Every public controller setter currently calls `MeshNode.request()` with the
acknowledged Set opcode. The dependency default is one retry after a five
second timeout, so a lost reply can occupy that call for about ten seconds.
Retransmissions reuse the already encoded TID, correctly identifying the same
logical transaction, while consuming a fresh network sequence number.

Sequential acknowledged frames are limited by the full Set/Status round trip.
In the old capture, isolated request-to-first-response candidates commonly
took about 70--110 ms. That implies only roughly 9--14 sequential acknowledged
updates per second before application work and jitter, and each update also
produces inbound Status traffic. The capture is from a different network and
its encrypted requests cannot all be identified as HSL, so this is a planning
range, not a benchmark.

Pipelining acknowledged Sets is a poor substitute. Mesh access replies have
no correlation identifier. `MeshNode` pairs concurrent waiters by source and
response opcode in FIFO order, so loss, firmware coalescing, or delayed Status
can associate a result with the wrong frame. It also makes animation pace
dependent on response behavior.

The codecs already support unacknowledged opcodes:

- upstream `generic_onoff_set(..., ack=False)`;
- upstream `light_lightness_set(..., ack=False)`;
- upstream `light_ctl_temperature_set(..., ack=False)`;
- local `light_hsl_set(..., ack=False)` and `light_xyl_set(..., ack=False)`.

The project controllers do not expose them. An unacknowledged frame should use
`MeshNode.send_access()`, not `request()`. It produces no Model Status and has
no access-layer retry. GATT is also written without ATT response. Therefore a
successful enqueue means “accepted locally for sending,” not “rendered by the
lamp.” This is appropriate for disposable intermediate animation frames, but
not for a final state whose confirmation matters.

A useful policy is:

1. stream intermediate frames with Set Unacknowledged;
2. use a periodic Get, for example once per second, for health and actual
   state sampling;
3. finish an effect with an acknowledged Set or a final Get/readback; and
4. on reconnect, discard old frames and send only the newest desired state.

## Sequence-number and TID budget

### Network sequence numbers

Each unsegmented animation Set consumes one 24-bit network SEQ. Retries,
proxy-filter configuration, and generated segment acknowledgments consume
additional values. The session persists the advanced cursor synchronously
before queueing each Network PDU, which is safe but expensive for animation:
`save_state()` writes and `fsync()`s a temporary JSON file, renames it, and
`fsync()`s the containing directory for every PDU.

At one PDU per frame, ignoring other traffic:

| Frame rate | SEQ consumption | Full 24-bit space |
|---:|---:|---:|
| 10 Hz | 36,000/hour | about 19.4 days |
| 20 Hz | 72,000/hour | about 9.7 days |
| 30 Hz | 108,000/hour | about 6.5 days |

The stack has no IV Update implementation, so exhaustion is a hard boundary,
not a wrap point. Continuous host streaming is consequently the wrong design
for an always-on ambient effect unless IV Update is implemented.

There is also a project bug that makes the immediate limit much lower:
`MAX_NEXT_SEQUENCE` in `src/sphere_mesh/state.py` is `0x100000`, while the
24-bit exhausted next-cursor sentinel is `0x1000000`. With the current check,
state persistence fails after only 1,048,576 sequence numbers: about 29 hours
at 10 Hz, 14.6 hours at 20 Hz, or 9.7 hours at 30 Hz. Correcting that constant
is required before any long animation run, but does not replace IV Update.

### Transaction IDs

Every Set, acknowledged or not, carries an eight-bit TID. Within the same
source, destination, and transaction scope, reusing a TID inside six seconds
means “retransmission of the previous transaction,” not a new visual frame.
The existing `StateStore.allocate_tid()` reserves the next TID durably before
the caller sends, which is safe across crashes, but it performs another full
atomic save and sync per frame.

For a continuous animation to one model element, 256 distinct TIDs must span
at least six seconds. The strict average ceiling is therefore 42.67 new
transactions per second. A 30 Hz stream wraps after 8.53 seconds and is safe
on this rule; 60 Hz wraps after 4.27 seconds and is not. Bursty scheduling
must enforce the same rolling-window rule rather than relying only on an
average.

Skipped TIDs and SEQs are harmless. Reserving ranges ahead of use is thus a
safe optimization: a crash burns the unused tail instead of risking reuse.

## Throughput, pacing, and retry limits

The current code has no animation pacing, bounded queue, coalescing, queue
depth metric, or write deadline. `BearerPump.put()` always succeeds into an
unbounded `asyncio.Queue`. If rendering produces frames faster than GATT can
write them, latency grows without limit and the lamp displays stale motion
after the producer has moved on or stopped.

`GattBearer.send()` serially awaits every Write Without Response call. That
provides ordering and whatever local flow control the Bleak backend applies,
but it does not expose remote receipt. At minimum ATT MTU, every standard HSL
frame becomes two serialized GATT writes. At the captured MTU it becomes one.
The negotiated MTU and actual write latency should be measured on the current
Linux link before selecting a production rate.

`MeshNode.request()` retries acknowledged Sets once after the complete timeout.
Unacknowledged `send_access()` does not retry. The outbound segmented path
sends every segment once and records Segment Acknowledgments without using
them for selective retransmission. This is not a blocker for the standard
lighting messages above because all remain unsegmented, but it rules out
assuming reliable delivery for a future large vendor animation payload.

The Sphere was configured with Network Transmit count 5 and interval steps 2.
That setting matters when the node originates Network PDUs on an advertising
bearer. It should not be used as a host animation retry guarantee: the GATT
write is to the Sphere acting as the proxy, the destination is normally an
element on that same node, and the current host receives no proof that an
unacknowledged model command was applied.

## Minimal code changes

The smallest responsible prototype is limited in scope and does not require a
new Mesh transport:

1. Correct the sequence exhausted sentinel from `0x100000` to `0x1000000` and
   add boundary tests. Continue to fail closed at exhaustion.
2. Add explicit unacknowledged lighting methods, or a small animation sender,
   which allocate one TID and call the existing codec with `ack=False`, then
   call `MeshNode.send_access()`.
3. Give the animation sender access to pump completion or a session-level
   `flush()`/drain primitive for shutdown and final-frame handling. Do not
   stop a session immediately after enqueueing an unacknowledged frame.
4. Put a bounded, latest-value-wins scheduler in front of the sender. Keep at
   most one pending visual frame, serialize all Sets for a transaction scope,
   and pace them initially at 10 Hz. Record sent, replaced, and failed counts.
5. Reserve SEQ and TID ranges durably rather than syncing the whole state file
   twice per rendered frame. Keep unused reservations burned after a crash.
   The reservation interface must cover proxy configuration and automatic
   Segment ACK traffic too, not only animation calls.
6. Enforce a per-transaction-scope rolling TID window below 42.67 Hz and retain
   TID identity if a logical frame is deliberately retransmitted.
7. Add optional transition-time and delay encoding to standard lighting
   setters. Use transition keyframes when the desired effect can be expressed
   by firmware interpolation.
8. Add periodic readback and connection-loss behavior: pause generation while
   disconnected, discard backlog, reconnect, and apply only the latest target.

For an initial manual experiment, range reservation can be deferred if tests
show the synchronous state saves sustain 10 Hz without event-loop stalls, but
the latest-value-wins queue and unacknowledged send path should not be
deferred. For a long-running service, range reservation and IV Update planning
are required.

## Validation plan

Before raising the default beyond 10 Hz:

- log negotiated ATT MTU, GATT writes per frame, queue delay, and send latency;
- run 5, 10, 20, and 30 Hz ramps for a short bounded duration;
- verify periodic Gets continue to return and the final acknowledged state is
  correct;
- inject a slow bearer and prove frames are replaced rather than accumulated;
- disconnect mid-effect and prove no stale frame is replayed after reconnect;
- test TID wrap and the six-second rolling window with a deterministic clock;
- test SEQ/TID range reservation across crashes; and
- inspect physical smoothness, because HSL values are known to be requantized
  by Sphere firmware after acknowledged Sets.

The acceptance criterion should be bounded end-to-end latency and correct
final state, not delivery of every intermediate frame.
