# Home Assistant runtime key and BLE proxy flow

Scope: read-only investigation of
`/local/src/old/ha-bluetooth-mesh`. The Home Assistant integration imports
the vendored `custom_components/bluetooth_mesh/btmesh` package. The relevant
vendored files are byte-for-byte identical to the corresponding files under
`src/btmesh`, so the latter are cited for the mesh protocol implementation.

## Conclusion

This integration does not learn keys from BLE advertisements or mesh
traffic. It is a consumer of an already-complete ThingOS `.connect` export.
That export supplies the NetKey, all exported AppKeys, node primary unicast
addresses and element/model layout, model-to-AppKey bindings, and each
node's DeviceKey.

The configured NetKey is used first as a discovery selector: `k3(NetKey)`
produces the eight-byte Network ID that provisioned proxy nodes advertise.
After connecting, the same NetKey derives the NID, network encryption key,
and privacy key needed to authenticate, decrypt, and deobfuscate network
PDUs. A model-bound AppKey then decrypts the upper transport/access layer.
The node/element addresses in the export tell Home Assistant which unicast
to target; the source address in a decrypted reply is matched against that
target. DeviceKeys are separate, per-node credentials used for Config Server
diagnostics such as Relay Get and Composition Data Get.

Therefore the key-acquisition obstacle absolutely still applies to a
pre-existing vendor mesh. In this particular product family the vendor app's
`.connect` export is the acquisition mechanism, so the obstacle is solved by
cooperative export rather than cryptanalysis. If an installed vendor mesh
cannot export or otherwise reveal its NetKey, AppKey(s), per-node DeviceKeys,
addresses, and bindings, this integration cannot join it. Seeing its 0x1828
advertisement reveals only `k3(NetKey)`, not the NetKey; the AppKey and
DeviceKeys are not advertised; encrypted traffic does not fill those gaps.
The practical alternatives are vendor-side key extraction/export, or
factory-resetting and reprovisioning nodes onto keys controlled by Home
Assistant. The latter abandons transparent coexistence and, for the observed
Hafele/ThingOS device, may also lose vendor activation/composition behavior.

## 1. Configuration is a credential-bearing network export

The config flow accepts the literal contents of a `.connect` JSON file,
parses it for validation, and then stores the raw JSON in the config entry
(`config_flow.py:51-69`, `94-118`). It does not provision a network or derive
credentials at setup time. Reconfigure replaces that stored export, including
when a node is added or a key is refreshed (`config_flow.py:127-152`).

At entry setup, `MeshCoordinator` reparses
`entry.data[CONF_CONNECT_JSON]` into a `Network`
(`coordinator.py:199-204`). Setup deliberately succeeds even when no proxy is
currently reachable; an initial probe and later retries run in the background
(`__init__.py:24-49`, `coordinator.py:454-483`).

`Network.from_connect` requires and parses:

- `netKeys[0].key` and index;
- every `appKeys[]` key and index;
- the nodes list;
- each usable node's primary `unicastAddress`, `deviceKey`, company ID,
  elements, models, bindings, and subscriptions.

See `network_model.py:293-354`, `382-395`, `499-518`, and `540-560`.
Element unicast addresses are reconstructed as primary unicast plus element
index (`network_model.py:521-537`). A malformed node may be skipped, but if no
usable node remains the import fails (`network_model.py:310-341`).

The network identifier is the exported mesh UUID when present, otherwise
`k3(NetKey)` as a stable subnet identity (`network_model.py:205-213`). This is
also used to prevent a reconfigure from silently changing an entry to a
different mesh (`config_flow.py:145-152`).

## 2. How the NetKey identifies a proxy advertisement

Provisioned proxy advertisements use service UUID 0x1828. Service data type
0x00 contains an eight-byte Network ID equal to `k3(NetKey)`; type 0x01
contains Node Identity hash plus random (`bearer.py:301-315`).

`find_proxy_address` computes `network_id = k3(net_key)`, scans Home
Assistant's complete Bluetooth discovery snapshot, parses 0x1828 service
data, requires identification type Network ID and an exact parameter match,
rejects stale advertisements, and returns the first match marked connectable
(`mesh_transport.py:118-188`). Thus this runtime path does not match Node
Identity advertisements; diagnostics can display those, but identifies them
only as `node-identity` (`mesh_transport.py:191-225`).

