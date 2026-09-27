# Sphere C Tuya/Olight datapoint inventory

## Scope

This is an inventory of datapoints mentioned by the repository's saved
Sphere C research as of 2026-09-27. It reconciles four different kinds
of evidence that must not be treated as interchangeable:

- **LIVE**: decoded, authenticated traffic from the locally owned
  Sphere C;
- **MINI**: the product-specific Sphere C mini-app's embedded schema or
  panel call sites;
- **APK**: static code in the Olight 3.2.5 APK, usually generic
  Thingclips/Tuya transport or conversion code; and
- **SPEC**: an inference, an older public-profile summary, or a generic
  Tuya mapping not confirmed for this product.

"Read evidence" means evidence that a value was actually reported, or
weaker evidence that a schema permits reading. "Write evidence" means a
product call site or observed transaction, not merely `mode: "rw"`.
DP 102 now has a live acknowledged write, separate readback, and
restoration. Other Sphere DP writes are not live-confirmed.

The product schema is embedded in the one-line mini-app bundle [S1]. It
contains 17 entries. DP 4 is retained below as a separately identified
generic/public-profile candidate because older research names it, but
it is absent from the saved Sphere C schema.

## Wire grammar and transport boundary

For vendor server `0x07d0:0x0004`, the APK and the live read establish
the ordinary DATA record grammar [S2] [S3]:

```text
dp_id:u8 || dp_type:u8 || value_encoding
```

The relevant Tuya type bytes are raw `00`, bool `01`, value `02`,
string `03`, enum `04`, and bitmap `05`. Raw and string carry a one-byte
length; bool and enum carry one byte; value is a four-byte big-endian
integer. The plain DATA body starts with command type `01` or `05` and
then records run to the end; it has no record count.

A schema DP is not necessarily carried as a vendor record. The APK maps
power, brightness, temperature, and color through SIG Mesh Generic
OnOff, Light Lightness, Light CTL Temperature, and Light HSL
respectively [S4]. Vendor-wire types below therefore say either
"confirmed", "APK-derived", or "not established" rather than assuming
that the mini-app property's Tuya type appeared on the vendor model.

## Evidence table

