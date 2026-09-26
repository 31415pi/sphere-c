# Bluetooth Mesh factory-fresh provisioning

## Scope and sources

This note covers Bluetooth Mesh provisioning, with emphasis on a
factory-fresh device and whether it must contain a permanent private
key. It uses Bluetooth SIG material only.

Primary normative source:

- Bluetooth SIG, *Mesh Protocol Specification*, version 1.1,
  especially Sections 3.9.6.1 and 5.4. The complete official HTML is
  available at
  <https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html>.
- Bluetooth SIG lists version 1.1.1 as the current adopted revision at
  <https://www.bluetooth.com/specifications/specs/mesh-protocol-1-1-1/>.
  The provisioning rules discussed here are the version 1.1 rules.

Useful official explanatory sources:

- Bluetooth SIG, *Mesh Security Overview*, version 2, Section 3:
  <https://www.bluetooth.com/wp-content/uploads/2025/10/MeshSecurityOverview_INFO_v2.pdf>.
- Bluetooth SIG, *Certificate-Based Provisioning: A Technical
  Overview*:
  <https://www.bluetooth.com/bluetooth-certificate-based-provisioning-a-technical-overview/>.
- Bluetooth SIG, *Mesh Remote Provisioning*, Section 1.1.3:
  <https://www.bluetooth.com/mesh-remote-provisioning/>.

The specification is normative. The informational publications are
used only to clarify its operation and security consequences.

## Short answer: is a permanent device private key required?

**No, not for ordinary Bluetooth Mesh provisioning.**

In the normal, in-band public-key path, the Provisioner and
Provisionee generate P-256 key pairs for this provisioning attempt.
They exchange public keys, compute the same ECDH secret, and the
specification then requires each party to delete the private-public
key pair generated in that step. A factory-fresh device therefore can
start without a permanent asymmetric identity private key.

This conclusion follows directly from Mesh Protocol Sections 5.4.2.3
and 5.4.2.5:

- when the Provisionee public key is not available OOB, both parties
  generate a new key pair;
- `ECDHSecret = P256(private key, peer public key)`;
- after computing `ECDHSecret`, each deletes the key pair generated in
  that step;
- the durable node-specific secret is instead the symmetric
  `DevKey`, derived during provisioning.

A permanent or factory-installed private key is an **optional design
choice** when using an out-of-band public key. Certificate-based
provisioning is the standardized Mesh 1.1 example: a device
certificate binds the Device UUID and identity information to the
device's static public key. The device must then retain and protect
the corresponding private key if it is expected to reuse that OOB
identity for later provisioning. The official security overview says
that dedicated storage can protect the private key for public-key OOB
provisioning (Section 9.3). That is not a baseline requirement for all
Mesh devices.

Likewise, a factory-programmed Static OOB authentication secret is
optional. It is an authentication value, not an ECDH private key.

## Roles and transport

An unprovisioned device becomes the **Provisionee** for the procedure.
The **Provisioner** initiates it and supplies the material that lets
the device become a node. The same provisioning protocol can run over
PB-ADV, PB-GATT, or PB-Remote. The bearer changes transport, not the
cryptographic provisioning sequence. See Mesh Protocol Sections 5,
5.2, and 5.4.

The protocol is independent from LE pairing and bonding. It creates
Mesh credentials, not a Bluetooth LE bond.

## Five protocol phases

Mesh Protocol Section 5.4.2 defines five steps.

### 1. Beaconing

The unprovisioned device advertises availability. Discovery identifies
it with a 128-bit Device UUID and may advertise that OOB information
is available. For PB-GATT, the device advertises the Mesh Provisioning
Service. OOB data itself is not sent in the beacon.

The Device UUID is an identifier, not a secret or private key. The
Bluetooth SIG security overview, Section 3.2, says it may be assigned
during manufacturing or randomly generated on first power-up. A
certificate that embeds the UUID naturally requires a stable,
production-known UUID.

### 2. Invitation and capability selection

After opening a provisioning bearer, the Provisioner sends
`Provisioning Invite`. The Provisionee replies with
`Provisioning Capabilities`, including:

- number of elements;
- supported provisioning algorithms;
- whether an OOB public key is available;
- whether Static OOB data is available or authenticated provisioning
  is mandatory;
- supported Output OOB and Input OOB actions and sizes.

The Provisioner selects compatible choices in `Provisioning Start`:
algorithm, in-band or OOB public key, authentication method, action,
and size. See Mesh Protocol Sections 5.4.1.1 through 5.4.1.3.

Mesh 1.1 defines two algorithm suites:

- `BTM_ECDH_P256_CMAC_AES128_AES_CCM`;
- `BTM_ECDH_P256_HMAC_SHA256_AES_CCM`.

The Provisionee is required to support the newer HMAC-SHA-256 suite.
The older suite remains for backward compatibility. See Mesh Protocol
Tables 5.21 and 5.29 and the Bluetooth SIG security overview,
Section 3.3.

### 3. Public-key exchange and ECDH

