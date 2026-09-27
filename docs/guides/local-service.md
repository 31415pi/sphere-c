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

In another terminal, inspect health and read states:

```console
./bin/sphere-meshctl health
./bin/sphere-meshctl control read
./bin/sphere-meshctl lighting lightness
./bin/sphere-meshctl lighting temperature
./bin/sphere-meshctl lighting hsl
./bin/sphere-meshctl vendor-read 2 102
./bin/sphere-meshctl effect list
./bin/sphere-meshctl effect get
./bin/sphere-meshctl effect select colour_2
./bin/sphere-meshctl effect activate colour_2
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
| `control` | `{"command":"read|on|off|toggle"}` |
| `lighting` | `{"command":"lightness|temperature|hsl|xyl..."}` |
| `vendor.read` | `{"dp_ids":[2,102]}` |
| `effect.list` | `{}` |
| `effect.get` | `{}` |
| `effect.set` or `effect.select` | `{"name":"colour_2"}` |
| `effect.activate` | `{"name":"colour_2"}` |
| `countdown.get` | `{}` |
| `countdown.set` | `{"seconds":60}` |
| `countdown.cancel` | `{}` |

Lighting commands use the same optional `=value` suffix documented in
the README. `vendor.read` sends only the Tuya/Olight DP Read opcode and
returns typed values plus their original bytes. It does not change the
Sphere. The response also identifies requested IDs omitted by the
device. Requests are limited to 64 KiB. Mutating and read commands
are serialized through the live Mesh session because access responses
have no correlation identifier beyond source and opcode.

`effect.list` includes all 32 schema effects with their saved English
labels and marks the 26 shown in the panel. `effect.get` reads DPs 2
and 102 and reports the current effect when DP 102 is returned. A
missing DP 2 is normal in the observed snapshot. `effect.set` and
`effect.select` are synonyms: they write DP 102 alone, preserving the
current work mode. `effect.activate` writes DP 2 (`colour`) and then
DP 102 in separate acknowledged requests, stopping if either fails.
Only visible picker names are accepted for writes. Write responses
may contain only an empty DATA acknowledgment, which confirms receipt
but not the resulting appearance. No automatic retransmission is used
for writes. These effect bytes are derived from the saved app and
are now validated for DP 102: a live experiment selected Aurora,
read it back, and restored Color Wheel. The DP 2 activation write and
the corresponding visible output remain untested.

The named countdown commands use DP 101, the countdown used by the
saved Sphere panel. `countdown.get` returns the current seconds and
the vendor snapshot. `countdown.set` accepts integer seconds from 0
through 43200; `countdown.cancel` is equivalent to setting zero. A
nonzero countdown may turn the lamp off later. Writes require an
acknowledgment and use no automatic retries. An empty acknowledgment
confirms receipt without reporting the resulting countdown. These
commands have only offline test coverage; no live countdown write has
been validated.

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
If running directly from this checkout rather than an installed package,
override `ExecStart` with the absolute path to `bin/sphere-meshd`.

Do not run the development-tree daemon and systemd service at the same
time against copies of the same mesh state. Separate sequence cursors
with the same source address and keys can reuse a nonce and invalidate
the network's security guarantees.
