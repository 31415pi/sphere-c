# Sphere C desktop GUI source audit

## Scope and evidence

This is a read-only source audit for a future Linux desktop GUI. It
compares the acquired Olight Sphere C mini-app with the current local
mesh daemon. No device commands were sent for this audit.

The primary UI source is the saved mini-app bundle:

```text
old/phone_external/files/Godzilla/MiniApp/
  9E6DD95017EDCABEA0673DC3A9AFAC12/
  Resources/0.5.9_RsPYtSG2yNGWYhKJ/
```

`app-config.json` identifies one page, `pages/home/index`, uses custom
navigation, and disables page scrolling. The page bundle is
`b1y/eo8LLi.js`; its English labels are in
`Common/I18N/d0dpga0h_1786209994536_en_US.json`. The saved code files
are minified and include framework/runtime code, so screen structure
is inferred from page registration, app-specific strings, image names,
and the individual page handlers. `docs/research/mesh/host_device_animation_audit.md`
records a deeper protocol-focused reading of this same bundle.

Other useful visual assets in the bundle's `images/` directory are
`bgImg.png`, `device_on.png`, `device_off.png`, `light_icon.png`,
`dimmer_icon.png`, `dimmer_icon_active.png`, `music_icon.png`,
`music_icon_active.png`, `timer_icon.png`, `countdown_icon.png`,
`setting_icon.png`, `setting_icon_active.png`, `game_icon.png`, and
`playMusic.gif`. `b1y/esdEim.css` supplies a compact reset and a
mobile-oriented PingFang/Source Han Sans/Roboto font stack. App config
sets a white navigation background. These assets are good references
for icon meaning and product identity, but the archived evidence does
not preserve a rendered screenshot or enough styling source to claim
pixel-level layout, exact colors, or responsive behavior.

## Screens and navigation evidenced by the source

The app appears to present one device-control panel rather than a
multi-page desktop-style hierarchy. Its visible affordances divide
into several control modes, with a timer/countdown flow and a settings
entry. The I18N file also contains generic Thingclips SDK terms such as
cloud schedules and games; their presence alone does not prove that
every such feature is enabled in the Sphere C panel.

| Area | Source evidence | Meaning for a desktop GUI |
|---|---|---|
| Device/home panel | Only configured page is `pages/home/index`; product label is Sphere C; `device_on.png` and `device_off.png` exist. | A single selected-device dashboard is the strongest source-backed starting point. Show connection and power state distinctly. |
| Light/color | `light_icon.png`; I18N labels `White`, `Color`, `Scene`, and mode names; bundle maps `switch_led`, `work_mode`, `bright_value`, `temp_value`, and `colour_data`. | Provide basic power and standard lighting controls. Do not imply all generic modes have known Sphere-specific behavior. |
| Dimming | `dimmer_icon.png` plus active variant; `dp_bright_value` label is Brightness; schema DP 3 is `bright_value`. | Brightness belongs in the primary control surface. Daemon has standard Lightness but lacks the mini-app's DP 3 write path. |
| Built-in effects | Effect names in I18N and the saved panel handler; visible set is recorded in the mesh audit. | Offer a labeled picker/grid and indicate the current DP 102 effect. Keep six panel-hidden schema values unavailable. |
| Music | Music normal/active icons and `playMusic.gif`; mini-app starts a microphone-driven transform and writes `music_data`. | Treat as a later feature: local daemon has no microphone or DP 8 streaming API. |
| Countdown | `timer_icon.png`, `countdown_icon.png`; labels include Countdown to OFF/ON, 1h–6h presets, cancel, and seconds. | A countdown dialog with presets and custom duration maps naturally to the daemon's DP 101 methods, but count-up/turn-on semantics are not supported locally. |
| Settings | Normal/active settings icons; I18N includes Setting, Restore factory light modes, and product/device messages. | Settings can initially contain connection/device info. Factory restore and generic schedules must not be represented as available daemon actions. |

The most firmly established navigation is icon-based switching between
the light/dimming/music/timer/settings affordances. The source does not
provide a desktop navigation precedent. A desktop app can preserve the
mode grouping while making connection status and feedback persistently
visible.

## Controls and state model

### Mini-app state

The app bundle uses Redux-style state slices named `systemInfo`,
`dpInfor`, and `countdown`. The countdown slice stores
`countdownData` and `currentColorData`. The app also has a typed DP
schema. For the Sphere-specific fields documented in the saved bundle:

| DP | Name | App role |
|---:|---|---|
| 1 | `switch_led` | power switch |
| 2 | `work_mode` | white, colour, scene, music mode |
| 3 | `bright_value` | brightness |
| 5 | `colour_data` | HSV color |
| 6 | `scene_data` | generic scene data; no Sphere panel call site found |
| 8 | `music_data` | write-only music rhythm stream |
| 101 | `countdown_2` | countdown used by saved panel |
| 102 | `colour_mode` | built-in effect enum |
| 103 | `battery_percentage` | read-only battery |

The page logic handles device on/off and effects. For effect selection,
the documented app path turns the device on if needed, sets work mode
to `colour`, and then writes the selected DP 102 enum. The saved UI
catalog exposes 26 entries: Flame; Cotton Candy; Aurora; Fireworks;
Fountain; Spiral; Color Wheel; Red Flash; Confetti; Butterfly;
Fireball; Gradient; Carousel; Magic Ball; Lilac; Sunset; Vibrant; Red
Light; Blue sky; Harvest; Willow Wisp; Zombie; Phantom Purple;
Christmas; Candy Cane; Snowfall. The schema has 32 values; the panel
omits Firefly, Floating, Autumn, New Year, one Christmas variant, and
one Snowfall variant.

