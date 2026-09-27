# Original Olight Sphere hardware: public evidence

Research date: 2026-09-26. This note is about the original
magnetic-charge **Sphere**, FCC ID `2A44S-SPHERE`, not the later
USB-C **Sphere C** (`2A44S-SPHEREC`).

## Bottom line

The public record firmly establishes a Bluetooth 5.0, 2.4 GHz
device with 1 and 2 Mbit/s GFSK PHYs, a 0.22 dBi PCB antenna, a
3.7 V 700 mAh internal battery, magnetic CC/CV charging at 4.2 V
and 0.5 A, and 123 LEDs (69 white plus 54 RGB). The FCC filing has
five pages of internal photographs, but the available photographs
do **not** provide a defensible, readable part number for the main
MCU/radio, external flash, or charger IC. No public part-number
claim for those ICs should be treated as confirmed.

## Confirmed facts

### Regulatory identity and dates

- FCC ID: `2A44S-SPHERE`; grantee: Shenzhen Olight E-Commerce
  Technology Co., Ltd.; FRN `0032024796`; model `SPHERE`; device
  description “PORTABLE LED BULB.” The application and grant are
  dated 2024-08-26. The certification is a non-modular Part 15C
  Digital Transmission System authorization.
- The authorized range is 2402–2480 MHz. The FCC grant gives a
  maximum conducted power of 0.002786 W (2.786 mW, about
  4.45 dBm). The filing index lists EMTEK (Dongguan) as the test
  laboratory and Timco Engineering as the TCB.
- The FCC exhibit index identifies these original-Sphere files:
  internal photos document `7603966` (five pages, created
  2024-08-24, SHA-256
  `c8661b501fd27ce3a841372ea35014c8cf1a58f40928024f817fc420f81ffce7`),
  BLE test reports `7603977` and `7603978`, antenna specification
  `7603974`, RF exposure report `7603980`, and label document
  `7603982`. Internal and external photos were held under
  short-term confidentiality until 2025-02-23; the block diagram,
  schematic, and operational description remain metadata-only /
  long-term-confidential in the public mirror.
- Canadian certification shown in Olight's manual is
  `IC: 28283-SPHERE`.
- Olight's manual declares conformity with the EU Radio Equipment
  Directive 2014/53/EU and RoHS 2011/65/EU plus (EU) 2015/863.
  It specifies Bluetooth operation at 2402–2480 MHz, maximum
  output no more than 10 dBm, and control distance no more than
  25 m. This is the manufacturer's declaration in the manual; a
  Sphere-specific signed EU DoC was not found in the currently
  indexed Olight download archive.

Primary/mirror sources:

