# Before firmware: report and protocol reference

Prepared 2026-10-04 against commit `c6043fc` and `STATUS.md`.
This plan records the completed blog report, `REPORT.md`, and
technical reference, `SPHERE-C-BT.md`, plus deferred follow-ups.
The owner subsequently requested committing the documentation.

## Execution checkpoint: 2026-10-04

Completed the documentation work after confirming authenticated
read access to the owner's awakened Sphere C. The temporary
daemon used the existing state and was stopped after the checks.
No lighting Set, vendor Write, reset or provisioning was issued.

- [REPORT.md](REPORT.md) now tells the investigation story.
- [SPHERE-C-BT.md](SPHERE-C-BT.md) contains the evidence ledger,
  protocol tables, complete preset catalog and current limits.
- Existing guides have corrected retry/motion wording, and older
  audits have explicit links to superseding results.
- The [OTA preparation document][ota-preparation] now includes an
  exact-device checklist, private manifest and ranked questions.
- All 235 tests passed in 12.91 seconds on this date. Synthetic
  message vectors and all 32 catalog rows were checked against
  current codecs/data; links, tables and manifest syntax checked.
- Current code/dependency review clarified incoming segment ACKs,
  immediate-duplicate suppression versus durable replay defense,
  ignored beacons, and the remaining IV/retransmission gaps.

Remaining work is intentionally explicit: preset video and media
licensing need the owner; hook/export validation needs a prepared
phone and agreed acquisition workflow. Device-visible deferred
experiments remain deferred. Reliability implementation and
desktop acceptance/packaging are separate optional projects, not
prerequisites for these completed documents.

[ota-preparation]: docs/research/hardware/original_sphere_ota_capture_plan.md

## Recommendation and scope

Write both documents now. The repository already explains how
local control became possible and contains enough evidence for a
useful protocol reference. Firmware is needed to investigate
internal behavior, not to explain the working controller.

Prioritize consolidating and checking the existing evidence over
new device experiments. Complete the protocol reference first so
the report can tell the story from a checked factual foundation.
Neither document should wait for video, another device, or an
update offer.

The initial planning pass created only `PLAN.md`. The execution
checkpoint above records the subsequent documentation work.
The owner's follow-up request authorizes refreshing `STATUS.md`
and committing all relevant documentation. It does not authorize
a push or additional device experiments.

## 1. Reconcile the evidence

- [x] Build a claim ledger while drafting `SPHERE-C-BT.md`.
  Record each claim's source, device, date/version, evidence
  category, and remaining uncertainty. Keep this ledger in the
  reference or its source notes; another top-level file is not
  necessary.
- [x] Distinguish authenticated device traffic, human visual
  observation, Sphere C panel/schema, APK code, current local
  implementation, generic specifications, and inference.
  A passing codec test establishes encoding, not device behavior.
- [x] Check implementation claims against current source and
  tests. Preserve historical observations with their dates;
  do not treat every research note as current documentation.
- [x] Resolve known stale or conflicting statements:
  - the old xyL identification versus the observed HSL models;
  - early missing-codec findings versus current access codecs;
  - early provisioning rollback versus later persistent setup;
  - old claims of no live vendor evidence versus later trials;
  - the guide's Aurora-only wording versus Red Flash observation;
  - blanket no-retry wording versus the current bounded DP 102
    selection retry;
  - 32 schema values, 26 panel choices including Color Wheel,
    and 25 entries in the desktop preset grid;
  - remembered DP 102 versus proven active work mode;
  - implemented controls versus the deferred startup-mode rule;
  - early generic chip guesses versus the later, stronger Telink
    GATT fingerprint, which still does not identify an exact MCU.
- [x] Make narrow corrections or add supersession links in
  existing guides where necessary. Keep historical research
  useful as evidence rather than rewriting its chronology.

Start with [STATUS.md](STATUS.md), the
[continuation guide](docs/plans/continuation.md), and the
[documentation index](docs/README.md). Cross-check against
`src/sphere_mesh/` and `tests/`, not live secret-bearing state.

