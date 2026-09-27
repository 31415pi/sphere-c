# Bluetooth Mesh device secrets

## Scope and terminology

This note covers a provisionee/node, not the provisioner's database.
It distinguishes protocol requirements from common implementation
choices. Bluetooth Mesh Protocol v1.1 is the normative source; the
Bluetooth SIG security and feature papers are explanatory sources.

"Persistent" below means retained across ordinary reset and power
loss. The Mesh Protocol defines protocol state and required behavior,
but generally does not prescribe a particular flash layout, secure
element, or key-storage API.

## Executive conclusion

A generic, interoperable Mesh device does **not** need a universal
factory secret. Before provisioning, it must have a unique, immutable
Device UUID, but that UUID is advertised in plaintext and is not a
secret. It must participate in P-256 ECDH during provisioning, but its
key pair may be generated ephemerally for that provisioning attempt.

Long-lived factory secrets exist only when the product selects an
optional provisioning design:

- a static OOB authentication value;
- a static P-256 private key for OOB public-key provisioning; or
- for certificate-based provisioning, the private key corresponding
  to the public key bound to the Device UUID in the device certificate.

After successful provisioning, every node has a derived, unique
Device Key and at least the primary NetKey. It also retains the
provisioning state needed to remain a node. Configuration normally
adds one or more AppKeys and may add more NetKeys. These post-
provisioning keys are the high-value secrets in a device dump.

## Requirement matrix

| Item | Before provisioning | After provisioning | Secret? |
|---|---|---|---|
| Device UUID | Mandatory, unique, lifetime-stable | Remains the same | No |
| Provisionee P-256 key pair | Mandatory for an attempt, but may be ephemeral | Need not be retained if ephemeral | Private half only |
| Static OOB value | Optional capability | May remain for reprovisioning | Yes/auth credential |
| Static OOB P-256 pair | Optional capability | May remain for reprovisioning | Private half only |
| Device certificate | Optional v1.1 feature | May remain | Certificate is public |
| Certificate private key | Required if certificate public key is used | Usually lifetime-stable | Yes |
| Device Key (DevKey) | Does not yet exist | One per node instance | Yes |
| Primary NetKey | Does not yet exist | At least one | Yes |
| AppKeys | Do not yet exist | Added during configuration as needed | Yes |

The first P-256 row is a protocol-time requirement, not necessarily a
factory-stored secret. The last two rows are persistent operational
secrets after provisioning.

## Device UUID

Mesh Protocol v1.1 section 3.11.3 says every device **shall** be
assigned a unique 128-bit Device UUID and that it **shall not change**
throughout the lifetime of the physical or software product. The UUID
is a mandatory field in the Unprovisioned Device beacon (section
3.10.2, table 3.77).

Consequences:

- The UUID must survive power cycles, firmware updates, and a return
  to the unprovisioned state.
- It is an identifier, not a credential. An unprovisioned device
  broadcasts it, so confidentiality must not be assumed.
- Uniqueness can be implemented with factory-programmed NVM, a
  vendor-derived identifier, or another compliant stable mechanism.
  The specification does not require that its storage be secret.

Normative source:

- [Mesh Protocol v1.1, sections 3.10.2 and 3.11.3][mesh]

## Provisioning ECDH keys

Provisioning uses P-256 ECDH. The provisioner sends an ephemeral
public key generated on demand. For the provisionee there are two
valid designs:

1. If its public key is not available out of band, the provisionee
   generates an ephemeral key pair and sends the public key in band.
2. If its public key is available out of band, it can use a static
   key pair whose public half was delivered through another channel.

The explanatory SIG security paper states these alternatives
explicitly. Therefore Bluetooth Mesh does not generally mandate a
factory-stored P-256 private key. A device that only supports in-band
public-key exchange can generate a fresh pair for each attempt and
discard it when the attempt ends.

A public key is not secret. The private key must be protected for as
long as it is usable. A static/OOB private key is a persistent factory
secret; an ephemeral private key is only transient RAM state.

Sources:

- [Mesh Security Overview v2, sections 3.4 and 9.3][security]
- [Remote Provisioning technical overview][remote]

## Static OOB authentication

