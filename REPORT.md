# A local controller for the Olight Sphere C

Bart Massey and Codex — October 4, 2026

The Olight Sphere C on the desk now has a Linux desktop controller.
It can turn the lamp on and off, change brightness and color, and
select the built-in lighting presets. The controller owns its
Bluetooth Mesh network and runs without the Olight app or cloud.
We reached that point without obtaining the lamp's firmware.

There are still things we cannot explain about the device. Some
reported states differ from what we just asked it to do. The
protocol advertises models whose physical purpose is unclear.
Most presets have names and working encodings but no recorded
classification as a pattern or an animation. Those gaps make
firmware interesting; they have not prevented useful local control.

This report tells the story so far. The companion
[Bluetooth protocol reference](SPHERE-C-BT.md) collects the exact
messages, tables, evidence and limitations. Our tested device is
the USB-C **Sphere C**. The earlier magnetic-charge Sphere remains
a separate hardware generation, even where its name or app makes
the two look closely related.

## The first capture raised the right question

The starting point was an Android Bluetooth snoop capture. It
contained more than five minutes of activity, a discoverable GATT
table and hundreds of Mesh Network PDUs. There was plenty to look
at, but no plaintext command saying “make the lamp red.”

The important discovery was the transport: standard Bluetooth Mesh
carried through a Mesh Proxy GATT connection. GATT was the pipe
between the phone and lamp. Inside it, Mesh messages had their own
encryption and authentication. The absence of ordinary Bluetooth
bonding did not make those messages readable.

Packet lengths and timing still helped. Repeated short exchanges
looked like model reads and status replies. Longer requests and
bursts suggested configuration or vendor traffic. But a message
shape cannot establish an opcode, and two different lighting
models can have the same shape. We kept those interpretations as
inferences rather than treating a plausible decode as a result.

The capture also carried a Network ID derived from the network
key. That would let us check a candidate key, but it was not a
practical way to recover one. Replaying encrypted packets would
not solve the problem either: Mesh includes sequence numbers and
replay defenses. We needed access to the existing network's keys
or a way to create a network of our own.
The [capture analysis](docs/analysis/capture.md) preserves the
original evidence and its limits.

## The app supplied a map

The Olight APK and its downloaded Sphere C panel were useful for
different reasons. The APK contained the Thingclips/Tuya machinery
for provisioning, Mesh messages and conversions. The small
product panel contained the lamp's schema, picker entries and
actual UI call sites.

That distinction mattered. A generic SDK can support a feature
that this product never uses. A schema can declare a writable
property without telling us what a physical lamp will do with it.
A product-specific call site is stronger evidence of intended
behavior, but still needs a device test to establish the result.

Tuya calls its application properties *datapoints*, or DPs. At
the panel level, power, brightness and color all look like DPs.
Following them down through the APK showed that ordinary power
and lighting controls become standard Mesh model messages.
Product features such as the preset selector instead use a Tuya
vendor model.

The panel resolved an especially useful ambiguity: the built-in
presets use DP 102, `colour_mode`. They do not use the generic
DP 6 `scene_data` property. Selecting a preset first enters
`colour` work mode through DP 2, then selects DP 102. The schema
contains 32 selector values; the phone picker exposes 26 of them.
One of those is Color Wheel, which opens interactive solid-color
controls. That leaves 25 presets in the desktop grid.

The app research also traced key storage and authenticated cloud
paths. It did not recover the old capture's network keys from the
available artifacts. Fortunately, understanding the provisioning
path gave us another route.
The [DP inventory](docs/research/mesh/vendor_dp_inventory.md)
separates product evidence from generic SDK behavior.

## Giving the lamp a network of our own

Bluetooth Mesh has several keys with different jobs. The NetKey
protects the network layer. An AppKey protects application traffic
and must be bound to the models it can control. Each node also
has a DeviceKey for configuration.

During standard provisioning, the provisioner and device exchange
public keys and use elliptic-curve Diffie–Hellman to derive shared
session material. The provisioner supplies the network key and
address; the DeviceKey is derived by both sides. This does not
require extracting a factory private key from the lamp.

Using that process over PB-GATT, we provisioned the Sphere C with
locally chosen network state. The first successful handshake was
not the end of the story: early attempts later rolled back to an
unprovisioned state. The persistent run completed the app's fuller
standard setup, including all eleven matching model bindings and
Network Transmit configuration.

That difference established a working procedure, but not a single
magic final command. We did not isolate which late operation was
the firmware's persistence checkpoint. The honest conclusion is
that the complete tested sequence worked.

The device then answered authenticated application reads, retained
membership across disconnect, and remained controllable after a
button-off/wake cycle. A later round trip returned it to the phone
app and then to local ownership again. The
[provisioning runbook](docs/guides/self-provisioning.md) records
those operations and the recovery boundaries. Turning the lamp
off is not evidence that its battery-powered controller lost power.

## The standard models did most of the work

Authenticated composition data showed a useful collection of
standard servers: Generic OnOff, Light Lightness, CTL Temperature
and HSL, alongside the vendor models. Reads and acknowledged
writes worked for the primary lighting controls.

Composition also corrected an earlier mistake. Model IDs `0x130a`
and `0x130b` had been mistaken for xyL support. They are actually
the HSL Hue and Saturation servers. The real xyL Server is
`0x130c`, and this Sphere C does not advertise it. The controller
now checks capabilities rather than sending a message to a model
whose number merely looked familiar.

Color needed another translation. The phone wheel uses HSV: hue,
saturation and value. The Mesh message carries HSL: hue,
saturation and lightness. HSV value cannot simply be copied into
HSL lightness. Following the panel and APK established the
conversion, which the desktop controller now applies.