The coordinator uses the imported NetKey in both discovery paths:

- a push callback computes `k3(net_key)` and triggers only for matching
  0x1828 Network-ID advertisements
  (`mesh_transport.py:334-359`, `coordinator.py:479-483`);
- connection setup again calls `find_proxy_address` with the imported NetKey
  (`coordinator.py:644-648`).

The match locates any GATT Proxy node for the subnet, not necessarily the
eventual command target. One proxy tunnel can relay traffic to the rest of
the mesh. Once an address is selected, Home Assistant resolves a connectable
BLE device and `establish_connection` opens GATT; the client is wrapped in a
non-provisioning `GattBearer` (`mesh_transport.py:297-331`).

An important operational detail is that a mesh node has one GATT proxy slot
and stops advertising 0x1828 while that slot is occupied. The coordinator
therefore keeps/reuses a connection, applies stale-advertisement exceptions
after its own link ends, and backs off repeated failures
(`mesh_transport.py:67-79`, `coordinator.py:599-648`, `747-760`). This is a
connection-availability issue, not evidence of missing or changed keys.

## 3. What the NetKey does after GATT connection

`MeshController` constructs a `MeshNode` with the imported NetKey, selected
AppKey, IV Index, source address, and sequence cursor
(`controller.py:109-140`). `NetworkContext.__post_init__` applies `k2` to the
NetKey to derive three network-layer values: NID, EncryptionKey, and
PrivacyKey (`network.py:44-64`).

On receive, network decoding:

1. rejects a mismatching NID or IVI;
2. uses the derived PrivacyKey to deobfuscate CTL/TTL, SEQ, and SRC;
3. constructs the network nonce from those values and the IV Index;
4. authenticates/decrypts destination plus lower-transport PDU with the
   derived EncryptionKey and NetMIC.

See `network.py:135-169`. A wrong NetKey generally fails before meaningful
source, destination, or access data can be recovered. The receive wrapper
drops such traffic as foreign/invalid rather than allowing it into access
handling (`node.py:250-269`).

The IV Index is also required for network and upper-transport nonces. The
coordinator starts from the export/persisted value, authenticates Secure
Network Beacons with a beacon key derived from the NetKey, and can adopt a
newer announced IV Index (`beacon.py:51-84`,
`coordinator.py:1122-1170`). This update mechanism still presupposes the
correct NetKey; an unauthenticated or foreign beacon is ignored.

Proxy configuration is itself sent as NetKey-protected network traffic. On
each new proxy link, the controller sets an accept-list filter and adds Home
Assistant's own mesh source address. A proxy connection initially forwards
no inbound traffic, so without this step Status replies would not reach the
client (`controller.py:234-264`).

## 4. What the AppKey does

The coordinator does not blindly use the export's first AppKey. It examines
the imported model `bind` indexes for the model IDs that the integration
controls, chooses an exported AppKey having the most relevant bound index,
and warns if bindings reference no key present in the export
(`network_model.py:217-260`, `coordinator.py:338-365`). The selected key is
passed to `MeshController` on every connection (`coordinator.py:696-700`).

`MeshNode` derives the six-bit AID with `k4(AppKey)`
(`node.py:102-116`). Ordinary application messages are upper-transport
encrypted with that AppKey, use AKF=1 and carry its AID
(`node.py:154-209`). On receive, an AKF=1 packet whose AID does not match is
dropped; a matching AID selects the configured AppKey, after which AES-CCM
authentication/decryption yields the access opcode and parameters
(`node.py:416-468`). The AID is only a short key identifier, not key material,
and collisions are possible; successful AES-CCM authentication is the actual
proof of the key.

This is why knowing only the NetKey is insufficient for light control. It
would expose the lower transport PDU, but not authenticate/decrypt an
AppKey-protected access message. Equally, having an AppKey without the NetKey
does not expose the network header and upper transport data.

## 5. How nodes are located and replies are associated

There are two distinct meanings of "locate":

- BLE discovery locates a proxy for the subnet using `k3(NetKey)`.
- Mesh addressing locates a logical node/model using the static node,
  element, and composition records imported from `.connect`.

