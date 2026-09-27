# Sphere C vendor-DP live query

## Result

On 2026-09-27, a locally provisioned Sphere C accepted the recovered
Tuya/Olight READ command on vendor model `0x07d0:0x0004`. The operation
was read-only; no vendor WRITE opcode was sent.

The combined request was:

```text
cc d0 07 01 02 02 66
```

The Sphere used a two-message reply. It first returned an empty DATA
acknowledgment:

```text
cd d0 07 01
```

It then returned a segmented DATA status snapshot equivalent to:

```text
cd d0 07 01
  07 02 00000000
  65 02 00000000
  66 04 07
  67 02 00000064
  68 01 00
```

The decoded records were:

| DP | Type | Raw | Value |
|---:|---|---|---:|
| 7 | value | `00000000` | 0 |
| 101 | value | `00000000` | 0 |
| 102 | enum | `07` | 7 (`colour_7`) |
| 103 | value | `00000064` | 100 |
| 104 | bool | `00` | false |

The same snapshot was returned for focused reads of only DP 2 and only
DP 102. DP 2 was absent in all three replies. The evidence therefore
does not establish a readable current `work_mode`; it does establish
the current DP 102 value and the exact live response grammar.

The repeated full snapshot suggests that the firmware treats the READ
ID list as a refresh trigger rather than a strict response projection.
Another possibility is that DP 2 is write-only or omitted in the
current state. A mode-changing experiment could distinguish these, but
was deliberately not performed during this read-only validation.

## Implementation consequence

The local controller waits briefly after the first matching DATA reply
and combines subsequent DATA records from the same element. Returning
only the first matching response would silently discard the segmented
snapshot because the first response contains no records.
