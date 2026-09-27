# btmesh dependency provenance

The Sphere tooling uses the MIT-licensed `btmesh` library from:

```text
https://github.com/dasimon135/ha-bluetooth-mesh
```

The project dependency is pinned to:

```text
version: 0.10.3
tag:     v0.10.3
commit:  a8f7bbbebaeba124d7d08cb16122055074f2ef46
```

That commit was the upstream `main` and `v0.10.3` target when checked
on 2026-09-26. PyPI exposed only `btmesh` 0.1.0 at that time, so a
normal `btmesh==0.10.3` dependency could not reproduce the tested
code. `requirements.txt` therefore uses a PEP 508 direct Git
dependency pinned to the full commit rather than a moving branch or
tag.

An earlier working copy vendored the upstream source during hardware
experimentation. The repository now treats that local checkout as a
replaceable cache and ignores `vendor/`; upstream source and tests are
not duplicated in this project history.

Comparison against a wheel built from the pinned commit found two
local changes in the earlier working copy. They are now explicit
first-party compatibility code rather than undocumented modifications
to the dependency:

- `src/sphere_mesh/transport.py` selects GATT characteristics within the
  intended Mesh service when firmware exposes duplicate UUIDs, ignores
  stale BlueZ advertisements, and can leave discovery active during a
  scan-to-connect handoff.
- `src/sphere_mesh/access.py` supplies Config Network Transmit Set and its
  Status opcode, both standard Bluetooth Mesh messages used by the
  Olight setup sequence but absent from upstream 0.10.3.

Project tests cover those deltas, and a read-only authenticated status
round trip against the physical Sphere passed using the installed
upstream wheel plus this compatibility layer.

The upstream license is MIT. Its license, source, and release history
remain available in the pinned upstream repository.