| ID | Schema name | Schema / vendor wire type | Read evidence | Write evidence | Known range or encoding | Semantics | Confidence / class | Source |
|---:|---|---|---|---|---|---|---|---|
| 1 | `switch_led` | bool; Tuya `01` in schema, but APK maps ordinary control to SIG Generic OnOff | **MINI:** `rw`; panel consumes current state. No live vendor record. Standard OnOff readback is live-verified elsewhere. | **MINI:** device panel calls `switch_led.set(true)` when an effect/music selection needs power. **APK:** ordinary mesh control is SIG, not a proven vendor write. | `false` / `true`; SIG state `0` / `1` | Main lamp power | High product semantics; high SIG mapping; vendor encoding speculative | [S1] [S4] [S5] |
| 2 | `work_mode` | enum; vendor `04` is **APK+MINI-derived** | **LIVE-negative:** focused and combined vendor reads requested DP 2, but all replies omitted it. **MINI:** `rw`. | **MINI:** effect UI writes `colour`; music UI writes `music`. **APK:** enum writer gives `02 04 VV`. No live write. | `00=white`, `01=colour`, `02=scene`, `03=music` | Selects the lighting operating mode | Very high schema/encoding; medium firmware readability; no live write | [S1] [S2] [S3] [S5] |
| 3 | `bright_value` | value; Tuya `02` in schema, but APK maps it to SIG Light Lightness | **MINI:** `rw`; panel consumes it. No live vendor record. | **MINI:** brightness slider writes the value. **APK:** converted to 16-bit SIG Lightness, with nonzero output clamped to at least 1. | Decimal `10..1000`, step 1; linear mapping to SIG `0..65535` | White-light brightness; color-mode value is carried inside DP 5 | Very high product range and SIG mapping; vendor transport not established | [S1] [S4] |
| 4 | `temp_value` (generic name) | value in generic Tuya model; APK maps to SIG CTL Temperature | No product-schema or live vendor evidence. Older compatibility research says the public profile shares generic DP 4. | **APK-only:** generic setter exists and sends SIG CTL Temperature with delta-UV zero; no Sphere panel call site in the saved bundle. | Saved research does not preserve a Sphere Tuya range. APK maps its supplied schema interval to `800..20000 K`. | Color temperature | **SPEC/generic**, not a saved Sphere C schema entry | [S4] [S6] |
| 5 | `colour_data` | string (`03`, max 255) at schema boundary; APK converts it to SIG HSL rather than an ordinary vendor string | **MINI:** `rw`; panel consumes parsed HSV. No live vendor record. | **MINI:** color wheel writes structured HSV through the typed DP setter. **APK:** converts to SIG HSL Set. | Exactly 12 hex digits in normal form: 4 each for hue `0..360`, saturation `0..1000`, value `0..1000` | Static color and color-mode brightness | Very high schema, UI, and SIG conversion; vendor-string use unproven | [S1] [S4] [S5] |
| 6 | `scene_data` | string (`03`, max 255) in schema; generic APK converter emits vendor raw (`00`) compact scene data | **MINI:** `rw` only. No Sphere UI read/call site and no live record. | **APK-only/generic:** scene compressor exists. The saved Sphere panel does not use DP 6 for its preset picker. | Normal form: 1-byte header plus 13-byte units; compact form has a 2-byte header and 4 bytes per unit. Exact packed-field semantics remain unresolved. | Generic Tuya SoC scene, not Sphere built-in-effect selection | High schema presence; medium generic codec; low product use | [S1] [S5] |
| 7 | `countdown` | value; vendor `02` **live-confirmed** | **LIVE:** returned `07 02 00000000` (0) even when other IDs were requested. | **MINI:** `rw` capability only; no clear Sphere panel setter call site. No live write. | `0..43200` seconds, step 1; live value was 4-byte big-endian zero | Countdown to turn the light off | Very high type/read/range; write behavior unconfirmed | [S1] [S3] |
| 8 | `music_data` | string (`03`, max 255), write-only in schema; APK converts it to 4-byte vendor raw (`00`) | No read is permitted by the schema; no live record. | **MINI:** music UI continuously calls `music_data.set(...)`. **APK:** vendor body derives as `08 00 04 XX XX XX XX`. No live write. | Normal form fields are mode (1 hex digit), then 4 hex digits each for hue, saturation, value, brightness, and temperature. Mesh compaction retains mode/hue/saturation/value in 4 bytes; the panel fixes saturation to 1000. | Phone-microphone-driven music rhythm | Very high product write path; high APK-derived wire encoding; no live write | [S1] [S5] |
| 33 | `power_memory` | raw (`00`), max 255 | **MINI:** `rw` only; no product call site or live record. | Schema capability only; no saved write call or transaction. | Opaque, `0..255` bytes | Behavior after power restoration, based on schema label only | High schema presence; low encoding/semantics | [S1] |
| 37 | `rtc_timer` | raw (`00`), max 128 | **MINI:** `rw` only; no live record. | Schema capability only; no saved write call or transaction. | Opaque, `0..128` bytes | Local/RTC timer configuration, based on schema label only | High schema presence; low payload semantics | [S1] |
| 38 | `timer_report` | raw (`00`), max 128 | **MINI:** `rw` only; no live record. | Schema capability only; no saved write call or transaction. | Opaque, `0..128` bytes | Timer-state report, based on schema label only | High schema presence; low access direction/payload semantics | [S1] |
| 101 | `countdown_2` | value; vendor `02` **live-confirmed** | **LIVE:** returned `65 02 00000000` (0). **MINI:** panel consumes it. | **MINI:** countdown dialog calls `countdown_2.set(seconds)`. No live write. | `0..43200` seconds, step 1; 4-byte big-endian integer on live wire | Countdown to toggle/off; this is the countdown DP actually used by the saved panel | Very high type/read/range and product write call; no live write | [S1] [S3] |
| 102 | `colour_mode` | enum; vendor `04` **live-confirmed** | **LIVE:** returned `66 04 07`, meaning `colour_7`. | **LIVE:** `07`, change to `02`, and restoration to `07` were acknowledged with exact DATA echoes and independently read back. **MINI:** effect picker writes this after DP 2=`colour`. | `00=flame`; `01..1f=colour_1..colour_31`. The visible picker omits some schema values. | Selects a built-in firmware effect; it is not DP 6 scene data and not an LED index | Very high live wire encoding, state change, restoration, and product semantics; visible appearance unobserved | [S1] [S2] [S3] [S5] [S8] |
| 103 | `battery_percentage` | value; vendor `02` **live-confirmed** | **LIVE:** returned `67 02 00000064` (100). **MINI:** read-only and displayed by panel. | Schema is `ro`; no write evidence. | `0..100%`, step 1; live four-byte big-endian value | Battery state of charge | Very high live and schema evidence | [S1] [S3] |
| 104 | `sleep_switch` | bool; vendor `01` **live-confirmed** | **LIVE:** returned `68 01 00` (`false`). | **MINI:** `rw` capability only; no saved panel setter call site. No live write. | `false` / `true` | Sleep-mode switch; the schema's untranslated display name is `0min`, so duration behavior is unknown | Very high live type/read; medium broad semantics; write behavior unknown | [S1] [S3] |
| 105 | `game_flag` | bool (`01`) if carried as an ordinary vendor DP; not live-confirmed | **MINI:** `rw` only; no panel read call site or live record. | Schema capability only; no saved setter or transaction. | `false` / `true` | “Whack-a-mole”/game flag (translated schema label); physical behavior unknown | High schema presence; low behavior and transport | [S1] |
| 106 | `factory_mode_reset` | bool (`01`) if carried as an ordinary vendor DP; not live-confirmed | Schema is write-only; no read evidence. | **MINI:** Restore Factory action calls `factory_mode_reset.set(true)` and then queries all schema IDs except write-only DP 8. No live write. | Panel writes `true`; reset scope and response are not captured | Restore factory lighting mode. It must not be conflated with Mesh Node Reset or full provisioning reset. | Very high product call site; low device-side consequences/wire observation | [S1] |
| 107 | `water_mode` | bool (`01`) if carried as an ordinary vendor DP; not live-confirmed | **MINI:** `rw` only; no panel read call site or live record. | Schema capability only; no saved setter or transaction. | `false` / `true` | Water mode, based on translated schema label only | High schema presence; low behavior and transport | [S1] |

