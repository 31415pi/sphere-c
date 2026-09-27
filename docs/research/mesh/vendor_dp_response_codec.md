# Tuya vendor DATA response codec

## Result

For the non-TID Tuya vendor model used by Sphere C, the access
opcodes are:

```text
READ  cc d0 07
DATA  cd d0 07
```

The exact parameter layouts implemented by the Olight 3.2.5 APK are:

```text
READ parameters:
    command_type:u8 || dp_count:u8 || dp_id:u8 * dp_count

DATA parameters:
    command_type:u8 || dp_record || dp_record || ...

dp_record:
    dp_id:u8 || dp_type:u8 || value_encoding
```

There is **no DP count in DATA**. There is also no request ID, TID,
or other correlation field in this non-TID parameter format. Records
run until the end of the access-message parameters.

For the two Sphere C enum DPs of immediate interest:

```text
DP 2 enum record:    02 04 VV
DP 102 enum record:  66 04 VV
```

Thus a DATA message containing both records is:

```text
cd d0 07  CT  02 04 MODE  66 04 EFFECT
```

`CT` is the command type. The APK recognizes both `01` and `05` as
ordinary `ThingVendorModelStatus` DATA. The corresponding READ forms
are:

```text
cc d0 07  01 02 02 66
cc d0 07  05 02 02 66
```

The read-action encoder selects `01` when its boolean mode flag is
false and `05` when it is true. The saved code does not name that flag
or establish its wire semantics. Use `01 02 02 66` as the ordinary
baseline vector, retain `05 02 02 66` as an APK-supported alternate,
and accept both in a decoder. The local artifacts contain no
authenticated Sphere C capture proving which command type its firmware
requires or returns. The most likely response behavior is to preserve
the request's command type, but that echo rule is an inference, not a
captured fact.

## Evidence set

The primary artifact is:

```text
old/apk/olight-3.2.5.apk
SHA-256 1fa9bddeb55ec39fba631e074be76f65ada37830735ae9a074f2aac83b6eecb9
```

It was already decoded to smali under:

```text
/tmp/olight-dex.7xC3jV/
```

The downloaded product mini-app is under:

```text
old/phone_external/files/Godzilla/MiniApp/
  9E6DD95017EDCABEA0673DC3A9AFAC12/
Resources/0.5.9_RsPYtSG2yNGWYhKJ/b1y/eo8LLi.js
SHA-256 024c0b833914dfd52102f4d96568ddda2be3223cf5a98b5b2c402eb18f1d8bf5
```

The relevant APK classes are:

- `com/thingclips/sdk/bluetooth/ppbppqq.smali`, the obfuscated
  `ThingVendorModelGetAction`;
- `com/thingclips/sdk/sigmesh/provisioner/
  ThingVendorModelStatus.smali`, the DATA parser;
- `com/thingclips/sdk/bluetooth/bpqqqpp.smali`, the access-message
  status dispatcher;
- `com/thingclips/sdk/bluetooth/pbbpdbb.smali`, the vendor DP-to-schema
  report translator; and
- `com/thingclips/sdk/bluetooth/pdpbqqp.smali`, the matching write DP
  record encoder.

These are local reverse-engineering names. The short obfuscated class
names are retained so the result can be reproduced against the saved
APK.

## READ encoding

`ppbppqq` constructs a vendor-model action with:

```text
model identifier  0x07d00004
company ID        0x07d0
request opcode    0xcc
expected opcode   0xcd
```

Its parameter assembler allocates `2 + number_of_DPs` bytes, writes
the command type, writes the DP count, and then writes every DP ID as
one byte:

```text
offset  size  meaning
0       1     command type: 01 if flag=false, 05 if flag=true
1       1     DP count, modulo 256
2       N     N one-byte DP IDs
```

For DP 2 and DP 102 together, `N=2`, `2=0x02`, and `102=0x66`:

```text
01 02 02 66
05 02 02 66
```

The action constructor also de-duplicates requested IDs through a
`HashSet`. Consequently, callers must not rely on the request retaining
input order. That does not change the response record grammar.