### Daemon-owned state

The desktop client should treat the daemon as the source of Bluetooth
and mesh truth. `docs/guides/local-service.md` and
`src/sphere_mesh/daemon.py` establish that `sphere-meshd` owns the
locked state file and persistent proxy connection. The UI should call
its private Unix socket and should not open Bluetooth or parse mesh
keys itself. API health reports `connecting`/`connected`, generation,
active device address, and last failure. It does not expose keys.

The daemon's persistent state (`src/sphere_mesh/state.py`) includes
mesh sequence/TID cursors, one active device and device records,
composition, and credentials. This is service state, not UI state; the
GUI need not mirror secrets or sequence cursors. The API is request /
response JSON, version 1, one newline-terminated message per socket
connection, maximum 64 KiB. Supported calls are listed in the table
below.

## Capability gap table

| UI capability from saved app | Local daemon/API status | Evidence / gap |
|---|---|---|
| Connection/online feedback | Supported | `health` returns connected state, generation, device address, and last failure. |
| Power on/off/read/toggle | Supported | `control` over Generic OnOff; state includes present, target, remaining time, and effective state. |
| Standard brightness | Supported at Mesh Lightness level | `lighting` accepts `lightness` read/set, using the standard Light Lightness Server. It is not mapped to app DP 3's `bright_value` semantics. |
| Color temperature | Supported | `lighting` accepts `temperature` read/set using CTL Temperature. |
| HSL color | Supported | `lighting` accepts `hsl` read/set using the Sphere's standard HSL model. The exact app DP 5 color format is not the daemon API. |
| xy color | Not available on Sphere composition | `lighting xyl` exists in controller/parser, but the captured Sphere composition lacks an xyL Server; daemon returns `unsupported`. |
| Read vendor datapoints | Supported, generic | `vendor.read` reads selected IDs and returns typed values, raw bytes, and missing IDs. It is read-only. |
| Effect catalog/current effect | Supported | `effect.list` returns 32 schema records and visibility metadata; `effect.get` reads DPs 2 and 102. |
| Select current effect while preserving mode | Supported | `effect.set` / `effect.select` writes DP 102 only. A GUI should use `effect.activate` for the app-like mode-then-effect behavior. |
| Activate effect as app does | Supported | `effect.activate` writes DP 2=`colour`, then DP 102. Both are separate acknowledged transactions; empty acknowledgments may not independently confirm appearance. |
| Countdown to OFF | Supported | `countdown.get/set/cancel` use DP 101, 0–43,200 seconds. Zero has live validation; positive delays and actual delayed power-off remain untested. |
| Countdown to ON | Not supported | Saved panel labels include this behavior, but local API exposes only DP 101 countdown/off semantics; no turn-on scheduling method. |
| App DP 3 brightness | Not supported as vendor-DP control | No guarded vendor write for DP 3; only standard Lightness Set exists. The mapping/equivalence between standard lightness and DP 3 is not established by this code. |
| Direct app color-data DP 5 | Not supported | No vendor write method for DP 5. HSL Set is available via SIG model instead. |
| Work-mode picker (white/colour/scene/music) | Partially supported | `effect.activate` sets colour mode as a side effect. No general `mode.set`; white behavior overlaps standard Mesh controls but scene/music do not. |
| Scene DP 6 / arbitrary custom scene editing | Not supported | Schema exists but panel call site for DP 6 was not found; daemon has no DP 6 codec or write endpoint. |
| Music/microphone rhythm | Not supported | App requires microphone capture and continuous DP 8 writes; local API has neither streaming nor microphone handling. |
| Battery percentage | Not supported | Schema DP 103 is read-only but no explicit API/UI model is implemented. Generic `vendor.read` could query it experimentally; that is not a normalized, validated battery feature. |
| Schedules / cloud timing / sleep / wake / factory reset | Not supported by daemon | Labels in generic I18N are not sufficient proof of Sphere panel support. No corresponding safe API methods exist. |

## Desktop GUI implications

1. Connect to the daemon socket and render its `health` result as the
   persistent connection indicator. Disable commands while disconnected
   and display the service's error message when a request fails.
2. The first useful view can combine on/off, standard brightness,
   temperature, HSL controls, a built-in effect picker, and countdown
   to OFF. Keep the currently selected effect and current power state
   separate; selecting an effect alone does not guarantee colour mode.
3. Use daemon catalog metadata for the effect picker. Grouping by
   visual style may be a desktop design choice, but no category taxonomy
   is evidenced in the saved picker. Avoid presenting the six omitted
   values as selectable.
4. Show that countdown writes may be acknowledged without a reported
   value, and that positive delayed-off behavior is not yet validated.
5. Mark music, custom scenes, battery, turn-on countdown, and app-only
   schedules as future work until the daemon adds and validates those
   paths. Do not infer them from SDK-wide translations or schema fields.

## Evidence paths

- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/app-config.json`
- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js`
- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/esdEim.css`
- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/images/`
- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Common/I18N/d0dpga0h_1786209994536_en_US.json`
- `docs/research/mesh/host_device_animation_audit.md`
- `docs/guides/local-service.md`
- `src/sphere_mesh/api.py`
- `src/sphere_mesh/daemon.py`
- `src/sphere_mesh/vendor.py`
- `src/sphere_mesh/lighting.py`
- `src/sphere_mesh/control.py`
- `src/sphere_mesh/state.py`
