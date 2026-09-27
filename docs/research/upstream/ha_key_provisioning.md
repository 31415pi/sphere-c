# Home Assistant Bluetooth Mesh key provisioning

## Scope and conclusion

Repository inspected: `/local/src/old/ha-bluetooth-mesh`, `main` at
`a8f7bbbebaeba124d7d08cb16122055074f2ef46` (2026-09-24). The upstream
clone was read only.

The shipping Home Assistant integration does **not** discover an existing
NetKey, AppKey, or DeviceKey from Bluetooth devices, and it does **not**
create/provision its own mesh network. It joins a network already provisioned
by the ThingOS / Häfele Connect Mesh vendor app. The user exports that network
as a `.connect` JSON file and pastes the file contents into the integration.
That export is the sole source of the NetKey, AppKey(s), and per-node
DeviceKey(s).

The repository does contain a complete provisioning state machine and a
separate Phase-0 feasibility script. Those can create a fresh test network and
derive a DeviceKey during provisioning, but neither is wired into the Home
Assistant config flow or runtime. Current documentation and changelog call
provisioning from Home Assistant unfinished/unreachable.

## Credential source and import path

### User-supplied `.connect` export

The config-flow module states the design directly: the user imports a network
already provisioned by ThingOS / Häfele, pastes the `.connect` JSON, and the
integration stores the raw JSON verbatim in the config entry
(`custom_components/bluetooth_mesh/config_flow.py:1-9`).

The actual flow:

1. Offers only a multiline `connect_json` field
   (`config_flow.py:72-78`).
2. JSON-decodes the paste and calls `Network.from_connect`
   (`config_flow.py:51-69`, `94-104`).
3. Uses the parsed network identity to deduplicate and creates a config entry
   whose `data` is exactly `{CONF_CONNECT_JSON: text}`
   (`config_flow.py:107-118`).

`CONF_CONNECT_JSON` is explicitly described as the raw `.connect` export
stored verbatim in the config entry (`const.py:7-9`). The end-to-end config
flow test verifies that the stored text can be parsed back and yields the
expected NetKey and node address (`tests/ha/test_config_flow.py:48-71`).

### Fields imported

`Network.from_connect` requires and parses:

- `netKeys[0].key` and its index
  (`btmesh/network_model.py:303-307`, `343-350`);
- every entry in `appKeys[]`, including key, index, and name
  (`btmesh/network_model.py:308`, `382-395`);
- each usable node's `deviceKey`, unicast address, company ID, elements,
  models, and model AppKey bindings
  (`btmesh/network_model.py:314-354`, `500-518`, `540-560`).

The model's opening documentation is explicit that another app provisioned
the network and exported the NetKey, AppKey, node composition, and addressing
(`btmesh/network_model.py:1-7`). The unit tests prove that the fixture's
NetKey, AppKey, and DeviceKey all become model fields
(`tests/test_network_model.py:23-46`).

Although the legacy comment at `network_model.py:182-184` says Phase 1 rides
on the first NetKey/AppKey pair, current code parses all AppKeys. At runtime it
selects the key whose index the controlled lighting models actually bind,
falling back to the first exported key only if no carried key matches
(`network_model.py:217-260`; test at
`tests/test_network_model.py:255-273`). The coordinator applies this selection
in `_resolve_app_key` (`coordinator.py:338-365`), and the HA test confirms index
4 is selected when the driven model binds index 4
(`tests/ha/test_coordinator.py:981-996`).

### Runtime loading and use

On every config-entry setup, `MeshCoordinator` reparses the raw JSON from
`entry.data[CONF_CONNECT_JSON]` (`coordinator.py:199-204`). It passes the
imported NetKey and resolved AppKey into `MeshNode`; it registers each
imported per-node DeviceKey for device-keyed Foundation-model traffic
(`btmesh/controller.py:109-149`). Thus:

- the imported NetKey provides network-layer encryption/authentication and is
  also used to derive the advertised Network ID;
- the selected imported AppKey encrypts normal lighting access messages;
- each imported DeviceKey enables node-specific configuration/diagnostic
  requests, such as Composition Data Get (`btmesh/controller.py:362-388`).

## Storage and refresh behavior

### Where the keys are retained

All key material remains embedded verbatim in the Home Assistant config
entry's `connect_json`. It is not split into a separate keyring or Home
Assistant `Store`. The diagnostics module warns that the entry contains the
whole export, including NetKey, AppKey, and every DeviceKey, and therefore
never echoes entry data (`diagnostics.py:1-12`). Its test asserts that no raw
key and no `connect_json`/`deviceKey` field appears in a diagnostic dump
(`tests/ha/test_diagnostics.py:77-90`).

The coordinator's Home Assistant `Store` is only replay-safety state: sequence
cursor plus IV Index. It is keyed by network identity and initialized at
`coordinator.py:205-218`; persistence writes `_state_to_save`, not credentials
(`coordinator.py:1110-1120`).

### Key/node changes require a new export

There is no live key synchronization. Adding a node or refreshing a key means
exporting again in the vendor app and using Reconfigure. The reconfigure flow
replaces `connect_json` only after verifying that the replacement describes
the same network (`config_flow.py:127-152`). Tests confirm the raw export is
replaced and a different network is refused
(`tests/ha/test_config_flow.py:237-282`). README says the same at
`README.md:100-106`, and the changelog records this as the v0.4.0 behavior
(`CHANGELOG.md:837-840`).

## What Bluetooth discovery actually discovers

The radio path discovers a proxy **for a key already known from the export**;
it does not discover the key.

