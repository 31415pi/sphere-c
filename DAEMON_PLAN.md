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

## 3. Basic control surface

Expose read, on, off, and toggle operations. Commands must use
acknowledged messages, report the Sphere's returned state, and fail
clearly when the proxy or node is unavailable.

## 4. Standard lighting models

Implement brightness, color temperature, HSL, and xyL operations
against the standard models present in the Sphere's Composition Data.
Read status back from the device rather than assuming a Set succeeded.

## 5. Stable local API and service

Add a versioned local API, a command-line client, lifecycle and health
reporting, structured logs, and service supervision. Keep mesh keys
and mutable protocol state private on disk.

## 6. Olight-specific functionality

Capture and decode the two Olight vendor models to support patterns,
animations, saved settings, and other behavior not represented by the
standard Bluetooth Mesh lighting models.

## Current status

Stage 1 is complete. The local adapter, GATT retry/cleanup behavior,
and live Mesh Proxy session now live in `sphere_transport.py`. The
provisioning and read-only control harness uses that shared module and
has been verified against the physical Sphere without changing its
light state.
