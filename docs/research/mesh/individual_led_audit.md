# Sphere C secondary-element / individual-LED audit

## Question and conclusion

The Sphere C exposes six Bluetooth Mesh elements. The primary element
has the normal lighting models and Generic OnOff. Each of the five
secondary elements has one Generic OnOff Server and no other model.

The following is proven for the active local allocation:

| Element | Address | Authenticated composition | AppKey bind |
|---|---:|---|---|
| primary | `0x0118` | lighting models + `0x1000` | success |
| secondary 1 | `0x0119` | only `0x1000` | success |
| secondary 2 | `0x011a` | only `0x1000` | success |
| secondary 3 | `0x011b` | only `0x1000` | success |
| secondary 4 | `0x011c` | only `0x1000` | success |
| secondary 5 | `0x011d` | only `0x1000` | success |

It is **not** proven that those five Boolean models select five
physical LED regions, emitters, driver rails, animation layers, or
anything else visible. No saved local transaction addresses a
secondary element with an application-model command. All decoded local
Generic OnOff and lighting traffic targets the primary element.

The most defensible current description is therefore:

> Five separately addressed Generic OnOff model instances exist and
> are bound to the local AppKey. Their physical semantics are unknown.

Calling them "individual LEDs" or even "five LED channels" would go
beyond the evidence. A Generic OnOff Server can expose only a Boolean
state. The composition supplies no channel names: all six element
location fields are zero.

## Evidence ledger

### 1. Authenticated Composition Data: proven

`state/sphere_state.json` records Composition Data page 0 for the
active node. It reports CID `0x07d0`, PID `0x0200`, VID `0x3530`, and
six elements. Element zero contains the standard lighting surface,
two vendor models, and Generic OnOff `0x1000`. Elements one through
five contain only `0x1000`.

This is not a model-ID inference from encrypted packet lengths. It was
obtained with Config Composition Data Get under the locally derived
DeviceKey and parsed from the authenticated response.

`state/provision_run.log` contains the same full composition at three
separate allocations:

- line 647: primary `0x010c`;
- line 739: primary `0x0112`; and
- line 1007: primary `0x0118`.

The repeated result rules out an address-specific parsing accident.
The current saved composition at `0x0118` maps element indices to
consecutive addresses exactly as Bluetooth Mesh requires.

The element count is also operationally relevant: the provisioner
advances `next_unicast` past all six addresses, so it does not allocate
another node over the secondary range.

### 2. Bind success on every secondary: proven

The provisioner derives bind targets from composition, rather than
assuming the element layout. Its allowlist includes SIG model
`0x1000`, so it binds the primary instance and all five secondary
instances.

An offline authenticated decode of `state/provision_run.log`, using
the locally saved NetKey and per-node DeviceKey, recovers the following
current-node exchanges. The leading status byte in every `0x803e`
response is `00` (Success):

```text
Config Model App Bind 0x0119, AppKey 0, model 0x1000
Config Model App Status: Success, 0x0119, model 0x1000
Config Model App Bind 0x011a, AppKey 0, model 0x1000
Config Model App Status: Success, 0x011a, model 0x1000
Config Model App Bind 0x011b, AppKey 0, model 0x1000
Config Model App Status: Success, 0x011b, model 0x1000
Config Model App Bind 0x011c, AppKey 0, model 0x1000
Config Model App Status: Success, 0x011c, model 0x1000
Config Model App Bind 0x011d, AppKey 0, model 0x1000
Config Model App Status: Success, 0x011d, model 0x1000
```

The earlier durable node at `0x0112` has the same five successful
binds at `0x0113` through `0x0117`. Thus the evidence is stronger than
the source code's intent or a console claim: each bind has a matching,
DeviceKey-authenticated success response from the node.

This proves that application traffic under AppKey index 0 is eligible
for those server instances. It does not prove what the firmware does
with an OnOff message after accepting it.

### 3. Saved application traffic: primary only

Offline NetKey/AppKey decryption of `state/provision_run.log` finds:

- Generic OnOff Get/Set and authenticated Status at primary `0x0118`;
- Lightness, CTL Temperature, and HSL traffic at `0x0118`; and
- no AppKey access request to, or Status from, `0x0119..0x011d`.

The same log contains the earlier read-only OnOff validation for
primary `0x0112`, but none for `0x0113..0x0117`.

`old/captures/self-provisioning/self-config-full-20260926.btsnoop`
can also be decrypted with the saved local keys. Its application-model
traffic consists of OnOff Get/Status at primary `0x0112`. It contains
no authenticated application transaction involving a secondary
address.

The old app-owned capture, `old/captures/sphere-c.btsnoop`, cannot
answer this question. Its old NetKey and AppKey are unavailable, so
source and destination addresses remain encrypted. Its repeated packet
shapes and response fans are useful structural observations, but they
cannot identify an element. A six-response-looking burst must not be
treated as evidence for six local elements: the capture includes other
mesh traffic, group-address candidates, retransmissions, and varying
status payloads.

One authenticated historical primary-element status to group `0xd000`
also appears in the local full capture/log. That supports prior primary
publication behavior, not secondary addressing.

### 4. APK and local implementation: configuration, not semantics

The inspected Olight/Thingclips setup path walks every composition
element and binds every matching `0x1000` server. The local provisioner
faithfully mirrors that generic loop. This explains why the five
secondary instances are bound.

It does not establish their product-specific meaning. The reviewed
post-provision path publishes or subscribes the **primary** Generic
OnOff Server in the relevant branches. The local runtime controller
also accepts one node unicast and always sends basic control to that
primary address. Its lighting controller finds the element hosting a
requested lighting model; every authenticated lighting server is on
the primary element.