Static OOB is an optional authentication capability. The static value
is binary and may be freshly generated or factory-programmed and made
available by an external channel such as NFC or QR code. It is mixed
into the provisioning confirmation exchange to help resist a
man-in-the-middle attack; it is not a Mesh network key.

The SIG recommends using all 256 available bits of entropy when a
factory static OOB value is used. It also says authentication data
must not be reused in another security procedure, such as Bluetooth
LE pairing. The older predictable-AuthValue security notice explains
that fixed, predictable, or low-entropy values allow offline brute
force and a later MITM attack.

Important nuance: a QR-printed value is intentionally disclosed to a
person/scanner with physical access. It is still a credential and
should be unique per device, but it does not have the same physical
confidentiality boundary as a private key in secure hardware.

Sources:

- [Mesh Security Overview v2, sections 3.5 and 9.4][security]
- [Bluetooth SIG predictable AuthValue notice][authnotice]

## Certificate-based provisioning

Certificate-based provisioning is an **optional** Mesh Protocol v1.1
feature. It provides a verifiable binding between:

- the Device UUID and manufacturer/company identity; and
- a static public key held by that device.

The X.509 device certificate contains the Device UUID and static
public key and is CA-signed. The certificate and public key are public
data. The corresponding private key is the secret that proves control
of that certified public key and must be present or otherwise usable
by the provisionee.

The certificate can be stored on the device as a read-only
provisioning record, or a provisioning record/OOB medium can provide a
URI through which the provisioner retrieves it. Intermediate
certificates may also be stored as provisioning records. Thus the
certificate itself is not necessarily device-resident, but the
matching private key must be available to the device.

If a device has a retrievable device certificate, the normative
protocol requires support for provisioning records and the relevant
OOB-information bits. This is conditional on implementing that
optional feature, not a requirement for all Mesh devices.

Sources:

- [Certificate-Based Provisioning technical overview][cert]
- [Mesh Protocol v1.1, sections 3.10.2, 5.5, and 5.6][mesh]
- [Mesh Security Overview v2, section 3.8][security]

## Device Key

The Device Key does not exist as a factory credential. During
successful provisioning, both sides independently derive it from the
ECDH shared secret and the ProvisioningSalt:

`DevKey = k1(ECDHSecret, ProvisioningSalt, "prdk")`

It is never sent over the air. It is known only to the node, the
Provisioner, and an authorized Configuration Manager. It protects
node-specific configuration traffic and the distribution of other
keys. The SIG calls it one of the most security-critical items a node
holds.

Every provisioned node therefore must retain its active DevKey. Mesh
v1.1 may refresh it without full reprovisioning. A node-removal
procedure requires deletion of all stored credentials, security
material, DevKey, and provisioning data. That deletion does not erase
the lifetime Device UUID or necessarily erase optional factory OOB or
certificate credentials, which may be needed for reprovisioning.

Sources:

- [Mesh Protocol v1.1, sections 3.9.6.1 and 3.11.7][mesh]
- [Mesh Security Overview v2, sections 3.6, 5.2, and 9.3][security]

## Typical storage before and after provisioning

### Generic device before provisioning

Mandatory or operationally necessary:

- immutable, unique Device UUID (public);
- firmware and device composition/capabilities (public metadata);
- a CSPRNG and P-256 implementation able to create an ephemeral key
  pair for the provisioning attempt.

Optional, depending on advertised capabilities:

- per-device static OOB authentication value;
- static P-256 private/public provisioning key pair;
- certificate private key;
- device certificate, intermediate certificates, or certificate URI
  in provisioning records.

There is no DevKey, NetKey, AppKey, mesh unicast address, or IV Index
for an unprovisioned device.

### Immediately after provisioning

The node normally retains in nonvolatile storage:

- derived DevKey;
- primary NetKey and its key index/refresh state;
- allocated unicast address range;
- IV Index and IV Update state;
- sequence-number state needed to avoid nonce reuse;
- related protocol state needed to operate and resist replay.

Only DevKey and NetKey in that list are secrets. Address, key index,
IV Index, and counters are security-relevant state but not secret.
The provisioning data PDU supplies the NetKey, key index, flags,
IV Index, and unicast address; the DevKey is independently derived.

### After configuration