## 2. Write SPHERE-C-BT.md

Audience: someone implementing an independent controller or
interpreting a capture. Make it a self-contained reference to
everything currently known, with explicit unknowns and links to
the detailed evidence. Do not reproduce the entire Mesh standard
or imply that generic SDK functionality works on this product.

Use small tables by topic. Include field width, byte order,
units/range, and evidence wherever applicable. Identify whether
hex examples are complete access messages or only parameters.

### Reference sections and required tables

| Section | Tables and coverage |
|---|---|
| Identity and scope | Sphere C versus original Sphere; app/artifact versions; Mesh company/product/version IDs versus Tuya product ID. |
| Discovery and GATT | Observed advertisement types, service and characteristic UUIDs, directions, notifications, PB-GATT versus Proxy, address rotation, Network ID and Node Identity. |
| Protocol layers | GATT, Proxy SAR, Mesh network/transport/access, key roles, sequence/IV/TID roles, segmentation and practical implementation limits. |
| Provisioning and setup | Ordered handshake/configuration stages, authentication method and evidence, composition read, AppKey addition, model bindings, Network Transmit settings, persistence observations. |
| Composition | All six elements using relative offsets; SIG/vendor model IDs, roles, binding status, tested operations, and unsupported paths. |
| Standard controls | OnOff, Lightness, CTL/Temperature, HSL; Get/Set/Status opcodes, parameter layouts, optional transition fields, units and actual support. |
| Color conversion | Product HSV ranges and text encoding, HSV-to-HSL conversion/scaling, brightness semantics, separation of power and color. |
| Vendor framing | WRITE/WRITE_UNACK/READ/DATA bytes, parameter offsets, command types, record types, lengths, integer byte order, parsing failures. |
| Datapoints | All 17 schema DPs: ID/name, type/range, schema access, actual SIG/vendor route, live evidence, local API support and unknowns. |
| Preset catalog | All 32 DP 102 values, decimal/hex, canonical name, slug/label, panel and GUI visibility, motion evidence. |
| Operation sequences | Select versus activate, power handling, Color Wheel and later color updates, countdown set/cancel, response and recovery behavior. |
| Responses and failures | Empty acknowledgments, typed echoes, delayed snapshots, missing DPs, command-type filtering, timeout ambiguity, partial failure and bounded retry. |
| Remaining surfaces | DP 6 scenes, DP 8 music, opaque DPs, second vendor model, group association, secondary elements, OTA clues and limits. |
| Support and open questions | Observed device behavior versus implemented client behavior, absent features, hypotheses and evidence needed to settle them. |

### Details that must remain explicit

- [x] Use composition-relative addresses; do not make a past node
  address a protocol constant. Six elements do not prove six
  independently addressable LEDs or a pixel buffer.
- [x] Explain that ordinary controls use SIG models even when the
  app exposes them as DPs. DP 4 is not in the saved Sphere C
  schema, although standard CTL exists. No xyL Server was found.
- [x] Document vendor server `0x07d0:0x0004` and client `0x0005`
  without treating the embedded client as another control target.
- [x] Show READ's count field and DATA's lack of a count or
  vendor transaction ID. Distinguish types `01`/`05` from `00`;
  reconcile the APK parser with the current receive path.
- [x] Separate transport acknowledgment, reported value,
  independent readback, and visual confirmation. Missing DP 2
  does not prove write-only access; DP 102 does not prove active
  mode after a restart.
- [x] Explain DP 102 presets versus generic DP 6 scenes. Keep
  hidden values and untested behaviors visibly marked. Aurora
  and Red Flash have motion evidence; other classifications
  remain unknown. Color Wheel is the special solid-color entry.
- [x] Record DP 101's validated zero write separately from
  untested positive countdown behavior. Schema write permission
  is not a tested or supported operation.
- [x] Include synthetic, annotated examples for standard color,
  vendor read, empty DATA, typed snapshot, sequential activation,
  and countdown zero. Check exact bytes with existing codecs and
  fixtures. Do not label synthetic examples as captured traffic.
