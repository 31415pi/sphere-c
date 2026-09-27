# Sphere C key provisioning

## Short answer

Bluetooth Mesh does not require a factory-installed private key.
In the normal provisioning path, the phone and device generate
temporary P-256 key pairs and use ECDH to derive the same shared
secret. They independently derive a session key and a unique
symmetric DeviceKey from that secret.

The phone chooses the mesh NetKey and AppKey. It sends the NetKey in
an encrypted provisioning message. After provisioning, it sends the
AppKey in a DeviceKey-protected configuration message and binds that
AppKey to the device's lighting models.

## Protocol sequence

1. The factory-fresh device advertises its public Device UUID and
   provisioning capabilities.
2. The phone and device exchange P-256 public keys. In the baseline
   flow, both private keys are freshly generated for this attempt.
3. Both calculate the same ECDH secret from their own private key and
   the peer's public key.
4. A confirmation exchange authenticates the transcript using the
   chosen OOB method and fresh random values.
5. Both independently derive `SessionKey`, `SessionNonce`, and
   `DevKey` from the ECDH secret and provisioning salt.
6. The phone encrypts and authenticates this provisioning data with
   AES-CCM under the session key:

   ```text
   NetKey || NetKey index || flags || IV Index || unicast address
   ```

7. The device decrypts the data and becomes a mesh node. The ECDH
   key pair and session material can then be discarded.
8. The phone configures the node with DeviceKey-protected messages:
   read composition, add AppKey, bind models, and set publication,
   subscriptions, and other parameters.

The DeviceKey is never sent over the provisioning bearer:

```text
DevKey = k1(ECDHSecret, ProvisioningSalt, "prdk")
```

Both sides arrive at the same value because their ECDH result and
provisioning transcript are identical.

## What is stored

Before provisioning, every device needs a stable Device UUID, but
that UUID is public. A normal in-band provisioning implementation
also needs a secure random generator and P-256 implementation, but
does not need a persistent asymmetric secret.

After provisioning, the device retains:

- its derived, node-specific DeviceKey;
- the shared NetKey and later the AppKey;
- its unicast address and model configuration;
- IV, sequence-number, replay, and key-refresh state.

A factory reset should erase this provisioned state. The permanent
Device UUID and any optional factory authentication material remain
so the device can be provisioned again.

## Optional factory secrets

A manufacturer may choose stronger provisioning authentication:

- a per-device Static OOB value printed as a QR code or stored in
  NFC/flash;
- a static P-256 key pair whose public key is obtained out of band;
- certificate-based provisioning, where a certificate binds the
  Device UUID to a static public key.

In the latter two cases, the corresponding private key is a real
factory secret and should be in protected NVM or a secure element.
Static OOB is a factory secret but is not an asymmetric private key.

These mechanisms are optional. With in-band public-key exchange and
No OOB authentication, there need be no permanent provisioning secret.
That simple mode is vulnerable to an active man-in-the-middle during
commissioning.

## Olight and Thingclips evidence

Static analysis of Olight 3.2.5 shows this likely Sphere C path:

1. The authenticated Thingclips cloud create/list API returns a
   `SigMeshBean` containing the home's NetKey and AppKey.
2. The app generates a fresh local `secp256r1` provisioner key pair.
3. The selected public-key mode is in-band.
4. The provisioning coordinator supplies an empty authentication
   value, which becomes 16 zero bytes.
5. The app derives the DeviceKey locally using the standard `prdk`
   calculation.
6. It sends the NetKey during provisioning and adds the AppKey during
   configuration.
7. It uploads the derived DeviceKey and node metadata to the
   authenticated mesh-join API.

No Sphere-specific Static OOB value, fixed Device UUID, or factory
P-256 private key was found in the app. The code therefore strongly
suggests ephemeral ECDH plus No OOB for Sphere C, with no permanent
device private key involved in SIG Mesh provisioning.

This is not yet proven for the physical Sphere C. The supplied
capture begins after provisioning and contains neither the
Capabilities nor Start PDU. A factory-reset provisioning capture
would reveal the advertised and selected authentication modes.

The device firmware could still contain unrelated per-device Tuya or
Olight manufacturing credentials. Such credentials are not required
by the SIG Mesh flow, and no evidence currently ties one to Sphere C
provisioning.

## Sources and detailed evidence

- Bluetooth SIG Mesh Protocol:
  https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/MshPRT_v1.1/out/en/index-en.html
- Bluetooth SIG Mesh Security Overview:
  https://www.bluetooth.com/wp-content/uploads/2025/10/MeshSecurityOverview_INFO_v2.pdf
- Bluetooth SIG provisioning overview:
  https://www.bluetooth.com/mesh-remote-provisioning/
- `docs/research/mesh/mesh_provisioning.md`
- `docs/research/app/device_secrets.md`
- `docs/research/app/tuya_provisioning.md`
