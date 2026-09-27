# Sphere C vendor model `0x07d0:0x0005`

## Result

`0x07d0:0x0005` is Tuya's **vendor client** model. The
Sphere C advertises it on element zero alongside vendor server
`0x07d0:0x0004`. The saved Olight 3.2.5 APK binds and addresses
`0x0004`, never `0x0005`, for the ordinary Tuya DP path. The
Sphere C-specific mini-app asks for properties through the native
SDK; it does not select a mesh model or demonstrate a distinct
`0x0005` feature.

No captured transaction establishes the Sphere using its local
vendor client to originate a message to another node. There is no
known safe *application* query addressed to `0x0005`. A
DeviceKey-protected Configuration Client query of its binding list
could inspect configuration state, but would not reveal its
implemented actions or prove that it uses them. The proven read-only
DP query targets server `0x0004`.

## Direct evidence

1. Three authenticated Composition Data page 0 results in
   `state/provision_run.log` (lines 647, 739, and 1007) report
   company `0x07d0`, product `0x0200`, and both vendor models on
   element zero. Elements one through five contain only Generic
   OnOff Server. `state/sphere_state.json` preserves the same
   composition. Composition reports model IDs, not their roles or
   registered opcode tables.
2. Tuya's [vendor model specification](https://developer.tuya.com/en/docs/iot-device-dev/tuya-sigmesh-device-vendor-model-access-standard?id=K9pikwhoo3gux)
   explicitly calls packed model `0x07D00004` the server and
   `0x07D00005` the client. The packed form corresponds to company
   ID `0x07d0` and 16-bit model ID `0x0004` or `0x0005` in the
   Sphere composition. The document assigns WRITE `0xc9d007`,
   WRITE_UNACK `0xcad007`, and READ `0xccd007` to client requests,
   DATA `0xcdd007` to server responses/reports, and marks STATUS
   `0xcbd007` reserved. These are Tuya protocol assignments, not
   observations of the Sphere's `0x0005` implementation.
3. The local APK disassembly at `/tmp/olight-dex.7xC3jV/` shows
   `com/thingclips/sdk/bluetooth/bbdpqbd.smali:459-506` building a
   model bind allow-list containing vendor `0x7d00004` and no
   `0x7d00005`. The active setup path binds AppKey 0 to the server,
   as traced in `docs/research/app/apk_post_provision_flow.md`.
   `src/sphere_mesh/provision.py` mirrors that server-only rule.
   This is evidence about Olight's and the local provisioner's
   binding behavior; it does not prove that binding the client is
   forbidden or impossible.
4. The APK's ordinary vendor read action
   `com/thingclips/sdk/bluetooth/ppbppqq.smali:5-73` specifies
   model `0x7d00004`, company `0x7d0`, request opcode `0xcc`, and
   expected reply `0xcd`. Its DP write encoder
   `pdpbqqp.smali:7-73` likewise uses `0x7d00004` and `0xc9`.
   A search for literal `0x7d00005` throughout the saved smali
   tree found no occurrence. This negative search is limited by
   obfuscation, computed IDs, and unexamined native code.
5. The authenticated local query in
   `docs/research/mesh/vendor_dp_live_query.md` sent
   `cc d0 07 01 02 02 66` and received `cd d0 07` DATA, including
   DP 102 (`66 04 07`). This establishes a usable server-side
   request and response route. Its record does not identify a
   separate local-client action or an opcode exclusive to `0x0005`.
6. The Sphere C mini-app bundle at
   `old/phone_external/files/Godzilla/MiniApp/9E6DD95017EDCABEA0673DC3A9AFAC12/Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js`
   contains the product DP schema and effect-picker property calls
   (`work_mode`, `colour_mode`). Its calls are at the property API
   level. The APK routes vendor DPs through `0x0004`; the bundle
   itself supplies no `0x0005` model binding, destination, opcode,
   or event handler. See `vendor_dp_inventory.md` for DP-level
   evidence.
7. The original Olight-network capture
   `old/captures/sphere-c.btsnoop` lacks its NetKey/AppKey, so its
   mesh access messages cannot be decoded. The local provisioning
   captures and later DP read demonstrate configuration and server
   traffic, but none is labeled as a `0x0005`-originated operation.

## Model roles and routing

Bluetooth Mesh model IDs are not carried in ordinary access
messages. An access message carries an opcode, source/destination,
parameters, and security context. The receiver's registered model
and AppKey binding determine which handler gets it. Consequently,
seeing `cd d0 07` in a capture does not by itself name a source
model. Tuya's table assigns DATA production to the server, and the
observed request to the Sphere server makes that the expected
route. The embedded client model could be used by firmware for
other Tuya interactions, but no saved evidence establishes one.

| Direction in Tuya protocol | Opcode on wire | Assigned action |
|---|---|---|
| Client to server | `c9 d0 07` | Acknowledged WRITE |
| Client to server | `ca d0 07` | WRITE_UNACK |
| Client to server | `cc d0 07` | READ |
| Server to client | `cd d0 07` | DATA response or report |
| Server status slot | `cb d0 07` | Reserved |

The app's parser dispatches DATA by command type after opcode
`0xcd`. Types `0x01` and `0x05` become ordinary
`ThingVendorModelStatus` DP records; other types route to
specialized status paths. This is opcode/parameter routing in the
APK, not proof that the Sphere's local `0x0005` model consumes
those reports. `vendor_dp_response_codec.md` documents the exact
record grammar.

## Binding expectation

The official app's provisioning allow-list includes only vendor
server `0x0004`, and the successful local provisioning run used the
same rule. Thus the operational expectation for normal Sphere DP
control is: bind the AppKey to `0x0004` on the primary element and
send client-originated READ/WRITE commands to that element address.
The external controller supplies the client behavior. Do not bind
`0x0005` merely because it appears in composition, and do not
address the Sphere's `0x0005` as if it were a second DP server.

Whether `0x0005` has a pre-existing AppKey binding, can be bound,
or has a publication/subscription configuration has not been
verified by a model-specific configuration read. The official app's
omission is strong workflow evidence, not a firmware capability
test. An inbound DATA report might also be handled by a broader
SDK listener rather than by a model that the app bound on the
Sphere.

## Safe query and remaining uncertainty

There is no documented `0x0005`-specific GET or read-only state
request. The known safe runtime query is the Tuya DP READ
`cc d0 07` to vendor **server** `0x0004`, already exercised on the
local device. For `0x0005`, the only useful nonmutating inspection
would be Configuration Vendor Model App Get (or a model
publication/subscription get, if supported) using the DeviceKey
against the primary element. Its answer would describe
configuration, not client semantics. No radio query was performed
for this note.

The strongest remaining hypothesis is that the Sphere firmware
includes Tuya's standard client model for common-stack parity or
for optional node-to-node features while the phone controls the
Sphere through its server. That is an inference. Assigning a
concrete product behavior, DP set, or status consumer to local
`0x0005` would require a decrypted device-originated transaction
or firmware handler evidence. The current artifacts do not supply
one.
