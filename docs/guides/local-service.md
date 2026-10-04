# Local Sphere service

## Purpose

`sphere-meshd` is the sole owner of the Bluetooth Mesh state and GATT
Proxy connection. It keeps the state file exclusively locked, reconnects
after link loss, and exposes a versioned local API through a private
Unix-domain socket. `sphere-meshctl` is a separate client; it never reads
mesh keys or opens Bluetooth itself.

The service emits one JSON log object per line. Health responses expose
connection state, generation, active device address, and the most recent
failure. They do not expose the NetKey, AppKey, or DeviceKey.

## Development-tree use

Start the service using the already provisioned state:

```console
./bin/sphere-meshd
```

To watch app commands and their Mesh responses, start it with
`--verbose`:

```console
./bin/sphere-meshd --verbose
```

Each debug interaction appears in a JSON log line. The trace shows
the API method and accepted parameters, Mesh operation and elapsed
time, decoded response, or error type. Effect activation shows the
mode and effect writes separately. Invalid parameters are redacted;
Mesh keys, state contents, and raw vendor bytes are omitted. Stop the
existing daemon before starting this instance, since only one daemon
can own the state and Bluetooth connection.

In another terminal, inspect health and read states:

```console
./bin/sphere-meshctl health
./bin/sphere-meshctl status
./bin/sphere-meshctl control read
./bin/sphere-meshctl lighting lightness
./bin/sphere-meshctl lighting temperature
./bin/sphere-meshctl lighting hsl
./bin/sphere-meshctl vendor-read 2 102
./bin/sphere-meshctl effect list
./bin/sphere-meshctl effect get
./bin/sphere-meshctl effect select aurora
./bin/sphere-meshctl effect activate aurora
./bin/sphere-meshctl countdown get
./bin/sphere-meshctl countdown set 60
./bin/sphere-meshctl countdown cancel
```

The lighting Set syntax matches the direct harness:

```console
./bin/sphere-meshctl lighting lightness=32768
./bin/sphere-meshctl lighting temperature=5000
./bin/sphere-meshctl lighting hsl=32768,0,65535
```

Only one process may own the state. Stop the service before running a
direct `sphere-mesh` command against the same state file.

## API protocol

The socket carries one newline-terminated JSON request and response per
connection. Version 1 requests have this envelope:

```json
{"version":1,"id":1,"method":"health","params":{}}
```

Successful responses contain `ok: true` and `result`. Failures contain
`ok: false` and an error object with a stable `code` and human-readable
`message`. Current methods are:

| Method | Parameters |
|---|---|
| `health` | `{}` |
| `status` | `{}` |
| `control` | `{"command":"read\|on\|off\|toggle"}` |
| `lighting` | `{"command":"lightness\|temperature\|hsl\|xyl..."}` |
| `vendor.read` | `{"dp_ids":[2,102]}` |
| `effect.list` | `{}` |
| `effect.get` | `{}` |
| `effect.set` or `effect.select` | `{"name":"aurora"}` |
| `effect.activate` | `{"name":"aurora"}` |
| `countdown.get` | `{}` |
| `countdown.set` | `{"seconds":60}` |
| `countdown.cancel` | `{}` |

Lighting commands use the same optional `=value` suffix documented in
the README. `vendor.read` sends only the Tuya/Olight DP Read opcode and
returns typed values, semantic names for known IDs, and the original
bytes. Unknown IDs have a `null` name. It does not change the
Sphere. The response also identifies requested IDs omitted by the
device. Requests are limited to 64 KiB. Mutating and read commands
are serialized through the live Mesh session because access responses
have no correlation identifier beyond source and opcode.

`status` combines the `health` fields with `capabilities` derived from
the saved Mesh composition. It works while reconnecting. A capability
means the required server model is present, not that the current link
is ready or that every device behavior has been visually tested.

Read and write results include `verification`:

| Field | Meaning |
|---|---|
| `transport_acknowledged` | The request received its expected Mesh or vendor response. |
| `response_reported_value` | That response contained a relevant state value. |
| `independent_readback` | This operation made a separate Get/Read request. |
| `visually_verified` | A human checked the visible lamp output during this operation. Currently always `false`. |

An acknowledged Set can report a state in its response, but that is
not an independent readback. An empty vendor DATA response sets only
`transport_acknowledged`. A Get/Read result can have a missing vendor
DP even though its request was answered; check `missing_dp_ids` too.