Both suites use elliptic-curve Diffie-Hellman on the NIST P-256 curve.
The Provisioner's key pair is freshly generated. There are two cases
for the Provisionee.

**No OOB public key.** Both parties generate a new key pair and send
their 64-octet public keys, X and Y, in `Provisioning Public Key`
PDUs. Both validate the peer key, then calculate:

```text
ECDHSecret = P256(own private key, peer public key)
```

The public keys are observable on the provisioning bearer, but the
shared secret is not transmitted. Both sides independently obtain the
same 256-bit result.

**OOB Provisionee public key.** The Provisioner obtains the device's
public key through an OOB channel and indicates that selection in
`Provisioning Start`. Only the Provisioner's newly generated public
key is sent in-band. A certificate, NFC record, or QR-mediated value
can supply the Provisionee public key. Certificate-based provisioning
adds a verifiable association between that public key, Device UUID,
and device identity.

After `ECDHSecret` is calculated, Mesh Protocol Section 5.4.2.3 says
that each party deletes the private-public key pair that was generated
in this step. Thus generated provisioning keys are ephemeral. This
deletion sentence does not turn an optional, pre-existing OOB device
key into an ephemeral key; such a device needs its corresponding
private key to use that OOB public key.

An in-band public-key exchange by itself does not authenticate device
identity and is vulnerable to active man-in-the-middle substitution.
The Bluetooth SIG security overview, Tables 3.1 and 3.2, explains how
OOB public keys and authentication data change that risk.

### 4. Authentication

Authentication binds an `AuthValue` to the transcript and to fresh
random values by a commit-then-reveal exchange. The exchanged
confirmation values commit to the random number before either side
reveals it. After both random values are sent, each side recomputes and
checks the peer's confirmation. See Mesh Protocol Sections 5.4.2.4.1
and 5.4.2.4.2.

`ConfirmationInputs` is the concatenation of the values of:

```text
Invite || Capabilities || Start ||
ProvisionerPublicKey || DevicePublicKey
```

The selected suite determines the confirmation construction:

```text
CMAC suite:
  ConfirmationSalt = s1(ConfirmationInputs)
  ConfirmationKey =
      k1(ECDHSecret, ConfirmationSalt, "prck")
  Confirmation = AES-CMAC_ConfirmationKey(Random || AuthValue)

HMAC-SHA-256 suite:
  ConfirmationSalt = s2(ConfirmationInputs)
  ConfirmationKey =
      k5(ECDHSecret || AuthValue,
         ConfirmationSalt, "prck256")
  Confirmation = HMAC-SHA-256_ConfirmationKey(Random)
```

The CMAC suite uses 128-bit random and confirmation values. The HMAC
suite uses 256-bit values. `s1` is AES-CMAC-based; `s2` is
HMAC-SHA-256-based. See Mesh Protocol Sections 3.9.2 and 5.4.2.4.1.

The authentication choices are:

- **No OOB:** `AuthValue` is zero. This provides no device identity
  authentication. With an in-band public key it does not defend
  against active MITM.
- **Static OOB:** a shared binary, numeric, or alphanumeric value is
  obtained outside the protocol. It may be factory programmed and
  conveyed by QR, NFC, packaging, or another vendor mechanism.
- **Output OOB:** the Provisionee generates and outputs a random value
  by blink, beep, vibration, numeric display, or alphanumeric display;
  the user enters it into the Provisioner.
- **Input OOB:** the Provisioner generates and displays a random value;
  the user enters it into the Provisionee by push, twist, numeric, or
  alphanumeric input. The Provisionee sends `Provisioning Input
  Complete` before the confirmation exchange.

Mesh 1.1 permits up to eight displayed or entered characters and up
to 32 octets of Static OOB data, depending on capabilities and suite.
More entropy gives stronger resistance to guessing. Mesh Security
Overview Sections 3.5 and 9.4 recommend maximizing available entropy.

The public-key OOB selection and authentication OOB selection are
independent. For example, a device can use an in-band ephemeral public
key plus Static OOB authentication, or an OOB public key plus No OOB
authentication. An authenticated OOB public key can itself prevent
public-key substitution, but authorization policy remains an
application/deployment concern.

### 5. Provisioning-data distribution

Only after authentication succeeds do both sides derive the session
material. For the algorithm flow described by Mesh Protocol Section
5.4.2.5:

```text
ProvisioningSalt =
    s1(ConfirmationSalt || RandomProvisioner || RandomDevice)

SessionKey =
    k1(ECDHSecret, ProvisioningSalt, "prsk")

SessionNonceFull =
    k1(ECDHSecret, ProvisioningSalt, "prsn")

SessionNonce = least-significant 13 octets(SessionNonceFull)
```

Although the HMAC suite uses `s2` for its 256-bit
`ConfirmationSalt`, Mesh Protocol Section 5.4.2.5 defines
`ProvisioningSalt` with `s1` and derives the 128-bit AES session
material with `k1`. The inputs include both fresh random values, so a
new successful run produces new session material.