A normally configured application node additionally stores:

- one or more AppKeys;
- possibly additional subnet NetKeys;
- model-to-AppKey bindings, publication/subscription state, and other
  configuration data.

AppKeys and all NetKeys are secrets. Derived network encryption,
privacy, identity, beacon, and private-beacon keys are also secret
security material, whether cached or recomputed from a NetKey.

The Bluetooth SIG implementation guide says Mesh keys should be kept
persistently in on-chip NVM across power cycles, resets, and firmware
updates. Key Refresh can temporarily require old and new key sets.

Sources:

- [Mesh Protocol v1.1, provisioning and key sections][mesh]
- [Bluetooth Mesh Networking primer, sections 4.15-4.17][primer]
- [Choosing a Bluetooth Mesh Solution, section 3.3][solution]

## Firmware and memory-extraction implications

The result depends on what was extracted:

1. **Generic firmware image only.** Correct per-device secrets should
   not be embedded in a common firmware binary. Extracting that image
   may reveal algorithms, public certificates, or public identifiers,
   but need not reveal a device's unique secrets.
2. **Factory-programmed data/NVM dump.** This can reveal a static OOB
   value and any static/certificate private key if stored in readable
   flash. Those credentials can enable impersonation or MITM against
   future provisioning, depending on the provisioner's policy.
3. **Dump of a provisioned node.** Plaintext NVM can expose DevKey,
   NetKeys, AppKeys, and current/old key-refresh material. NetKey
   compromise grants subnet-level network access; AppKey compromise
   grants access to its application security domain; DevKey compromise
   exposes and enables node-specific configuration traffic, including
   later key distribution to that node.
4. **Secure-element/key-vault design.** A firmware or ordinary flash
   dump may contain handles and public data but not extractable private
   keys. The SIG specifically recommends dedicated hardware storage
   for node credentials, certificate keys, OOB private keys, and
   factory static OOB data.

The specification's "trash-can attack" defense is often
misinterpreted. Mesh supports removing a node and refreshing NetKeys
and AppKeys so a discarded node's old credentials no longer authorize
network use. It does not make plaintext secrets physically
unextractable. Secure storage, debug-port controls, authenticated boot
and update, per-device credentials, lifecycle erasure, and network
key refresh after suspected extraction are implementation/deployment
responsibilities.

Blast radius matters:

- Device UUID/certificate/public key disclosure is not secret loss.
- A per-device OOB/private key compromises that device's onboarding.
- A DevKey compromises management of that one node.
- An AppKey compromises its application domain across all holders.
- A NetKey compromises the subnet across all holders.
- A common factory OOB value or private key compiled into every unit
  turns one firmware extraction into a fleet-wide problem.

The last point is a security inference from the Mesh key model and
not a separate SIG protocol mandate. Per-device values and hardware
key isolation sharply reduce that failure mode.

Sources:

- [Mesh Security Overview v2, sections 9.1-9.4][security]
- [Mesh Protocol v1.1, security model and node removal][mesh]
- [Mesh Device Firmware Update Model, security considerations][dfu]

## Mandatory versus optional, in one sentence

For every physical/software Mesh product, the persistent pre-
provisioning requirement is a lifetime-stable Device UUID, which is
not secret; persistent factory secrets are feature-dependent, while
a provisioned node necessarily acquires persistent operational secrets
(DevKey and at least one NetKey) and normally later receives AppKeys.

## Sources

[mesh]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html

[security]: https://www.bluetooth.com/wp-content/uploads/2025/10/MeshSecurityOverview_INFO_v2.pdf

[cert]: https://www.bluetooth.com/bluetooth-certificate-based-provisioning-a-technical-overview/

[remote]: https://www.bluetooth.com/mesh-remote-provisioning/

[authnotice]: https://www.bluetooth.com/learn-about-bluetooth/key-attributes/bluetooth-security/predicatable-authvalue/

[primer]: https://www.bluetooth.com/bluetooth-mesh-networking-primer/

[solution]: https://www.bluetooth.com/wp-content/uploads/Files/Marketing/mesh-downloads/Choosing-A-Bluetooth-Mesh-Solution.pdf

[dfu]: https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/DFU/out/en/index-en.html
