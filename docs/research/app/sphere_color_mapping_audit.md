# Sphere phone panel color mapping audit

## Findings

The saved Sphere C phone panel has no color-temperature/white-temperature
slider or control. Its visible light-mode list contains the white-light
preset and a color-wheel affordance. The white-light selection sets
`work_mode` to `white`; the color wheel writes `colour_data`. The Sphere
mini-app schema contains `bright_value` (DP 3) and `colour_data` (DP 5),
but no `temp_value` DP. The generic Olight APK temperature setter is not
evidence of a Sphere panel feature.

The wheel's UI state is HSV: hue 0–360 degrees, saturation 0–1000, and
value 0–1000. During an interactive wheel change, the handler preserves
the current value (`v: t.v || 1000`) and passes the resulting hue and
saturation to the DP setter. The color-mode brightness slider also
preserves hue and saturation while writing `value = 10 * sliderValue`.

For a single-device panel, the typed setter serializes the three HSV
fields as three zero-padded four-digit hexadecimal numbers, in the order
hue, saturation, value. This is the 12-hex-digit `colour_data` payload;
the group path explicitly calls `Nj.formatter({hue, saturation, value})`.
The product DP is a string at the Tuya schema edge. In the APK's standard
Mesh route, that HSV triple is converted to HSL and sent as a SIG Light
HSL Set: HSL lightness, hue, and saturation are each scaled to 16-bit
fields. Hue and saturation are scaled from their app domains; HSV value
is converted to HSL lightness rather than copied directly. Brightness
is a separate Light Lightness control in white mode; in color mode it is
carried as HSV value inside DP 5.

No product color-wheel handler or saved APK mapping examined here applies
a gamma correction step. The app uses HSV state for the wheel and
converts that state to HSL for the Mesh payload. Do not add a gamma
transform to the documented conversion absent additional evidence.

## Preview behavior

The `ColorWheelDialog` component (`eg`) renders the `UOe` color wheel
with current hue and saturation (`hs:{h:t.h,s:t.s}`), and the wheel
callback sends its new `h` and `s` while preserving `v`. The saved panel
also renders `LightDimmer/Screen/ColorBrightness` (`eS`) below the
brightness slider; its state is the selected color brightness. The
captured source does not establish a separate CSS solid-color preview
swatch or a gamma-adjusted preview pipeline. The wheel itself is the
source-backed color preview; a UI implementation can render a solid
sample from the same HSV values, but that would be a new presentation
choice rather than a reproduced detail.

## Source evidence

- `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js`, line 1,
  app component `ey` (ScenarioController): its `I` handler selects
  `whiteLight` via `work_mode.set("white")`; `colorWheel` opens the
  picker; its `R` handler calls
  `colour_data.set({hue:e.h,saturation:e.s,value:t||1})`. In group mode
  the analogous call uses `Nj.formatter` for DP 5.
- The same bundle's `eg` (`ColorWheelDialog`) passes current hue and
  saturation to `UOe`; `s` retains `t.v || 1e3` while forwarding new
  wheel `h,s`. Component `ew` is the brightness slider: white mode
  writes DP 3 as `10 * e`; color mode writes DP 5 with existing hue and
  saturation and `value:10 * e`.
- The same bundle's `mP` product schema includes `bright_value` id 3
  and `colour_data` id 5, but no `temp_value`. Its product DP 5 typed
  formatter `Nj` formats hue, saturation, and value using four hex
  digits each.
- [`docs/research/mesh/vendor_dp_inventory.md`](../../mesh/vendor_dp_inventory.md),
  evidence rows 3–5, separates Sphere mini-app evidence from generic
  temperature support and records DP 5's 12-digit HSV payload.
- [`docs/research/mesh/step4_sphere_model_map.md`](../../mesh/step4_sphere_model_map.md),
  “Lightness / brightness” and “HSL”, records APK conversion from HSV
  text to 16-bit Mesh Lightness/Hue/Saturation and the independent
  Lightness mapping. The APK-derived temperature route is generic and
  lacks a Sphere panel call site.

The app JavaScript is minified into a single line, so symbol names and
distinctive handler expressions are provided above as navigation aids.
