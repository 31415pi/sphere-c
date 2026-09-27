# Sphere C desktop GUI plan

## Goal

Build a polished Linux desktop client in the general spirit of the
Sphere phone panel: one product-focused dashboard, large direct lighting
controls, an effect picker, compact mode navigation, and clear device
feedback. It should feel related to the phone experience without being
a pixel-for-pixel copy or reusing vendor artwork as application assets.

The source audit behind this plan is
[desktop_gui_source_audit.md](../research/app/desktop_gui_source_audit.md).
The saved mini-app supports the single-dashboard direction, but it does
not preserve enough rendered styling to justify claims about exact
layout or colors.

## Architecture choice

Use PySide6 with QML for the first implementation.

- The repository and daemon are already Python.
- QML suits a responsive, touch-friendly, phone-like control surface.
- `QLocalSocket` can speak the daemon's newline-delimited JSON protocol
  without adding an asyncio bridge or giving the GUI Bluetooth access.
- QObject-backed models can expose connection, lighting, effects, and
  pending/error state cleanly to QML.
- The GUI remains an unprivileged client. It never reads mesh keys,
  opens BlueZ, or writes the daemon state file.

Keep GUI dependencies optional, likely behind a future `gui` project
extra and a `sphere-mesh-gui` console entry point. Package original
icons and artwork created for this project rather than copying the
archived app assets.

## Window and navigation

Start with a narrow, resizable window around 440 by 760 logical pixels.
At wider sizes, allow the controls and effect grid to form two columns.
The persistent shell contains:

- a product header and connection badge;
- a large stylized Sphere preview that reflects the selected color or
  effect category without pretending to simulate the animation;
- a compact error/progress area; and
- bottom navigation for Light, Effects, and Settings.

The phone bundle also evidences music and timer affordances. Music is
deferred at the protocol level, and positive countdown behavior is not
useful enough yet, so neither belongs in the initial navigation. They
can appear later when their daemon features are intentionally promoted.

## Initial views

### Light

Provide power state, brightness, color temperature, and HSL color.

- Read all available standard lighting state on entry and after a
  reconnect.
- Keep a local preview responsive while the user drags a control.
- In the first release, send an acknowledged Set on release rather than
  flooding the mesh during every pointer movement.
- Replace the preview with the returned state, since the firmware may
  requantize HSL values.
- Disable only the controls absent from `status.capabilities`; preserve
  the rest of the view while disconnected.

### Effects

Render the 26 visible catalog entries from `effect.list` as a searchable
grid of labeled cards. Do not hard-code or expose the six hidden values.

- Use `effect.get` for the current selection.
- Use `effect.activate SLUG` for an app-like selection.
- Show a pending state across both writes.
- On success, issue `effect.get` for an independent readback.
- If activation partially fails, show the structured failure details
  and refresh before offering Retry. Do not automatically repeat either
  write.
- Treat Color Wheel as an interactive/static color mode; only Aurora is
  currently visually verified as an animation.

### Settings and diagnostics

Show the daemon socket, connection state, generation, node address,
last failure, and capability summary. Include Copy Diagnostics using a
redacted JSON structure. Never display keys, sequence numbers, or the
state-file contents.

Offer guidance when the daemon is missing or reconnecting, but do not
silently install, enable, or restart a system service. A later explicit
service-management flow can be designed separately.

## Client state and request policy

Use a small state machine: daemon unavailable, reconnecting, connected,
command pending, and recoverable error. Connection state comes from
`status`; device values come from their dedicated read methods.

- Allow one in-flight mesh operation at a time in the GUI even though
  the daemon also serializes requests.
- Coalesce refreshes and discard stale UI results by request generation.
- Poll `status` slowly while connected and more quickly while waiting
  for the daemon or Bluetooth session.
- Refresh power, lighting, effect, and DP 103 battery after connection
  generation changes and when the window regains focus.
- Distinguish transport acknowledgment, value in the response,
  independent readback, and human visual confirmation in diagnostics.
  The normal controls should translate these into plain language.
- Never retry a mutating operation automatically after a timeout.

## Delivery phases

1. **Shell and daemon client**

   Add optional PySide6 packaging, QML resources, socket framing, error
   envelopes, status polling, and mocked API tests.

2. **Read-only dashboard**

   Display connection, capabilities, power, lighting state, effect,
   battery, and diagnostics without any mutating controls.

3. **Standard lighting control**

   Add power, brightness, temperature, and HSL interactions with pending
   state, returned-state reconciliation, and keyboard accessibility.

4. **Effect library**

   Add the 26-entry catalog, activation flow, independent readback, and
   partial-failure recovery UI.

5. **Desktop integration**

   Add an original icon, `.desktop` metadata, notifications for durable
   failures only, theme testing, HiDPI testing, and packaging checks.

6. **Deferred features**

   Reconsider countdown, music streaming, host-side animation, and
   service management only as separate, validated milestones.

## Acceptance criteria for the first useful release

- The app works without access to mesh keys or Bluetooth permissions.
- It remains usable across daemon and Bluetooth reconnects.
- Every mutating control shows pending, success, or actionable failure.
- No timed-out write is retried automatically.
- The effect grid is populated from the daemon and exposes only visible
  effects.
- Capability-gated controls match the saved composition.
- Keyboard navigation, screen-reader labels, light/dark themes, and
  HiDPI layouts are tested.
- Automated tests run against a fake Unix-socket daemon; a final manual
  pass uses the real service and only already-validated operations.