No local APK note, script, CLI, or decoded capture documents a mapping
such as `element 1 = red`, `element 2 = green`, a spatial zone, or an
LED index. The five secondaries may be a product feature, a reused Tuya
composition template, internal logical states, or real output groups.
The artifacts do not distinguish these explanations.

The original Sphere's public 69-white/54-RGB LED counts do not fill
this gap. They concern another hardware generation and, even there,
do not reveal driver topology. Five Boolean mesh models cannot by
themselves address 123 emitters individually.

## Confidence classification

### Proven

- The Sphere C reports six elements under authenticated Composition
  Data.
- The active secondary addresses are `0x0119..0x011d`.
- Each secondary advertises only Generic OnOff Server `0x1000`.
- All five secondary `0x1000` instances returned authenticated AppKey
  bind success for AppKey index 0 at both durable allocations.
- Existing local live validation exercised only the primary element.
- The current CLI has no element-address selector for Generic OnOff.

### Likely, but not proven

- Each secondary should accept an AppKey-authenticated Generic OnOff
  Get and answer from its own element address. Composition plus bind
  success strongly predicts this, but it has not been recorded.
- The five models were intentionally emitted by the firmware rather
  than created by the parser. Three authenticated composition reads
  agree exactly.
- They may represent internal independently modeled channels or
  subfunctions. Their repeated OnOff-only layout makes that plausible,
  but a fixed SDK composition template is also plausible.

### Unsupported

- There are five independently controllable physical LED regions.
- The elements map to RGB, white, animation layers, or five spatial
  zones in element order.
- A Set to one secondary will leave primary power and mode unchanged.
- Secondary OnOff can select an individual LED. Neither the model nor
  the hardware evidence supports emitter-level indexing.
- Old encrypted response fans are replies from the five secondaries.
- Successful configuration bind alone proves visible behavior.

## Safe experiments

### A. Offline verification (no radio)

Before another live run, preserve a reusable decoder report that lists
timestamp, source, destination, opcode, and parameters without printing
keys. Apply it to `state/provision_run.log` and all four local snoop
captures. This will make the current negative result reproducible and
catch any secondary transaction overlooked by prose notes.

The existing capture helper decrypts unsegmented AppKey access. A small
audit-only extension would add unsegmented DeviceKey access so the bind
exchange above can be reproduced directly. Do not commit decoded keys
or raw state-file contents.

### B. Read-only live element sweep (recommended next test)

Open one proxy session using the authoritative state and serialize six
acknowledged Generic OnOff Gets to:

```text
0x0118 0x0119 0x011a 0x011b 0x011c 0x011d
```

For each request, require both:

1. Generic OnOff Status `0x8204`; and
2. a source address exactly equal to the requested element.

`MeshNode.request()` already enforces the source/opcode pair. Record
present, optional target, remaining time, latency, timeout, and any
unsolicited status. Send sequentially, not concurrently, and use the
normal durable sequence allocator/state lock. A Get is read-only at the
access-model level, although it necessarily consumes controller
sequence numbers and updates local state.

Do not use the current basic-control CLI unchanged: it always targets
the selected node's primary address. Use an audit harness that derives
the addresses from saved composition. Never hard-code addresses beyond
this recorded run.

Interpretation:

- five correctly sourced Status replies prove addressability and
  readable Boolean states, not visible channel control;
- timeouts weaken the assumption that all bound instances are runtime
  servers, but do not by themselves prove absence;
- replies sourced only by the primary would contradict normal element
  semantics and merit a packet-level check.

Repeat the sweep while the user changes lamp state with the physical
button. This remains read-only from Linux. Correlation can show whether
secondary states mirror primary power, remain constant, or track modes.
It still cannot prove independent control.

### C. Read-only configuration confirmation (optional)

Implement Config SIG Model App Get/List for each secondary and request
the bound AppKey indexes under the DeviceKey. This independently
confirms retained binding after reconnect or wake. It is lower value
than experiment B because the authenticated bind statuses already
prove initial success, and a binding says nothing about physical output.

### D. Reversible same-value Set (requires explicit approval)

Only after B succeeds, send an acknowledged Generic OnOff Set to one
secondary using its **already reported present value** and a fresh TID.
This tests write dispatch with minimal intended visible change. Require
a correctly sourced Status, then re-read all six elements and the three
primary lighting states.

Even a same-value Set may have undocumented side effects, so this is
not read-only and should not be bundled into an ordinary audit run.

### E. Reversible visible Set (required for physical mapping)

Proving independent output control ultimately requires mutation. With
the user observing or recording video:

1. read and save all six OnOff states plus primary lighting state;
2. choose one secondary whose present state is known;
3. Set it to the opposite value with an acknowledged request;
4. observe power, color, spatial output, animation, and button behavior;
5. immediately restore the saved Boolean value;
6. re-read every state; and
7. stop after any unexpected effect.

Test one secondary per run. Do not broadcast, use a group address, or
send unacknowledged Sets. Treat silence, a success Status with no
visible change, and mirrored whole-lamp behavior as distinct outcomes.
Only repeated, restored tests can justify naming a physical channel.

## Recommended terminology until tested

Use "five secondary Generic OnOff elements" in code and docs. If the
read sweep succeeds, "five independently addressable Boolean model
instances" is also accurate. Reserve "LED channel", "zone", and
"individual LED" for mappings demonstrated by observed, reversible
per-element Sets.