- [FCC filing index and grant](https://fccid.io/2A44S-SPHERE)
- [FCC internal-photos exhibit metadata](https://fccid.io/2A44S-SPHERE/Internal-Photos/Internal-Photos-7603966)
- [FCC BLE test report 1, document 7603977](https://fcc.report/FCC-ID/2A44S-SPHERE/7603977.pdf)
- [FCC RF-exposure report mirror, document 7603980](https://manuals.plus/m/0b29e73caa808c3a87adec9c63e62b78a12aed9658f111c889280be69bec06b2)
- [Olight-hosted Sphere manual](https://www.olightstore.it/file/sphere-user-manual.pdf)

### Radio

FCC report `EDG2406270030E01001R` says:

- Bluetooth version: 5.0.
- PHY/data rates: 1 Mbit/s and 2 Mbit/s.
- Modulation: GFSK.
- Channels: 40.
- Frequency range: 2402–2480 MHz.
- Antenna: PCB antenna, 0.22 dBi gain.
- Highest measured conducted power: 4.45 dBm on the 2 Mbit/s PHY
  (the value behind the 0.002786 W grant).
- Test supply: 4.2 V from USB/charger or 3.7 V from battery.
- Declared operating range: 0–40 °C.

The RF-exposure report also contains a 13.56 MHz “NFC” field-
strength evaluation (30.68 dBµV/m and calculated peak output
about -64.48 dBm). That is a confirmed test-report entry, but the
public operational description does not explain the feature. It
must not be taken by itself as proof of a user-accessible NFC tag,
NFC controller, or protocol endpoint.

Important unit correction: one extracted test-report table prints
“4.45 dBm (0.002786 mW).” That parenthetical is dimensionally a
typo. 4.45 dBm is 2.786 mW, and the FCC grant correctly records
0.002786 **W**.

### Battery and charging

The manufacturer manual confirms:

- Internal battery: 3.7 V, 700 mAh, 2.59 Wh.
- Charging is through the bottom magnetic charging port using the
  included USB-A magnetic cable.
- Cable input: 5 V, 1 A.
- Cable output: CC/CV, 4.2 V ±0.05 V, 0.5 A.
- Full-charge time: about two hours.
- The battery is included in the quoted 93 g product mass and is
  not presented as user replaceable.

These numbers establish the cell's electrical rating, but not its
manufacturer, chemistry subtype beyond the lithium-battery safety
language, pouch dimensions, protection IC, or connector type.

### LEDs and mechanical envelope

The Olight manual confirms:

- 69 white LEDs, type `TZ.123.S2-3014W1Y-003-3`, stated maximum
  0.5 W per LED, 2700 K, CRI 80.
- 54 RGB LEDs, type `TZ.0807RGB-TX1812IXU-F01+`, stated maximum
  0.165 W per LED.
- Total: 123 LEDs.
- Size: 60 mm diameter by 54.5 mm high; mass 93 g including
  battery; 1.5 m impact rating. Olight manual revisions indexed
  online disagree between IP56 and IP57; the later Olight-hosted
  manual says IP57. Record the revision when relying on the ingress
  rating.
- The user-facing construction is a transparent outer shell,
  decorative ring, bottom switch, and magnetic charging port.
  Olight states that the luminaire's light source is not
  replaceable and the whole luminaire should be replaced at end of
  light-source life.

An independent 1Lumen review independently counted the same 69
white and 54 RGB designations from the manual and notes that
opening the unit requires destructive disassembly. A 2026 Reddit
gallery documents a destructive teardown, but it is secondary
evidence and postdates the FCC filing; it is useful for layout
context, not for IC identification.

- [1Lumen original Sphere review](https://1lumen.com/review/olight-sphere/)
- [Independent teardown gallery and discussion](https://www.reddit.com/r/flashlight/comments/1rdrq5m/anyone_curios_about_whats_inside_of_an_olight/)

## Component-identification audit

| Requested item | Publicly confirmed | Not confirmed |
| --- | --- | --- |
| MCU / BLE radio | Bluetooth 5.0, 1M/2M GFSK, 40 channels; radio and control electronics are not an FCC-certified plug-in module | Vendor and part number; whether radio and application MCU are one SoC |
| Program / asset flash | No readable, authoritative flash marking in the public exhibits | Vendor, capacity, bus, and whether storage is internal to the SoC |
| Charger / power IC | 4.2 V ±0.05 V, 0.5 A CC/CV system-level charging specification | Charger IC, protection IC, fuel gauge, LED-driver topology |
| Battery | 3.7 V, 700 mAh, 2.59 Wh internal battery | Cell maker, exact cell model/dimensions, protection-board marking |
| RF antenna | PCB antenna, 0.22 dBi | Exact matching-network values |
| NFC-like circuit | FCC RF-exposure report includes a 13.56 MHz NFC measurement | Purpose, IC, coil layout, protocol, or accessibility |
| LED population | 69 white plus 54 RGB, with Olight-listed LED codes | LED die maker and per-board wiring/topology |

### Why no MCU/flash/charger part number is reported

The FCC internal-photo exhibit is real and public, but the relevant
top markings are obscured, too small, or not photographed squarely
enough to read reliably. The block diagram and schematic are not
public. Searches of the FCC text, Olight manuals/downloads, reviews,
and the independent teardown found no trustworthy transcription.
Package shape, pin count, or resemblance to parts used in other
Olight products is insufficient evidence. Any exact IC name offered
without a legible macro photograph, board under magnification, or a
firmware/debug identifier should be labeled speculation.

One useful marking *is* readable on the original Sphere control PCB:
the factory-pad group includes `SWS`, alongside `TX`, `RX`, `3V`, and
other test labels. Telink defines SWS as its proprietary “Single Wire
Slave” programming/debug interface, and Tuya's official TLSR825x
documentation instructs manufacturers to connect a programmer's SWM
pin to the target's SWS pin. This is strong evidence for a Telink
MCU family, especially when combined with the Telink SDK fingerprints
measured on Sphere C. It still does not identify the exact Telink part:
the QFN top marking in the FCC photograph is deliberately obscured.

- [Telink TLSR8258 datasheet: SWM/SWS interface](https://wiki.telink-semi.cn/doc/ds/DS_TLSR8258-E_Datasheet%20for%20Telink%20BLE%2BIEEE802.15.4%20Multi-Standard%20Wireless%20SoC%20TLSR8258.pdf)
- [Tuya TLSR825x platform and SWS flashing](https://developer.tuya.com/en/docs/iot-device-dev/bluetooth_platform_ble_tlsr825x?id=Kcnreatkaj9er)
- [Tuya TLSR825x Bluetooth Mesh lighting SDK](https://developer.tuya.com/en/docs/iot-device-dev/bluetooth-sigmesh-lighting-sdkdemo-instructions?id=Ka5e0524ewo4n)

## Carefully bounded inferences

The following are engineering inferences, **not confirmed facts**:

- Because the FCC application says modular approval does not apply
  and specifies a PCB antenna, the BLE radio is almost certainly
  implemented directly on the Sphere's PCB rather than as a
  separately certified radio module.
- A single BLE SoC may also be the main application MCU. This is a
  common architecture for a device of this size, but the public
  evidence does not exclude a separate controller.
- The 13.56 MHz FCC entry suggests a near-field coil or resonant
  circuit exists somewhere in the tested sample. Its likely use
  cannot be assigned from the public record. In particular, do not
  conflate it with the magnetic charging contacts: magnetic
  attachment and 13.56 MHz RF are independent properties.
- The 123-LED count and teardown imagery imply multiplexed or
  serially controlled LED subassemblies, but neither the bus nor the
  driver topology is published.
- The 3.7 V / 700 mAh rating is consistent with a single-cell
  rechargeable lithium pack. The exact lithium chemistry and pack
  protection arrangement remain unknown.

## Most useful next evidence

To close the component gaps, obtain high-resolution, straight-on
macro photographs of both sides of every original-Sphere PCB after
removing shields, adhesive, and conformal coating where safe. Record
all laser markings before desoldering. Continuity tracing around the
PCB antenna, SWD/JTAG pads, magnetic-charge contacts, battery leads,
and any 8-pin memory package would then distinguish a BLE SoC,
external flash, charger/protection chain, and LED-driver buses.
Comparisons must use FCC ID `2A44S-SPHERE`; Sphere C boards are a
different certified product.