- [x] Explain single-transmitter ownership, sequence reservation,
  TID handling, and current replay/IV/retransmission limits.
  Keep implementation policy distinct from wire requirements.

Primary source clusters:

- [Model map](docs/research/mesh/step4_sphere_model_map.md),
  [codec audit](docs/research/mesh/step4_codec_audit.md), and
  [live validation](docs/analysis/step4-live-validation.md).
- [Self-provisioning](docs/guides/self-provisioning.md) and
  [capture analysis](docs/analysis/capture.md).
- [DP guide](docs/guides/vendor-datapoints.md),
  [inventory](docs/research/mesh/vendor_dp_inventory.md), and
  [DATA codec](docs/research/mesh/vendor_dp_response_codec.md).
- [Response handling](docs/research/mesh/vendor_type00_runtime_audit.md)
  and [timeout audit](docs/research/mesh/effect_timeout_runtime_audit.md).
- [Color mapping](docs/research/app/sphere_color_mapping_audit.md),
  [phone semantics](docs/research/app/phone_effect_mode_semantics.md),
  and [motion evidence](docs/research/app/effect_motion_classification.md).
- Current `access.py`, `control.py`, `lighting.py`, `vendor.py`,
  `provision.py`, `transport.py`, `state.py`, `daemon.py`, and
  `gui/backend.py`, with their corresponding tests.

## 3. Write REPORT.md as a blog report

Audience: a technically curious reader new to Bluetooth Mesh.
Lead with the result: a usable Linux desktop controller for
the Sphere C, running without the Olight app or cloud. Explain
the investigation through decisions and discoveries, not a commit
log or a directory-by-directory research dump.

Proposed narrative:

1. **The lamp and the question.** Introduce the USB-C Sphere C,
   the desire for local control, and the separate original model.
2. **What the first capture could tell us.** Explain Proxy GATT
   and encrypted Mesh; packet lengths suggested message shapes
   but did not reveal authenticated commands or recover keys.
3. **Following the app.** Describe what APK code, phone data,
   and the product panel revealed about provisioning, models,
   color conversion and presets. State where extraction stalled.
4. **Owning the network.** Explain self-provisioning, standard
   configuration, early rollback, and later durable membership.
   Do not attribute persistence to one setup message without
   evidence that isolates its effect.
5. **From packets to a usable lamp.** Cover standard controls,
   vendor presets, the single-owner daemon, and the desktop UI.
   Use a short concrete interaction to make the result tangible.
6. **The surprises.** HSL versus xyL, HSV conversion, the
   selected-versus-active preset distinction, and why an empty
   acknowledgment is not visible success.
7. **Where the evidence stops.** Discuss incomplete motion
   classification, intermittent timeout, CTL's unclear visible
   effect, no known per-LED command, and transport limitations.
8. **Why firmware comes next.** State the specific questions an
   image might answer, while making clear that working local
   control and useful documentation already exist.

- [x] Keep prose connected and approachable. Define NetKey,
  AppKey, DeviceKey, DP and provisioning when first needed.
- [x] Link technical claims to `SPHERE-C-BT.md` and the evidence.
  Use short examples and at most a few explanatory figures;
  leave byte catalogs and exhaustive tables in the reference.
- [x] Use original diagrams or approved project screenshots if
  useful. Do not copy proprietary app artwork or preview media.
- [x] Distinguish recorded observations from demonstrations made
  for the report. Do not invent experiments, success rates, or
  firmware conclusions to improve the story.

## 4. Other useful work before acquiring firmware

These optional follow-ups are not document prerequisites or
instructions to resume all previously deferred work.

