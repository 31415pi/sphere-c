# Sphere C desktop app

The optional `sphere-mesh-gui` application is a graphical client for
the local Sphere service. It never opens Bluetooth or reads the Mesh
state file. `sphere-meshd` remains the sole owner of the radio session,
keys, sequence cursor, and command serialization.

The [project report](../../REPORT.md) explains the investigation;
the [Bluetooth reference](../../SPHERE-C-BT.md) documents the
protocol and evidence behind these controls.

## Install and launch

Install the project with its optional Qt dependency:

```console
./.venv/bin/pip install -e '.[gui]'
```

Start the daemon, then launch the app:

```console
./.venv/bin/sphere-meshd
./.venv/bin/sphere-mesh-gui
```

The app uses `state/sphere.sock` when launched from, or installed
editable from, a development checkout containing
`state/sphere_state.json`. Otherwise it chooses the systemd user-service
path under `XDG_RUNTIME_DIR` even while the daemon is still starting, so
reconnection remains pointed at the right socket.
Override either choice explicitly when needed:

```console
sphere-mesh-gui --socket /path/to/control.sock
```

The `SPHERE_MESH_SOCKET` environment variable provides the same
override for desktop launchers or development sessions.

## Original Sphere compatibility

This application and its provisioning workflow have only been tested
with Sphere C. Standard power, Lightness, and HSL controls may be
reusable if an original magnetic-charge Sphere advertises the matching
Bluetooth Mesh services and composition models. The built-in effect
catalog, vendor datapoints, and recovery procedure are Sphere C-specific
until tested otherwise.

Do not factory-reset or provision an original Sphere merely to try the
GUI. Begin with the passive checks in the
[original-Sphere compatibility assessment](../research/compatibility/original-sphere.md),
using separate state and that device's own keys if testing progresses
to authenticated access.

## Views

The **Light** view provides:

- acknowledged power control;
- a shared Brightness slider, also visible above Effects, with a
  logarithmic response from the device's roughly 3% minimum active
  level to full brightness;
- hue and saturation controls using the Sphere's Mesh HSL
  transport; and
- battery percentage from read-only vendor DP 103 when the Sphere
  returns a decodable value.

Slider changes are sent only after release. The UI then accepts the
state returned by the Sphere rather than assuming the requested value
persisted.

Click **Color** to select the device's static Color Wheel mode and apply
the chosen hue, saturation, and Brightness. Color Wheel is an internal
mode rather than an animation, so it is intentionally omitted from the
Effects view.

After this app selects an effect or static Color, changing Brightness
sends an HSV value update through Mesh HSL. It preserves the current
hue and saturation and does not reselect the effect. Outside a color
mode selected in this session, Brightness uses Mesh Lightness. The
slider's percentage represents its logarithmic GUI position, not a
linear percentage of the Mesh value. Power remains a separate control;
Brightness never sends zero.

The Sphere does not reliably report its current work mode. The daemon
can restore the active effect when it has a direct DP 2 read or a mode
acknowledged during the current Bluetooth connection. Restarting or
reconnecting the daemon discards that in-memory knowledge. When DP 2
is omitted after restart, the selected DP 102 preset remains readable
but the daemon cannot tell whether it is active; the GUI currently
falls back to the ordinary **Light on** label. This is a known display
limitation, not evidence that the device left its effect. The phone
source establishes the brightness write pattern, though the firmware's
visual response to an HSL update during a flash effect is not yet
proven.

The 2026-10-04 authenticated read-only wake check again omitted
DP 2. See [the recorded values][wake-check]; readable DP 102
still did not establish active work mode.

[wake-check]: ../../SPHERE-C-BT.md#fresh-read-only-wake-check

The illustrated Sphere uses a lighter visual brightness range than
the device command so dark settings remain easy to see on screen.
Its hue and saturation still follow the selected color.

The Sphere phone panel presents HSV controls and converts between HSV
and the Mesh HSL fields. The desktop app now performs the same
conversion in both directions and renders its preview from HSV. It does
not treat HSV brightness as mathematical HSL lightness.

The **Effects** view loads the catalog from the daemon and presents the
25 built-in presets from the saved phone panel. Aurora and Red
Flash have observed motion; the other 23 grid entries remain
unclassified. User-recorded captures will establish Patterns and
Animations and supply previews after media licensing is settled.
Selecting a tile uses `effect.activate`: colour mode first, then
the chosen effect.
The client
never repeats the whole activation automatically. The daemon waits for
the mode transition to settle and may repeat only the idempotent effect
selection once if that second write receives no response. The client
always performs an independent `effect.get` afterward, including after
a partial failure.

The **Settings** view shows the socket, connection generation, node
address, model-derived capabilities, battery, and last connection
failure. Copy Diagnostics places a redacted JSON summary on the
clipboard; it contains no keys or sequence state.

## Connection and error behavior

The app polls daemon status and refreshes device state whenever the
connection generation changes or the window regains focus. Controls
remain disabled while a command is pending or Bluetooth is unavailable.

Errors remain visible until dismissed. If effect activation fails after
the mode write, the details identify the completed and failed steps and
warn that the device state may be uncertain. Use Refresh State before
deciding whether to retry.

Battery refresh is opportunistic because some vendor replies are
special statuses rather than datapoint records. The service ignores
those while looking for an ordinary response; if none arrives, the app
leaves battery hidden without blocking the functional controls.

The authenticated composition advertises CTL Temperature, and the
lower-level CLI retains that experimental command. It is intentionally
absent from the desktop app because live changes produced no obvious
visible effect and the saved phone panel does not establish it as a
Sphere control.

Countdown and music controls are intentionally absent. Positive
countdown behavior has not been useful enough to promote, and the phone
app's microphone-driven music mode was explicitly deferred.

## Desktop integration

Python packaging installs:

- the `sphere-mesh-gui` console program;
- `sphere-mesh-gui.desktop`; and
- an original scalable `sphere-mesh-gui` icon.

An editable install inside `.venv` keeps those shared files inside the
virtual environment; most desktop shells do not search that location.
The command above therefore installs a working GUI program, but does
not by itself add an application-menu entry. A distribution package or
explicit per-user installation must place the launcher and icon in the
corresponding `XDG_DATA_HOME` application and icon directories, with
`sphere-mesh-gui` available on the launcher's `PATH`.

The application follows the desktop light/dark palette, supports a
resizable narrow layout, and exposes keyboard focus and accessible
labels for controls and effect cards.

No phone-app artwork or media is packaged. The desktop icon and QML
Sphere illustration are original project assets distributed under the
repository's MIT license. Their provenance and credit are recorded in
`packaging/desktop/ASSETS.md`; Qt uses the user's system fonts.