## Live snapshot boundary

The live read request asked for DP 2, DP 102, or both. In every case
the firmware first returned an empty DATA acknowledgment and then the
same five-record snapshot [S3]:

```text
cd d0 07 01
cd d0 07 01
  07 02 00000000
  65 02 00000000
  66 04 07
  67 02 00000064
  68 01 00
```

Thus DPs 7, 101, 102, 103, and 104 are live-confirmed vendor records.
The result does not prove that the request ID list filters responses;
in fact it strongly suggests that the list triggers a status refresh.
DP 2's omission is evidence of omission, not proof that DP 2 is
write-only or unsupported.

## Reconciled conflicts and exclusions

1. Older compatibility research called DP 7 “music”, following the
   generic Tuya mapping. The product schema and live wire establish
   that Sphere C DP 7 is `countdown`; music is product DP 8 [S1] [S3]
   [S6].
2. Older compatibility research called DP 102 a scene DP. The saved
   product schema and panel logic establish `colour_mode`, the built-in
   effect selector. Generic scene data remains DP 6 [S1] [S5].
3. DP 4 is not in the saved product schema. It remains in this document
   only because the older public-profile/generic mapping identified it;
   it must not be advertised as locally confirmed Sphere capability.
4. Parameter prefix `86 02 <group-address>` and response
   `86 01 00` are an APK-derived conditional vendor group-association
   command, not a typed DP record. The current artifacts do not prove
   that Sphere C executes that branch [S7].
5. DP ID 0 appears in APK parsing as an all-DP-style query convention;
   it is not a Sphere schema datapoint [S2].
6. Nothing in the schema or captures identifies a pixel, LED index, or
   segment-buffer DP. DP 102 selects firmware effects only [S5].

## Source index

- **[S1] Product mini-app schema and panel:**
  [`old/phone_external/.../b1y/eo8LLi.js`](../../../old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js),
  line 1. Product PID `d0dpga0h`; the complete schema begins at
  `mP=[{...}]`.
- **[S2] APK codec audit:**
  [`vendor_dp_response_codec.md`](vendor_dp_response_codec.md).
- **[S3] Authenticated device read:**
  [`vendor_dp_live_query.md`](vendor_dp_live_query.md).
- **[S4] SIG model mapping:**
  [`step4_sphere_model_map.md`](step4_sphere_model_map.md).
- **[S5] Product animation and call-site audit:**
  [`host_device_animation_audit.md`](host_device_animation_audit.md).
- **[S6] Older generic/public-profile summary:**
  [`../../analysis/compatibility.md`](../../analysis/compatibility.md).
- **[S7] Conditional non-DP vendor group command:**
  [`post_provision_commit_diff.md`](post_provision_commit_diff.md).
- **[S8] Authenticated DP 102 write:**
  [`effect_write_experiment_plan.md`](effect_write_experiment_plan.md).

The APK-derived findings ultimately reproduce from
`old/apk/olight-3.2.5.apk` (SHA-256
`1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9`).
The product bundle SHA-256 is
`024c0b833914dfd52102f4d96568ddda2be3223cf5a98b5b2c402eb18f1d8bf5`.
