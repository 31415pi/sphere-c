# Sphere C Linux control daemon

The goal is a reliable, locally controlled Linux service for a
self-provisioned Sphere C. Work proceeds in six independently testable
stages.

## 1. Reusable mesh transport

Extract the proven local-Bluetooth, GATT-session, retry, and cleanup
code from the provisioning harness. Keep provisioning behavior
unchanged and cover the extracted boundary with unit tests.

## 2. Persistent connection and protocol state

Maintain a long-running Mesh Proxy connection, reconnect when the
radio or Sphere drops it, and persist the sequence and transaction-ID
cursors before they can be reused after a crash.

Persist the next network sequence number and sync it before placing
each PDU on the transmit queue. This one-PDU write-ahead policy may
burn a number after a crash, but it must never reuse one. A slightly
uncertain cursor may be recovered manually by advancing through
read-only acknowledged probes, with an explicit nonce-reuse warning.
If the cursor is missing or badly stale, retain the mesh keys but move
the controller to a fresh, non-overlapping unicast address. If the
keys are lost, factory-reset and reprovision the Sphere.

Keep transaction IDs across reconnects. On a restart with an unknown
recent TID, waiting beyond the model transaction window is an
acceptable initial recovery policy.

## 3. Basic control surface

Expose read, on, off, and toggle operations. Commands must use
acknowledged messages, report the Sphere's returned state, and fail
clearly when the proxy or node is unavailable.

The first CLI surface accepts repeatable `--control` options so a batch
can share one GATT Proxy connection. This matters on the Sphere C: a
clean client disconnect can pause its connectable advertising window.
A 30-second scan showed that Network ID advertising resumes without a
physical wake, although a five-second reconnect scan can miss it.

## 4. Standard lighting models

Implement brightness, color temperature, and HSL operations against
the standard models present in the Sphere's Composition Data. Keep
xyL capability-gated: its codec is useful for other nodes, but the
observed Sphere C has no xyL Server. Read status back from the device
rather than assuming a Set succeeded.

## 5. Stable local API and service

Add a versioned local API, a command-line client, lifecycle and health
reporting, structured logs, and service supervision. Keep mesh keys
and mutable protocol state private on disk.

## 6. Olight-specific functionality

Capture and decode the two Olight vendor models to support patterns,
animations, saved settings, and other behavior not represented by the
standard Bluetooth Mesh lighting models.

The first Stage 6 slice implements the Tuya/Olight datapoint framing
used by vendor server `0x07d0:0x0004`, including strict typed response
parsing. A read-only service operation can query selected datapoints
without changing the lamp. Effect writes remain deliberately disabled
until a separately reviewed, reversible write experiment is approved.

Live reads confirmed the grammar and exposed a two-part response: a
short empty DATA acknowledgment followed by a segmented status
snapshot. Reads for DP 2, DP 102, and both together all returned the
same records for DPs 7, 101, 102, 103, and 104. DP 102 was enum 7
(`colour_7`); DP 2 was omitted. This indicates that the firmware treats
the requested-ID list as a status-refresh trigger rather than a strict
projection, or that DP 2 is not readable in the current state.

## Current status

Stage 1 is complete. The local adapter, GATT retry/cleanup behavior,
and live Mesh Proxy session now live in
`src/sphere_mesh/transport.py`. The provisioning and read-only control
harness uses that shared module and has been verified against the
physical Sphere without changing its light state.

The recovery path has also been exercised in both directions: Linux
reset the locally owned node, the Olight app claimed and controlled it,
the physical pairing gesture returned it to an unprovisioned beacon,
and Linux reclaimed it at `0x0118`. A separate connection then returned
an authenticated status using the retained local keys.

Stage 2 is complete with a versioned, exclusively locked state store,
write-ahead sequence and TID handling, active-node selection, complete
session shutdown, and reconnect supervision.

The initial Stage 3 CLI surface is complete. Repeatable `--control`
commands provide acknowledged read, on, off, and toggle operations and
print the returned present, target, and effective state. All four were
verified against the physical Sphere; a two-toggle batch ended with an
authenticated `present=0` readback. Wiring the same controller into the
long-running supervisor remains part of the daemon work.

The initial Stage 4 surface is complete. It has a tested controller
and CLI for acknowledged Lightness, CTL Temperature, HSL, and
capability-gated xyL Get/Set operations.
Each operation resolves the server's element from saved Composition
Data and reports the returned status. The observed Sphere C supports
the first three models and correctly rejects xyL locally. Live read
and reversible Set batches succeeded. HSL status can be requantized
by the firmware after an acknowledged Set, so the controller reports
later Get results without assuming that the requested tuple persisted.

The initial Stage 5 service is operational. `sphere-meshd` exclusively
owns the state and persistent proxy connection, reconnects through the
Stage 2 supervisor, emits structured JSON logs, and exposes versioned
health and control methods on a private Unix socket. `sphere-meshctl`
is a separate client with no key access. A hardened systemd user unit
provides startup and restart supervision. The service and all read-only
methods have been validated against the physical Sphere.