`find_proxy_address` receives the imported NetKey, computes
`network_id = k3(net_key)`, and looks for a proxy advertisement carrying that
8-byte Network ID (`mesh_transport.py:132-168`). A matching advert identifies
which proxy belongs to the already-known subnet. The advert exposes only the
derived Network ID, not the 16-byte NetKey. The tests construct adverts by
computing `k3(NET_KEY)` and verify matching/foreign selection
(`tests/ha/test_mesh_transport.py:37-44`, `67-80`).

Similarly, a Secure Network Beacon is authenticated using the NetKey already
loaded into the controller (`btmesh/controller.py:196-209`). From an
authenticated beacon the coordinator may learn and persist a newer **IV
Index**, resetting its sequence cursor (`coordinator.py:1122-1170`). This is
not key recovery. A beacon that cannot be authenticated under the imported
NetKey is ignored.

Device-keyed probes also do not read DeviceKeys from nodes. The controller
first registers each DeviceKey from the export
(`btmesh/controller.py:141-149`) and then uses that known key to send a
Foundation-model request (`btmesh/controller.py:362-388`). A response can
confirm reachability or report composition, but it does not export the node's
DeviceKey.

No integration code calls `Provisioner`, `scan_unprovisioned`, opens a bearer
with `provisioning=True`, or handles provisioning PDUs. Within
`custom_components/bluetooth_mesh`, searches for these symbols find only their
vendored definitions/constants, not a config-flow or coordinator call site.

## Provisioning code present but not exposed to Home Assistant

### Library provisioner

The vendored `Provisioner` is a transport-independent PB-GATT state machine
(`btmesh/provisioner.py:1-18`, `141-155`). Important key semantics:

- its caller must supply an existing 16-byte NetKey, NetKey index, IV Index,
  and assigned unicast (`provisioner.py:145-178`);
- it creates ephemeral P-256/private random provisioning material if not
  supplied (`provisioner.py:125-138`, `180-182`);
- after ECDH and confirmation it **derives a new DeviceKey** as
  `k1(secret, provisioning_salt, "prdk")`, then encrypts and sends the caller's
  NetKey/index/IV/unicast as Provisioning Data
  (`provisioner.py:360-387`);
- it does not discover an old DeviceKey and does not create/distribute an
  AppKey. AppKey Add / model binding is a later configuration operation.

The integration vendors this code as part of its internal stack, but having
the class in the package is not evidence that HA provisions. Documentation
settles the point: `docs/2026-07-26-audit-debug-and-improvements.md:344-346`
says `provisioner.py` is complete but unreachable from HA, and the changelog
repeats this at `CHANGELOG.md:216-217`.

### Separate Phase-0 own-network harness

`phase0/provision_and_toggle.py` is the only code that creates credentials. On
first state creation it generates random 16-byte NetKey and AppKey values and
writes them to its own JSON state file (`phase0/provision_and_toggle.py:222-240`).
It scans an **unprovisioned** beacon, provisions the selected device using its
fresh NetKey (`phase0/provision_and_toggle.py:373-445`), receives/derives the
new DeviceKey, and saves it in that Phase-0 state (`phase0/provision_and_toggle.py:497-516`).
It subsequently reloads those three keys, registers the DeviceKey, and performs
AppKey configuration/binding (`phase0/provision_and_toggle.py:760-817`).

This is an experimental command-line hardware harness, not the Home Assistant
integration's credential path. Git commit `5be435c` introduced it explicitly
as a "provision-and-toggle feasibility script." Later commit `ddc7f6e` recorded
the hardware result and chose to ride the vendor app's network for Häfele,
while leaving own-network provisioning for generic devices as a future Phase-1
idea. The actual HA config flow arrived in commit `326cd2d` with the explicit
design of importing and storing raw `.connect` JSON.

## Documentation/history corroboration

- README: the integration joins the existing mesh rather than replacing it
  (`README.md:34-36`).
- README coexistence section: the vendor app created the network; its export
  carries NetKey/AppKey/addressing; HA imports it and shares those keys
  (`README.md:153-164`). Installation requires exporting `.connect` and keeping
  it private because it contains mesh keys (`README.md:175-184`).
- Forum draft: provisioning still happens in the vendor app; HA joins the
  existing network; adding a new lamp from HA is not implemented
  (`docs/forum-post-en.md:70-77`).
- Audit roadmap: HA provisioning remains open and the library provisioner is
  unreachable (`docs/2026-07-26-audit-debug-and-improvements.md:344-346`).
- Git history: `326cd2d` introduced the HA config flow as `.connect` import;
  `a82da1a` later added re-import for node/key changes; `d0a02e8` added parsing
  all AppKeys and selecting by exported model bindings; `6b4094f` still labels
  provisioning as library-only and unreachable from HA.

## Direct answers

| Question | Answer |
| --- | --- |
| How does HA obtain NetKey? | User-pasted vendor `.connect` export, `netKeys[0].key`. |
| How does HA obtain AppKey? | Same export; it parses all `appKeys[]` and selects one by imported model bindings. |
| How does HA obtain DeviceKey? | Same export, from each node's `deviceKey`. |
| Where are credentials stored? | Verbatim inside the config entry's raw `connect_json`; not in the coordinator's `Store`. |
| Can discovery recover existing keys? | No. It matches `k3(imported NetKey)` against the advertised Network ID and authenticates beacons with the already-known NetKey. |
| Does HA provision or create its own network? | No, not in the shipping integration. Provisioning remains unreachable from HA. |
| Is provisioning code present? | Yes: a reusable library state machine plus a separate Phase-0 CLI harness. The harness can generate a fresh NetKey/AppKey and derive/store a new DeviceKey, but it is not wired into HA. |