Even a correctly encoded Set did not always mean that a later
Get would return the same raw numbers. One HSL restoration was
acknowledged exactly, then settled to a different tuple. Further
calibration suggested conversion or quantization inside the
firmware, rather than a single fixed scaling error. The separate
Lightness state also differed from HSL lightness. We preserve
those states as reported instead of pretending they are one value.

CTL Temperature was another useful reminder of the evidence
boundary: the model accepted commands, but the desktop trial
showed no obvious visible temperature change. The GUI leaves that
control out. The [lighting validation](docs/analysis/step4-live-validation.md)
and [desktop observations](docs/analysis/step6-operational-validation.md)
record the difference between protocol support and visible effect.

## A reply is not the same thing as success

The vendor protocol made that distinction unavoidable. A read can
produce a short empty acknowledgment followed by a larger status
snapshot. The snapshot can contain datapoints we did not request
and omit one we did. DP 2, the work mode, is regularly absent.

Writes have a related problem. An empty DATA reply confirms that
an expected response arrived, but supplies no value. A typed echo
reports a value, but is still not an independent readback. Neither
one proves what a person can see on the lamp.

The service therefore reports four things separately: transport
acknowledgment, a value in the reply, an independent readback, and
visual verification. Automated calls never claim the last one.
That may seem fussy until a command times out. The lamp may have
acted and lost the reply; “failed request” does not necessarily
mean “unchanged device.”

One Cotton Candy activation did time out during a GUI trial.
Later transitions worked, so the evidence does not justify calling
the preset unsupported. The current service identifies which
activation step failed and may retry the DP 102 selection once
after a timeout. It does not repeat the entire activation. The
[timeout investigation](docs/research/mesh/effect_timeout_runtime_audit.md)
explains both that mitigation and why the original cause remains
unconfirmed.

Missing work mode also creates a visible UI limitation. After a
daemon restart, the selected DP 102 preset may be readable while
the service cannot establish that it is active. Treating every
remembered preset as active would be a useful simplifying policy
for a single-controller setup, but it would still be an assumption.
That proposed startup rule remains deferred.

## From a control script to a desktop application

The working system has one daemon holding the Bluetooth connection
and authoritative state. The CLI and Qt desktop client talk to a
private Unix socket; neither owns a second Mesh transmitter.

This arrangement matters because a network sequence number is
security state. Reusing it with the same source and security
context can reuse an encryption nonce. The daemon reserves and
persists sequence progress before queueing outgoing traffic, and
an exclusive lock prevents another local instance from owning the
same state at the same time.

The desktop app provides power, a logarithmic brightness slider,
phone-compatible color controls, a searchable preset grid and
battery status when the device supplies it. A normal session can
select Aurora, adjust brightness, then enter Color Wheel and pick
a solid color. Aurora has been observed animating, and Red Flash
has been observed flashing. The remaining names and still images
do not tell us which are static patterns and which animate.

On October 4 we checked the lamp again after the owner reported
sleeping and waking it. The daemon had not been running. Starting
it with the existing state was enough to reconnect and obtain
authenticated power, lighting and vendor replies. The lamp
reported power on, Flame selected, battery at 100%, and both
countdowns at zero. DP 2 was again missing. We made no lighting
writes and did not infer Flame's appearance from that response.

The existing automated suite also passed: 235 tests in 12.91
seconds. Those tests cover code, codecs and simulated service
behavior; they do not replace the separately recorded hardware
observations. Installation and usage are in the
[desktop guide](docs/guides/desktop-app.md).

## What a firmware image might answer

There is no known command here for painting individual LEDs.
Composition reports five secondary OnOff elements, but neither
their successful bindings nor their Boolean model type tells us
what physical function they represent. Calling them five LED
zones would get ahead of the evidence.

There are other unresolved surfaces: opaque timer and power-memory
datapoints, hidden preset values, generic scene handling, and the
internal relationship between color state and an active effect.
Positive countdowns, microphone streaming and true battery-off
persistence tests remain deliberately deferred. Full IV Update
recovery, durable receive replay protection and reliable outbound
segment retransmission are also unfinished client work, which
does not require firmware to address.

We do have a stronger hardware clue than a generic Tuya label.
The captured Sphere C GATT table matches Telink OTA and custom
UUIDs and an unusual `0x7fdd` service found in Telink's SDK. That
strongly suggests a Telink-derived software stack. It does not
identify an exact chip: protocol constants can survive a port to
another target. The [fingerprint analysis](docs/research/hardware/original_sphere_chip_inference.md)
keeps those two claims separate.

The available APK and copied phone files contained OTA machinery
but no recovered device image. A recorded update opportunity
concerns an original magnetic-charge Sphere, so any image obtained
that way would first tell us about that model. It would be
comparative evidence for Sphere C, not a substitute for its own
firmware.

The planned acquisition point is inside the official app, after
download and verification but before Bluetooth framing. We can
prepare the metadata checklist and artifact handling now, while
leaving the boxed device's possible one-time update offer intact.
An image might expose handlers and effect tables, or it might be
wrapped, encrypted or differential. It is a promising next source
of evidence, not a promise to explain every behavior.

The most useful product follow-up is simpler: record the visible
presets, classify their motion, and make licensed previews for
the desktop app. Meanwhile, the lamp is already useful from
Linux, and the [protocol reference](SPHERE-C-BT.md) gives the next
implementation a much firmer starting point than an encrypted
capture and a list of plausible guesses.
