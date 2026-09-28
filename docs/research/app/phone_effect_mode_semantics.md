# Phone effect and color-mode semantics

## Findings

The picker entry labeled Color Wheel is `colour_7` (DP 102). It is
special in the phone UI: selecting it follows the same color-mode
selection path as other color presets, then opens the interactive
color-wheel dialog. It is not an animated effect like Aurora.

For an individual device, the handler first turns on DP 1 if needed,
then invokes the typed setter for DP 2=`colour`, then DP 102=`colour_7`.
The two mode writes are separate sequential calls. Selecting Color
Wheel additionally opens its dialog; subsequent wheel movement writes
DP 5 (`colour_data`) with the new hue and saturation while retaining
the current value. Group control publishes DPs 2 and 102 together.

Entering color mode from the Light tab is explicit in the phone's
picker handler: `work_mode.set("colour")` occurs before
`colour_mode.set(e.code)`. The wheel's DP 5 update happens later,
after the dialog reports a selection. Thus DP 2 is not implicitly
assumed by the phone before color data is sent; it is explicitly
written as part of selecting the color preset. In group mode, DPs 2
and 102 are passed together and wheel changes publish DP 5.

The Light-tab brightness slider branches on `work_mode`. In white
mode it writes DP 3=`bright_value` as `10 * sliderValue`. Otherwise,
it writes DP 5=`colour_data`, preserving current hue and saturation
and setting HSV value to `10 * sliderValue`. This handler does not
write DP 102 and does not reselect the running effect. Therefore a
brightness adjustment while Red Flash (`colour_8`) is selected sends
color data only; any visual impact on the firmware effect is device
behavior not established by this code path.

## Call-flow evidence

Source is the saved mini-app bundle:

`old/phone_external/files/Godzilla/MiniApp/`
`9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/`
`0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js`

- ScenarioController defines `colorWheel` as `code:"colour_7"`.
  Its picker callback `I(e)` turns on DP 1 if needed. For
  `whiteLight`, it writes work mode `white`; otherwise the individual
  path calls `b.work_mode.set("colour")`, followed by
  `b.colour_mode.set(e.code)`. For `colorWheel`, it then opens the
  dialog (`h(!0)`).
- The same callback's group branch publishes `{2:"colour",102:e.code}`.
- The dialog wheel callback `R(e)` writes
  `b.colour_data.set({hue:e.h,saturation:e.s,value:t||1})` for an
  individual device. The group branch formats the same HSV fields
  and publishes DP 5.
- The brightness component's handler checks whether work mode is
  `white`. White mode writes `bright_value.set(10*e)`. The color branch
  reads current hue and saturation, then calls
  `colour_data.set({hue:i,saturation:u,value:10*e})`; it contains no
  color-mode/effect setter.
- The product schema maps DP 2 to `work_mode`, DP 3 to `bright_value`,
  DP 5 to `colour_data`, and DP 102 to `colour_mode`; the picker maps
  Red Flash to `colour_8` and Color Wheel to `colour_7`.

The minified bundle is one physical line. Existing source interpretation
and protocol details are documented in
[`sphere_color_mapping_audit.md`](sphere_color_mapping_audit.md),
[`host_device_animation_audit.md`](../../mesh/host_device_animation_audit.md),
and [`vendor_dp_inventory.md`](../../mesh/vendor_dp_inventory.md).

## Implementation recommendation

Mirror the individual-device phone order when entering a firmware
color preset: ensure power on, write DP 2=`colour`, then write the
chosen DP 102 value. For Color Wheel, expose the interactive color
controls and send DP 5 updates after mode selection. While an effect
is active, treat the phone brightness control as an HSV value update
through DP 5; do not issue a DP 102 reselect based on this evidence.
The source establishes write intent and ordering, but does not prove
how changing DP 5 affects a currently animated effect such as Red
Flash.
