# Step 4 live lighting validation

## Result

The initial standard-lighting controller was tested against the
locally provisioned Sphere C at `0x0118`. Lightness, CTL Temperature,
and HSL Get and acknowledged Set all returned authenticated status.
The test changed each state, restored the value read before the test,
and read all three states again over the same GATT Proxy connection.

The saved composition correctly predicts the live capabilities:

- Lightness Server `0x1300` responds;
- CTL Temperature Server `0x1306` responds;
- HSL Server `0x1307` responds; and
- xyL Server `0x130c` is absent and is rejected before connecting.

## Read-only baseline

The first live batch made no changes and returned:

```text
Lightness:       present=1956
CTL Temperature: present=800 K, delta_uv=0
HSL:             lightness=3637, hue=1820, saturation=65535
```

The independent Lightness and HSL-lightness fields did not report the
same number. The controller preserves both values rather than treating
them as interchangeable.

## Reversible Set test

The second batch performed these acknowledged operations:

1. set Lightness to `4000`, then restore `1956`;
2. set CTL Temperature to `3000 K`, then restore `800 K`; and
3. set HSL hue to `20000`, then restore the baseline HSL tuple.

Every Set Status immediately reported the exact requested values.
Follow-up Gets confirmed that Lightness and CTL Temperature restored
exactly.

The HSL follow-up exposed a firmware behavior that needs further
characterization. Restoring HSL `(3637, 1820, 65535)` was immediately
acknowledged with those exact values, but the later HSL Get returned
`(3276, 1638, 65535)`. The scale change is consistent with an internal
color conversion or coarse schema quantization. It is not consistent
with message corruption: the Set Status parsed correctly and the
other model states remained correct.

A separate read-only connection after the lamp had settled returned
the same `(3276, 1638, 65535)` HSL tuple while Lightness remained
`1956` and CTL Temperature remained `800 K`. The conversion is stable,
not merely an in-flight transition that omitted Remaining Time.

Two calibration observations showed that this is not a single scale
factor. A compensating Set of `(4038, 2022, 65535)` settled to
`(3997, 2002, 65535)`. The hue results correspond to coarse whole-degree
steps, while lightness follows the firmware's internal color
conversion. A final Set of `(3678, 1840, 65535)` settled to the exact
original `(3637, 1820, 65535)` tuple. Final reads also confirmed the
original independent Lightness `1956` and temperature `800 K`.

The daemon should therefore continue to report actual HSL Status and
must not claim that a requested raw HSL tuple remains stable merely
because its Set was acknowledged. A later API can expose normalized
color values and characterize the firmware's conversion separately.
