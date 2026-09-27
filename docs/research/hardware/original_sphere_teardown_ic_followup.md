# Original magnetic Sphere teardown/MCU follow-up

Research date: 2026-09-26.

This note is limited to the first Olight **Sphere**, which uses the
USB-A magnetic charging cable and is certified as FCC ID
`2A44S-SPHERE`. It excludes the later USB-C **Sphere C**, FCC ID
`2A44S-SPHEREC`.

## Result

No public teardown, gallery, video, forum comment, or FCC exhibit I
found provides a readable processor manufacturer or part number for
the original magnetic-charge Sphere.

The strongest primary visual evidence is the original Sphere FCC
internal-photo exhibit, document `7603966`. Page 4 plainly shows the
radio/control PCB and its QFN processor next to the PCB antenna and
metal-can crystal. The entire top-mark area of that QFN has been
covered with opaque pink/red material. There are no exposed letters
or digits from which a part number can be transcribed. Thus the FCC
photo confirms the package location and direct-on-PCB radio layout,
but not the chip identity.

The only clearly original-Sphere independent teardown located is a
51-second Reddit video posted before Sphere C launched. It displays
the shell and cylindrical LED assembly while the lamp is operating,
but never removes or turns over the lower controller/power assembly.
No processor package appears in a usable frame, and no comment names
one.

The later, much clearer 13-photo Reddit teardown that is often cited
as a Sphere teardown is actually **Sphere C**. It cannot be used as
evidence for the original Sphere's processor.

## 1. FCC internal photos: exact evidence

Primary/mirror source:

- [Original Sphere internal-photo exhibit, FCC ID
  2A44S-SPHERE, document 7603966](https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966)