The integration does not discover its entity inventory by decrypting ambient
traffic. The light platform enumerates `coordinator.network.nodes`, determines
the lighting output elements from their imported models, and creates entities
from those records (`light.py:151-202`). For each entity it stores the node or
element unicast and, for a single-output/multi-element node, separately finds
the element that hosts each target model (`light.py:222-315`). This matters
because an opcode sent to an element that does not host the model is silently
ignored.

Commands are addressed directly to that imported unicast. For example,
Generic OnOff Set sends to the supplied unicast and expects Generic OnOff
Status (`controller.py:395-409`). The received network PDU reveals `src` only
after NetKey deobfuscation/authentication (`network.py:147-168`). After AppKey
decryption and access parsing, a request is completed only when response
source equals the requested destination and opcode equals the expected Status
opcode (`node.py:470-477`, `486-529`). There is no independent runtime lookup
from BLE address to node unicast; the BLE peer is merely the chosen proxy.

## 6. DeviceKeys and diagnostic node reachability

Each node's DeviceKey from the export is registered under its primary unicast
when a controller is created (`controller.py:141-149`). AKF=0 Config/Foundation
traffic uses the DeviceKey registered for the destination, rather than the
shared AppKey (`node.py:143-180`). Incoming device-keyed replies select the
key by source address (`node.py:454-468`).

The integration uses this for Relay Get and Composition Data Get. Those
requests address a known imported node unicast and reach its Config Server
without relying on lighting-model AppKey bindings (`controller.py:330-393`).
This makes them a useful diagnostic distinction between transport failure and
a model silently rejecting an otherwise delivered application command. It
does not eliminate key acquisition: a correct per-node DeviceKey is still
required, and the runtime obtains it from `.connect`.

## 7. Pre-existing vendor mesh and the acquisition obstacle

The architecture was explicitly changed from "own network" provisioning to
riding the vendor network. Hardware testing found that vendor activation
produced the full lighting model composition bound to the app's AppKey; the
working solution was to export NetKey, AppKey, and addresses and send standard
SIG messages on that existing mesh
(`docs/plans/2026-07-19-phase0-report.md:18-44`). The README describes the same
coexistence contract and warns that the `.connect` file contains secret mesh
keys (`README.md:153-184`). Provisioning still occurs in the vendor app
(`docs/forum-post-en.md:70-77`).

Consequences for an arbitrary already-installed vendor mesh:

1. **A compatible export solves the problem.** Importing current keys,
   bindings, addresses, composition, and DeviceKeys lets Home Assistant join
   without changing the mesh. This is what the code implements.
2. **An advert is only a verifier/selector.** Given a candidate NetKey, HA can
   calculate `k3` and check whether a proxy advert matches. The advertised
   eight-byte Network ID does not supply the 128-bit NetKey. The implementation
   contains no reverse/recovery path: discovery APIs require `net_key` as an
   input (`mesh_transport.py:132-164`, `334-346`).
3. **Passive network capture is not key acquisition.** Network decoding needs
   NetKey-derived privacy/encryption keys and an authenticated MIC; access
   decoding then needs the correct AppKey. Device-key traffic needs the
   node-specific DeviceKey. Captured ciphertext or AID/NID fields allow testing
   a key already obtained elsewhere, not deriving the missing secrets.
4. **Partial credentials are insufficient.** NetKey alone can find a proxy and
   open/decrypt the network layer, but cannot issue accepted application
   commands without the bound AppKey and addresses. AppKey alone cannot locate
   the proxy or unwrap the network layer. Missing DeviceKeys specifically
   disable device-keyed configuration/diagnostics. Missing composition/bindings
   makes addresses and the correct AppKey uncertain and causes silent drops.
5. **No export means the original obstacle remains.** Keys must be obtained
   through a vendor-supported export or extracted from the vendor
   provisioner's storage. Otherwise the standards-compliant fallback is to
   factory-reset/reprovision devices into a new network whose keys HA owns.
   The early design explicitly described that option as avoiding extraction
   but forfeiting vendor-app coexistence
   (`docs/plans/2026-07-18-btmesh-design.md:23-33`). For this particular vendor,
   the later hardware result warns that own-network provisioning left the lamp
   unactivated with a minimal composition
   (`docs/plans/2026-07-19-phase0-report.md:18-44`).

In short: the integration is evidence that standard mesh control works once
the complete credential/configuration set is available. It is not evidence
that a pre-existing mesh can be joined from radio observations alone.