The 25-byte plaintext is:

```text
Network Key (16) || Key Index (2) || Flags (1) ||
IV Index (4) || Unicast Address (2)
```

The Flags carry the Key Refresh and IV Update state. The address is
for the primary element; the Provisionee assigns consecutive unicast
addresses to its remaining elements.

The Provisioner encrypts and authenticates this plaintext with
AES-CCM using `SessionKey` and the 13-octet `SessionNonce`. The result
is 25 octets of encrypted provisioning data plus an 8-octet MIC in
the `Provisioning Data` PDU:

```text
(ciphertext, MIC) =
    AES-CCM_SessionKey(SessionNonce, ProvisioningData)
```

The Provisionee verifies the MIC and decrypts before installing the
NetKey, network state, and addresses. It then returns
`Provisioning Complete`; the bearer is closed, and it is now a node.

## DeviceKey derivation

The `DevKey` is not present in the encrypted provisioning plaintext
and is never sent over the bearer. Both parties derive it independently
from the ECDH secret and provisioning salt:

```text
DevKey = k1(ECDHSecret, ProvisioningSalt, "prdk")
```

This is specified in Mesh Protocol Section 3.9.6.1 and referenced by
Section 5.4.2.5. The Bluetooth SIG security overview, Section 3.6,
also states explicitly that both sides derive the device key and that
it is not transmitted over the air.

`DevKey` is a 128-bit, node-specific access-layer key. It is known to
the node, Provisioner, and Configuration Manager. It protects
configuration traffic between that node and a Configuration Manager
and is implicitly bound to every NetKey known by the node. It is not
the P-256 private key used to produce `ECDHSecret`.

## What persists

### On the newly provisioned node

The state needed after completion includes:

- the derived `DevKey`;
- the delivered `NetKey` and its NetKey Index;
- Key Refresh and IV Update state from the flags;
- the IV Index;
- the assigned unicast address range.

The node can later receive AppKeys through configuration messages
protected with its DevKey. AppKeys are not part of the provisioning
data PDU.

The specification's node-removal rule confirms this persistence model:
Mesh Protocol Section 3.11.7 requires removal to delete stored
security credentials, stored security material, the device key, and
the provisioning data, after which a provisioning-capable node becomes
unprovisioned.

### At the Provisioner or Configuration Manager

The manager must retain the network's NetKey and the node's DevKey,
plus the allocated address and related network database state. Mesh
Protocol Section 3.9.6 says the DevKey is known only to the node, the
Provisioner, and a Configuration Manager. This lets the manager send
node-specific configuration and distribute AppKeys securely.

### Normally transient

For baseline in-band public-key provisioning, these are per-session
values and are not operational Mesh credentials:

- P-256 provisioning private/public key pair;
- `ECDHSecret`;
- `ConfirmationSalt` and `ConfirmationKey`;
- `RandomProvisioner` and `RandomDevice`;
- `ProvisioningSalt`;
- `SessionKey` and `SessionNonce`.

The specification explicitly mandates deletion of the generated EC
key pair after ECDH. It does not give these other intermediates an
after-provisioning role. An implementation should erase transient
secrets as soon as they are no longer needed. The durable outputs are
the DevKey and installed network state, not the session derivation
chain.

### Optional factory-persistent items

Depending on the product's chosen OOB scheme, a device may also retain:

- a stable Device UUID;
- a Static OOB authentication value;
- an OOB P-256 private key and corresponding public key;
- a device certificate and possibly intermediate certificates or
  provisioning records.

These items enable stronger identification or authentication, but the
baseline protocol does not require them. The certificate technical
overview, Sections 1.2 and 2.2, describes a static key pair as something
an unprovisioned device *may* have and explains how the certificate
contains the static public key.

## Security interpretation for factory-fresh devices

There are three distinct concepts that should not be conflated:

1. **Device UUID:** a public discovery identifier. It can be generated
   on first boot unless a certificate or manufacturing workflow needs
   it fixed earlier.
2. **Provisioning P-256 private key:** ephemeral in the baseline
   in-band-public-key flow; optional and persistent only for an OOB
   public-key identity scheme.
3. **DevKey:** a durable symmetric node credential created from the
   one-time ECDH exchange during provisioning.

Consequently, a blank device can generate its EC key pair at first
provisioning, authenticate through an available OOB user interaction
or even No OOB, derive and store its DevKey, and discard the EC key
pair. There is no universal Bluetooth Mesh requirement to inject a
unique permanent asymmetric device secret at manufacture.

However, using both an in-band public key and No OOB authentication
is vulnerable to active MITM. Avoiding a factory private key does not
mean the commissioning path is automatically secure. Strong options
include high-entropy Static OOB data, suitable Input/Output OOB, or a
protected static private key with a verifiable OOB/certificate-bound
public key. The Bluetooth SIG security overview, Section 9.4, advises
using OOB public keys or OOB authentication data and warns that a
provisioner should not silently fall back when a device expected to
have an OOB public key cannot supply it.