- [Original Sphere filing index](https://fccid.io/2A44S-SPHERE)

The exhibit page identifies all of the following:

- applicant: Shenzhen Olight E-Commerce Technology Co., Ltd.;
- product code: `SPHERE`;
- FCC ID: `2A44S-SPHERE`;
- application date: 2024-08-26;
- document type: Internal Photos;
- five pages;
- document ID: `7603966`;
- SHA-256:
  `c8661b501fd27ce3a841372ea35014c8cf1a58f40928024f817fc420f81ffce7`.

I visually inspected all five rendered pages.

### Page-by-page observations

- Pages 1-2 show the opened product, cylindrical LED structure,
  battery, and lower electronic assembly. The board markings are too
  small for component identification.
- Page 3 shows the circular charging/switch board and a close view of
  factory pads on the narrow control PCB. Pad silkscreen includes
  `TX`, `RX`, and `3V`. Another short label appears consistent with
  `SWS`, but this raster photograph is not sharp enough to treat that
  reading alone as an exact MCU identifier.
- Page 4 provides the decisive radio-board view. A meandered PCB
  antenna is boxed and labeled `ANT`. Immediately above it is a large
  square leaded QFN, with a metal-can crystal beside it. Opaque
  pink/red material covers the complete center/top-mark region of the
  QFN. No processor marking survives visibly.
- Page 5 shows the `700mAh 3.7V` cell. It does not add processor
  evidence.

The pink/red area is localized to the processor top rather than being
a general blur or low-resolution artifact: the package leads, nearby
passives, antenna trace, and crystal remain visible. Whether the
material was applied for confidentiality or another lab reason is not
stated in the exhibit, so only the visual fact should be asserted.

### What can and cannot be inferred

The QFN beside a 2.4 GHz PCB antenna and crystal is very likely the
radio SoC and probably the application MCU. The FCC filing also says
modular equipment does not apply. That supports a direct-on-board BLE
SoC architecture, not a certified plug-in radio module.

It does **not** distinguish TLSR8258, TLSR8278, another Telink family,
or a non-Telink MCU. Package shape, probable pin count, the possible
`SWS` pad, and board layout are family clues, not a unique part-number
readout.

## 2. Genuine independent teardown: Reddit video

Source:

- [“For scienxe” original-Sphere teardown video and
  discussion](https://www.reddit.com/r/flashlight/comments/1j3ktfc/for_scienxe/)

The post was published 2025-03-04. The author says that the unit was
hard to open and identifies it in a reply as “the olight sphere.” The
51-second video shows the cut-open transparent shell and the full
cylindrical LED cage in operation. It is useful confirmation of the
mechanical/LED construction.

It is original-Sphere evidence rather than Sphere C evidence because
it predates Olight's April 2025 Sphere C launch. Olight's launch press
release is dated 2025-04-14, and a contemporaneous Sphere C video
calls it a new product in the April 14-18 anniversary sale:

- [Olight anniversary release announcing Sphere C,
  2025-04-14](https://www.prnewswire.com/news-releases/celebration-des-18-ans-dolight--illuminer-les-reves-de-plus-de-20-millions-dutilisateurs-dans-le-monde-302427893.html)
- [“Olight Sphere C - All Modes (As of April 2025),” published
  2025-04-17](https://www.youtube.com/watch?v=1K4pHr4GUZc)

Frame-by-frame inspection at one-second intervals found no clear view
of the lower control PCB. The video keeps the LED cage fitted to the
base and never exposes a processor top marking. The discussion talks
about construction, LED density, light modes, and the app, but has no
occurrence of `chip`, `MCU`, `processor`, or `Telink`, and no comment
offers a component number.

## 3. The frequently cited 2026 gallery is Sphere C

Source:

- [“Anyone curios about whats inside of an OLight Sphere?”
  13-photo gallery](https://www.reddit.com/r/flashlight/comments/1rdrq5m/anyone_curios_about_whats_inside_of_an_olight/)

This gallery must be excluded from original-Sphere MCU claims.

Exact evidence:

- The first gallery photo shows three lamps; the disassembled/right
  lamp has a cable inserted into a side port. That is the USB-C
  charging arrangement, not the original Sphere's bottom magnetic
  contacts.
- A commenter directly says, “Didn't know they made a USB-C version.
  Cool tear down!”
- The post was published 2026-02-24, well after Sphere C's April 2025
  launch.
- The dedicated Sphere C FCC filing is separately indexed as
  `2A44S-SPHEREC`, not `2A44S-SPHERE`:
  [Sphere C internal-photo exhibit](https://fccid.io/2A44S-SPHEREC/Internal-Photos/Internal-photos-8483616).

No gallery comment contains `chip`, `MCU`, `processor`, or `Telink`.
Even if one of its later photos yields a readable IC, that would
identify Sphere C hardware only and cannot be projected backward onto
the original magnetic-charge board.

This also means that describing this gallery as an “independent
original Sphere teardown” is incorrect. It is an independent Sphere C
teardown.

## 4. Reviews, forums, mirrors, and videos checked

- [1Lumen original Sphere review](https://1lumen.com/review/olight-sphere/)
  explicitly says opening the Sphere requires destructive disassembly
  and expresses a wish to see inside. It supplies no teardown photo or
  IC name.
- Searches of BudgetLightForum, Taschenlampen-Forum, Reddit, YouTube,
  Bilibili, Russian-language pages, and Chinese-language pages found
  no additional teardown tied to the original Sphere and no processor
  transcription.
- FCC mirrors and general review pages repeat the exhibit metadata or
  consumer specifications. None transcribes the covered QFN.
- Searches for exact combinations of `2A44S-SPHERE`, `SWS`, `Telink`,
  `TLSR8258`, `chip`, `MCU`, and `processor` found no source that names
  the original Sphere processor.

Negative search results are not proof that no private or unindexed
photo exists. They do establish that the currently indexed public
sources do not support a defensible exact part number.

## 5. Evidence grading and conclusion

| Claim | Strength | Reason |
| --- | --- | --- |
| Original board has an on-PCB radio/controller QFN | High | FCC page 4 shows it beside crystal and PCB antenna |
| Processor top marking is unreadable | High | Complete mark area is visibly covered by opaque material |
| Factory pads include UART-style labels | High | `TX`, `RX`, and `3V` are legible on FCC page 3 |
| Another pad is `SWS` | Medium | Appearance is consistent, but source raster is soft |
| MCU is from a Telink family | Medium | Possible `SWS` plus other project-level protocol evidence; no package mark |
| MCU is specifically TLSR8258 | Low | Compatible hypothesis, no unique public discriminator |
| 2026 Reddit gallery is original Sphere | False | Side USB-C cable and explicit USB-C comment identify Sphere C |

The defensible endpoint remains: **the original magnetic-charge
Olight Sphere's public photos strongly suggest an integrated BLE SoC,
and may support a Telink-family inference, but no public photo or
comment identifies its exact processor.** A straight-on macro photo
of the uncoated package from a privately opened original Sphere, or a
non-destructive debug/firmware chip identifier, is still required.