An empty list encodes as `CT 00`. Elsewhere, the vendor parser uses
DP ID zero as an all-DP-style query (`CT 01 00`). Neither convention
is needed for a focused DP 2/102 read.

## DATA record grammar

`ThingVendorModelStatus.parseStatusParameters()` removes the first
parameter byte as `commandType`, then walks all remaining bytes as DP
records. Its exact per-type grammar is:

| Type | Byte | DATA value encoding |
|---|---:|---|
| raw | `00` | `length:u8 || length bytes` |
| bool | `01` | one byte |
| value | `02` | four bytes, big-endian integer |
| string | `03` | `length:u8 || length bytes` |
| enum | `04` | one byte |
| bitmap | `05` | `length:u8 || length bytes`; valid lengths 1, 2, 4 |

The resulting general DATA layout is therefore:

```text
CT
  ID 00 LEN RAW...
  ID 01 BOOL
  ID 02 B3 B2 B1 B0
  ID 03 LEN STRING...
  ID 04 ENUM
  ID 05 LEN BITMAP...
```

This agrees with the APK's write encoder, but the DATA parser is the
direct evidence for response decoding. In particular, a DATA decoder
must not reuse the READ body's second byte as a count. In DATA, byte 1
is already the first record's DP ID.

The APK dispatcher sends only command types `01` and `05` on opcode
`cd` to `ThingVendorModelStatus`. Other command types select special
vendor status classes or generic pass-through parsing. For a focused
DP state reader, accepting `01` and `05` and rejecting other command
types avoids misinterpreting group, heartbeat, time, DST, and other
vendor control replies as DP state.

The saved parser catches bounds/type exceptions and may retain a
partial DP list. A new codec should instead reject a truncated record,
an unknown type, an invalid bitmap length, or trailing bytes that do
not form a complete record.

## Exact DP 2 values

The Sphere C mini-app schema defines DP 2, `work_mode`, as:

```text
range = ["white", "colour", "scene", "music"]
```

The native report translator treats the enum byte as a zero-based
index into the schema's `range` array. Therefore:

| Wire value | Schema value |
|---:|---|
| `00` | `white` |
| `01` | `colour` |
| `02` | `scene` |
| `03` | `music` |

The complete DP 2 record is:

```text
02 04 00   # white
02 04 01   # colour
02 04 02   # scene
02 04 03   # music
```

## Exact DP 102 values

The Sphere C mini-app schema defines DP 102, `colour_mode`, with this
ordered range:

```text
["flame", "colour_1", "colour_2", ... , "colour_31"]
```

The same zero-based enum translation gives:

```text
00       flame
01       colour_1
02       colour_2
...
1e       colour_30
1f       colour_31
```

The complete DP 102 record is:

```text
66 04 NN
```

where `NN` is `00..1f`. Values outside that interval are invalid for
the saved Sphere C schema, even though the generic wire field can hold
a full byte.

## Response examples

If a `01 02 02 66` READ returns both values in one DATA message, these
are exact codec vectors:

```text
# work_mode=colour, colour_mode=flame
cd d0 07 01 02 04 01 66 04 00

# work_mode=colour, colour_mode=colour_2 (Aurora)
cd d0 07 01 02 04 01 66 04 02

# work_mode=music, colour_mode=colour_31
cd d0 07 01 02 04 03 66 04 1f
```

The same DP body with leading command type `05` is also valid to the
APK parser:

```text
cd d0 07 05 02 04 01 66 04 02
```

The parser also permits separate DATA messages, because it consumes
whatever complete records occur after one command byte:

```text
cd d0 07 01 02 04 01
cd d0 07 01 66 04 02
```

No local Sphere transaction proves whether the lamp batches the two
requested DPs, replies separately, preserves request order, or adds
other records. A robust reader must allow all four possibilities. It
should collect requested IDs from one or more matching DATA messages
until both are present or a timeout expires.

## Request/response correlation