`effect.list` includes all 32 schema effects with their saved English
labels, canonical schema names, and unique slugs for the 26 shown in
the panel. Hidden entries have a `null` slug. `effect.get` returns the
same metadata for the current effect and reads DPs 2 and 102. A
missing DP 2 is normal in the observed snapshot. Its `work_mode`
field is DP 2's numeric value, and `effect_active` is true only
when that mode is `colour` (1). Both are `null` when the mode is
unknown. `work_mode_source` is `readback`, `cached`, or `null`;
the cache holds the last known mode for the current daemon connection
and may be stale if another client changes the mode while DP 2 reads
omit it. The selected DP 102 effect alone does not prove it is active.
`effect.set` and `effect.select` are synonyms: they write DP 102 alone,
preserving the current work mode. `effect.activate` writes DP 2
(`colour`) and then DP 102 in separate acknowledged requests, stopping
if either fails.
Both `effect.set` and `effect.select` remain supported for existing
clients; `effect.select` is the clearer spelling for new clients.
If the mode write is acknowledged but the effect write fails, the
error contains `details.completed_steps`, `failed_step`, and
`result_uncertain`. Clients should read the current effect before
deciding what to do next.
Writes accept a visible slug (for example, `aurora`, `color-wheel`,
`cotton-candy`, or `candy-cane`) or its canonical schema name (for
example, `colour_2`). The six hidden schema values cannot be written.
Write responses
may contain only an empty DATA acknowledgment, which confirms receipt
but not the resulting appearance. Ordinary vendor writes disable
the node's automatic retries. During activation, the daemon waits
250 ms after DP 2 and may retry only DP 102 once after a response
timeout. Selection alone and countdown writes are not retried.
These effect bytes are derived from the saved app and
are now validated for DP 102: a live experiment selected Aurora,
read it back, and restored Color Wheel. Live activation and visible
output have since been confirmed.

The named countdown commands use DP 101, the countdown used by the
saved Sphere panel. `countdown.get` returns the current seconds and
the vendor snapshot. `countdown.set` accepts integer seconds from 0
through 43200; `countdown.cancel` is equivalent to setting zero. A
nonzero countdown may turn the lamp off later. Writes require an
acknowledgment and use no automatic retries. An empty acknowledgment
confirms receipt without reporting the resulting countdown. These
commands have live wire validation for the neutral value zero: it was
echoed by the Sphere and independently read back. Positive durations
and their delayed power-off behavior remain untested.

The service socket is mode `0600`. If its path is occupied by a regular
file, the daemon refuses to replace it. A stale socket left by a crash
is removed only after a connection attempt proves no service is
listening.

## systemd user service

An example unit is in `packaging/systemd/sphere-mesh.service`. It expects
the console programs to be installed on the user PATH and the private
state file at:

```text
~/.local/state/sphere-mesh/state.json
```

Install the unit under `~/.config/systemd/user/`, place the existing
mode-`0600` state file at that location, and then run:

```console
systemctl --user daemon-reload
systemctl --user enable --now sphere-mesh.service
systemctl --user status sphere-mesh.service
```

The unit uses `%t/sphere-mesh/control.sock`, restarts after failures,
and restricts filesystem writes to its runtime and state directories.
The installed client's default socket is the development-tree path, so
pass the service socket explicitly:

```console
sphere-meshctl --socket "$XDG_RUNTIME_DIR/sphere-mesh/control.sock" health
```

The package must be installed in an environment visible on the unit's
`PATH` (`~/.local/bin`, `/usr/local/bin`, or `/usr/bin`). Check that
`command -v sphere-meshd` resolves to that installation before enabling
the unit. A virtual environment elsewhere needs a local `ExecStart`
override pointing to its absolute `sphere-meshd` executable.
If running directly from this checkout rather than an installed package,
override `ExecStart` with the absolute path to `bin/sphere-meshd`.

Do not run the development-tree daemon and systemd service at the same
time against copies of the same mesh state. Separate sequence cursors
with the same source address and keys can reuse a nonce and invalidate
the network's security guarantees.

## Operational validation

The isolated service test uses a temporary state file and socket, and
replaces the Bluetooth transport with a fake. It checks exclusive state
ownership, socket permissions and cleanup, health across reconnection,
and serialization of simultaneous requests. It does not contact a
device:

```console
./.venv/bin/python -m pytest -q tests/test_operational.py
```

After installing the user unit, these checks only inspect service
status and the local health response:

```console
systemctl --user status sphere-mesh.service
systemctl --user show sphere-mesh.service -p MainPID -p ExecMainStatus
sphere-meshctl --socket "$XDG_RUNTIME_DIR/sphere-mesh/control.sock" health
```

`health` makes no Mesh request. A `connecting` response can be normal
while the device is asleep; inspect `last_failure` and the service log
if it remains disconnected. Do not start a second daemon using a copy
of the same provisioned state to perform this check.