| Priority | Work | Value and dependency |
|---|---|---|
| First product follow-up | Preset video and classification | User supplies a continuous recording in current grid order; segment, classify and record evidence. Agree media license/attribution before adding previews and Patterns/Animations UI. |
| Offline preparation | Firmware capture readiness | Turn existing OTA research into an exact-device checklist and test capture/export handling with synthetic data before risking a one-time update offer. |
| Before expanding protocol use | Focused reliability review | Check durable replay protection, IV recovery and segmented retransmission against current code; scope separate fixes and fault-injection tests as needed. Firmware is not a prerequisite. |
| Optional polish | Desktop acceptance and packaging | Accessibility, keyboard/HiDPI behavior and launcher installation can be checked with a fake daemon; avoid system changes during documentation work. |

Keep the prior deferrals: startup-mode inference/persistence,
music, positive countdowns, hard battery-off tests, CTL trials,
opaque/factory DP writes, multi-Sphere implementation and active
original-Sphere compatibility tests. Include their limitations in
the documents without scheduling their implementation.

## 5. Prepare the firmware handoff

The saved research found no firmware image in the inspected APK
or copied phone artifacts. That is a bounded historical finding,
not a claim that no firmware can be obtained today.

The recorded approximately 140 KB update opportunity concerns
the original magnetic-charge Sphere. An original-Sphere image
would be comparative evidence, not proof of Sphere C internals.
The saved [GATT fingerprint audit][chip-audit] strongly supports
a Telink-derived stack on Sphere C, without identifying its MCU.
Include the OTA/custom UUID pairs and `0x7fdd` service in the
reference, with observed presence separated from unknown command
semantics. USB investigation found charging but no data device
in a limited trial; that does not prove no debug path exists.

[chip-audit]: docs/research/hardware/original_sphere_chip_inference.md

- [x] Summarize the candidate acquisition path from the
  [OTA capture plan](docs/research/hardware/original_sphere_ota_capture_plan.md)
  and [artifact audit](docs/research/hardware/original_sphere_firmware_artifacts.md).
  Prefer capturing verified downloaded bytes inside the official
  app before Bluetooth framing. The actual runtime path still
  needs confirmation; generic SDK classes do not prove its use.
- [x] Define the private artifact manifest: exact model, app/APK
  version and hash, firmware versions, channel/type, full-versus-
  differential flag, size, hashes, metadata and capture origin.
  Preserve raw bytes separately from derived analysis. Redact
  credentials and signed URLs from public documentation.
- [ ] Prepare hooks and export checks as a separate future task,
  using synthetic data where possible. Phone instrumentation,
  account use and device operations need their own concrete plan.
  Do not pair the boxed original or accept an update to prepare
  these two documents. Do not interrupt an OTA as a chip test.
- [x] Rank questions for a future image: DP handlers and hidden
  preset values; mode/reporting behavior; effect timing and
  brightness; any pixel interface; MCU/image format; persistence
  and OTA structure. Mark each as a question, not a promised
  discovery. Encryption, compression or a differential package
  may limit what can be learned.

## 6. Validation and completion

- [x] Check every protocol table against current code and the
  cited evidence. Browse primary specifications if a normative
  detail needs verification; record version and retrieval date.
- [x] Verify example encodings offline using synthetic values.
  Existing codec tests suffice unless a concrete coverage gap
  justifies an additional test.
- [x] Date the historical 235-test result explicitly.
  Do not claim a new pass without running the suite. A prose-only
  change does not require a new full test run.
- [x] Check relative links, Markdown tables and whitespace. Wrap
  prose near 65 columns; allow wider tables and unavoidable URLs.
- [x] Review for secrets, private identifiers, copyrighted media,
  unsupported claims and confusion between device generations.
  Do not use `state/sphere_state.json` to fill documentation gaps.
- [x] Add concise links from `README.md` and `docs/README.md` to
  the report and reference; cross-link the two documents.
- [x] Inspect the final diff and report unresolved questions.
  The original uncommitted handoff was delivered; the owner then
  requested a documentation refresh and local commit.

Done means the blog tells a supported, readable story and the
reference lets another implementer locate every known message,
mapping and limitation without reconstructing the research
history. Neither requires a live experiment or a firmware image.
