# Step 6 operational validation

This record closes the initial Olight-specific daemon milestone. It
separates automated lifecycle coverage, live read-only checks, and work
that was deliberately deferred.

## Automated service lifecycle

`tests/test_operational.py` runs the real service entry point with a
temporary state file and Unix socket while substituting an in-memory
Bluetooth transport. It verifies:

- exclusive state ownership while the service is running;
- socket mode `0600` and socket removal after shutdown;
- serialization of simultaneous client requests;
- session loss, cleanup, reconnection, and generation changes; and
- sequence state remaining valid after clean shutdown.

The complete suite passed with 195 tests on 2026-09-27. The systemd
user unit also passed `systemd-analyze --user verify`.

## Installed entry points

The project-local virtual environment initially contained an older
editable installation that predated `sphere-meshd` and
`sphere-meshctl`. Refreshing that editable installation restored all
four declared console programs. Their help paths were exercised before
the live check.

This is an environment finding rather than a source defect: an existing
editable install does not automatically regenerate console scripts
after `pyproject.toml` changes. The service guide now calls out the
required executable and socket-path checks.

## Live read-only restart check

The installed project-local `sphere-meshd` was started with the normal
provisioned state and a temporary socket. No Set or vendor Write was
sent. The checks established:

1. `status` reported a connected `0x0118` device and the expected
   OnOff, Lightness, CTL Temperature, HSL, vendor-read, effect, and
   countdown capabilities; xyL correctly remained unavailable.
2. A read requesting DP 103 returned the normal five-record snapshot.
   Every known record carried its semantic name, including battery at
   100%, Aurora at DP 102=`02`, and both countdowns at zero.
3. A second daemon using the same state was rejected by the exclusive
   state lock.
4. After the first process fully exited, a fresh daemon acquired the
   same state, reconnected, and returned healthy generation 1 status.
5. Both service exits removed their temporary socket.

The shutdown log is emitted before final transport and lock cleanup.
Starting a replacement during those last few seconds can still see the
old lock; a supervisor waits for process exit and therefore does not
have this race.

## Boundary

The repository's user unit was verified but not installed or enabled
in the user's service manager. Positive countdown behavior, a true
battery-off persistence test, and induced failure between live effect
writes were not needed for this milestone. Partial activation behavior
is covered offline and exposed through structured API error details.

## Desktop live-trial observations

A first desktop-GUI trial on 2026-09-27 added three product-specific
observations:

- CTL Temperature Sets received protocol support but produced no obvious
  visible change. The saved Sphere phone panel also has no temperature
  control, so the desktop app no longer presents or polls it.
- A bright pale-yellow static color made the cap LEDs appear much
  brighter than the cylinder LEDs; the cylinder may have been extremely
  dim or unlit. This was an uncontrolled visual observation, not evidence
  of separate addressable zones or failed emitters.
- After Flame was active, activating Cotton Candy timed out waiting for
  an acknowledged vendor DP write. No automatic write retry was made.
  Structured error details now identify whether the mode write or
  effect write lacked a reply.