The plain `cc`/`cd` format provides semantic but not unique
transaction correlation:

1. Match the three-byte opcode/company tuple: READ `cc d0 07` expects
   DATA `cd d0 07`.
2. For a unicast query, require the DATA source to belong to the
   queried node and the destination to be the local client address.
   A stricter host can require the known vendor-server element.
3. Require the same AppKey/security context used for the request.
4. Accept command type `01` or `05`; prefer the requested command type
   when deciding which response is most likely related.
5. Parse records and retain only requested DP IDs 2 and 102.
6. Use a bounded receive window and deduplicate repeated Mesh delivery
   by the normal network/transport replay state.

Do not expect the Mesh sequence number, SeqAuth, or request opcode to
be echoed in the DATA parameters. Do not prepend or consume a TID in
this codec. The APK has distinct TID-bearing vendor status paths using
other opcode bytes (`cb` and `ce`); they are not the `cc` READ / `cd`
DATA pair documented here.

Because an unsolicited state publication has the same `cd` DATA
syntax, opcode, source, command type, and requested DP IDs can make a
reply highly likely but cannot cryptographically bind it to one READ.
Serialize focused reads per node and treat a matching publication
during the receive window as fresh state rather than claiming an exact
request ID match.

The APK's own completion waiter is looser and makes the lack of a
transaction key explicit. It stores the action's expected response
opcode (`cd`), allows five seconds, masks the received source with
`fff8`, compares that node base with its target, and then compares the
received opcode with `cd`. It does **not** compare the response command
type or returned DP IDs. Consequently, any element in the same
eight-address node block can satisfy the APK address test, and two
simultaneous ordinary DATA waits for the same node cannot be uniquely
attributed by the vendor parameters.

This behavior is in `pbbqbqp.smali`: the source-range comparison is at
physical lines 1018-1205 and the ordinary opcode-only completion is at
1659-1713. The response wait is created from the action's expected
opcode after send completion in `bppqppq$bdpdqbp.smali`.

A group-address READ is less precise: every member may reply from its
own unicast source. Correlation must then be per source address, and a
fan of DATA messages is expected.

## Confidence

| Finding | Confidence | Basis |
|---|---|---|
| READ is `cc d0 07`; DATA is `cd d0 07` | Very high | APK action constants and dispatcher |
| READ body is `CT COUNT IDs...` | Very high | Direct byte-buffer assembly |
| DATA body is `CT RECORD...`, with no count | Very high | Direct DATA parser loop |
| Enum record is `ID 04 VALUE` | Very high | Direct DATA parser and matching write encoder |
| DP 2 is `02`; DP 102 is `66` | Very high | Saved product mini-app schema |
| DP 2 values are `00..03` in documented order | Very high | Schema order plus native zero-based translation |
| DP 102 values are `00..1f` in documented order | Very high | Schema order plus native zero-based translation |
| Command types `01` and `05` are both DP DATA | Very high | APK dispatcher accepts both |
| `01` is the ordinary baseline read command type | High | Direct action encoding; `05` remains an unnamed flag-selected alternate |
| DATA echoes the READ command type | Medium | Protocol-shaped inference; no authenticated local capture |
| Two requested DPs arrive in one ordered DATA | Low/unknown | Parser permits it, but no authenticated local capture |

## Implementation consequences

A production query API should send a focused request rather than an
all-DP query:

```text
opcode:      cc d0 07
parameters:  01 02 02 66
```

Its response decoder should:

- accept only `cd d0 07` from the intended node/security context;
- parse `CT` once, then parse records to end of parameters;
- accept `CT` values `01` and `05`;
- decode DP 2 and 102 only when their type byte is `04`;
- validate DP 2 as `00..03` and DP 102 as `00..1f`;
- merge separate or combined responses; and
- expose correlation as source/opcode/window based, not TID based.

The byte layouts above are suitable as exact unit-test vectors. The
device's choice of `01` versus `05`, batching, ordering, and response
fan-out still require an authenticated live read to upgrade the noted
behavioral inferences to hardware facts.
