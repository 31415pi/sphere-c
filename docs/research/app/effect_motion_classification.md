# Sphere C effect motion classification

## Finding

The saved Sphere C mini-app and DP schema do not encode a static versus
animated distinction for the GUI picker. Each picker entry has the same
`type:"colour"` field, a DP 102 enum code, and a still PNG thumbnail.
The schema lists DP 102 values but no motion flag, preview type, or
timing metadata. The APK evidence examined does not add product-specific
motion metadata. Therefore most entries remain unknown from artifacts.

Two classifications are independently supported by live observation.
Aurora (`colour_2`) visibly animated after selection. Red Flash
(`colour_8`) visibly flashed with recurring dark periods during GUI
brightness testing. Color Wheel (`colour_7`) is excluded from the
requested table; prior live testing found that it retained the previous
static red color. This establishes the wheel's interaction semantics,
not a general static/effect taxonomy.

## GUI-visible entries (Color Wheel excluded)

| DP 102 slug | Picker label | Motion classification | Evidence |
|---|---|---|---|
| `flame` | Fireplace / Flame | Unknown | Thumbnail only |
| `colour_1` | Cotton Candy | Unknown | Thumbnail only |
| `colour_2` | Aurora | Animated | Live observation after DP 2=`colour`, DP 102=`02`; see `host_device_animation_audit.md` |
| `colour_3` | Fireworks | Unknown | Thumbnail only |
| `colour_4` | Fountain | Unknown | Thumbnail only |
| `colour_6` | Spiral | Unknown | Thumbnail only |
| `colour_8` | Red Flash | Animated | Live observation of repeated flashes and dark periods during brightness testing |
| `colour_9` | Confetti / Cheerleaders | Unknown | Thumbnail only |
| `colour_10` | Butterfly | Unknown | Thumbnail only |
| `colour_11` | Fireball / Small Rocket | Unknown | Thumbnail only |
| `colour_12` | Gradient | Unknown | Thumbnail only |
| `colour_13` | Carousel | Unknown | Thumbnail only |
| `colour_15` | Magic Ball | Unknown | Thumbnail only |
| `colour_16` | Lilac | Unknown | Thumbnail only |
| `colour_17` | Sunset | Unknown | Thumbnail only |
| `colour_18` | Vibrant | Unknown | Thumbnail only |
| `colour_20` | Red Light | Unknown | Thumbnail only |
| `colour_21` | Blue Sky | Unknown | Thumbnail only |
| `colour_24` | Harvest | Unknown | Thumbnail only |
| `colour_25` | Willow Wisp / Magic Blue Flame | Unknown | Thumbnail only |
| `colour_26` | Zombie / Zombie Green | Unknown | Thumbnail only |
| `colour_28` | Phantom Purple | Unknown | Thumbnail only |
| `colour_29` | Christmas / Celebrate | Unknown | Thumbnail only |
| `colour_30` | Candy Cane | Unknown | Thumbnail only |
| `colour_31` | Snowfall / Snowflake | Unknown | Thumbnail only |

The alternate labels reflect differences between the product schema's
English translations and the picker keys/localized labels. Names are
not used as evidence of motion.

## Artifact inspection

- The picker catalog and callbacks are in
  `old/phone_external/files/Godzilla/MiniApp/`
  `9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/`
  `0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js` (minified, one line).
  The catalog groups entries under Conventional, Nature, Festival,
  and Atmosphere; each has a `code`, `imgUrl`, and `type:"colour"`.
  The handler selects DP 102, with no animation branch or timing.
- Picker thumbnails are individual PNG stills under the same bundle's
  `images/conventional/`, `images/nature/`, `images/festival/`, and
  `images/atmosphere/` directories. Filenames correspond to the picker
  keys; they do not describe motion. The only GIF found in the mini-app
  assets is `images/playMusic.gif`, a music UI indicator, not a preset
  preview.
- The product DP schema in the same JS declares DP 102 as an enum with
  `flame` and `colour_1` through `colour_31`; it has no separate static
  list, motion flags, durations, or preview payload fields.
- `docs/research/mesh/host_device_animation_audit.md` provides the
  schema catalog, visible subset, and live evidence. It records Aurora
  animation and prior static red remaining when Color Wheel was set.
- Existing APK research describes generic SIG Mesh and vendor-DP
  mapping, not an Olight Sphere preset motion catalog. No APK mapping
  found in the inspected research establishes motion per DP 102 value.

## Recommended empirical capture

Capture each of the 25 entries on the same powered Sphere in a stable,
fixed-exposure video. Before each selection, record current DP 2/DP 102
state and set DP 2=`colour`, then select the entry through the app (or
the already validated DP 102 path). Record at least 20–30 seconds after
settling, with a neutral stationary camera and no ambient light changes.
Compare frames or sampled RGB values from fixed points on the sphere;
record temporal changes separately from brightness flicker caused by
camera exposure. Repeat a suspected static preset and an animated
preset as controls, and note any presets requiring a longer cycle.
Preserve the video filename, preset slug, wire value, timestamp, and
whether DP 5/brightness changed during capture. Do not classify from
thumbnail appearance or preset names alone.
