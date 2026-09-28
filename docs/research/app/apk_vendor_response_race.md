# Olight APK handling of early Tuya DATA type `00`

## Question and conclusion

This note traces the Olight 3.2.5 / bundled Thingclips SDK behavior for
vendor DATA opcode `0xCDD007` when a command-type `0x00` message can
precede ordinary command-type `0x01` or `0x05` DATA.

The APK does not route every `0xCDD007` message through its datapoint
parser. It reads the first parameter byte before selecting a status
class. Types `0x01` and `0x05` become
`ThingVendorModelStatus`; type `0x00` falls through to generic
`VendorModelStatus` pass-through. Thus the early `00` message does not
become a DP list through the normal dispatcher. Both messages still
arrive as separate decoded status objects at the shared access-message
status path. The command callback matches the source node block and
expected opcode, but not the status subclass, command type, or returned
DP IDs. An early generic `00` status can therefore complete a waiting
read. This does not prevent a later `01`/`05` status from independently
entering the device-property update path.

## Source artifacts

The source is the Olight 3.2.5 APK at
`old/apk/olight-3.2.5.apk`, previously disassembled to
`/tmp/olight-dex.7xC3jV/smali6/`. Relevant classes:

- `com/thingclips/sdk/bluetooth/bpqqqpp.smali`:
  access status dispatcher, method
  `bdpdqbp(AccessMessage)`.
- `com/thingclips/sdk/sigmesh/provisioner/ThingVendorModelStatus.smali`:
  ordinary DP DATA parser, method
  `parseStatusParameters()`.
- `com/thingclips/sdk/sigmesh/provisioner/VendorModelStatus.smali`:
  generic vendor pass-through status class.
- `com/thingclips/sdk/bluetooth/ppbpddp.smali`:
  access-message receive path; around lines 1930–1934 it calls the
  dispatcher for received access messages.
- `com/thingclips/sdk/bluetooth/ppbppqq.smali`:
  vendor read action, model `0x7d00004`, company `0x07d0`, request
  opcode `0xcc`, response opcode `0xcd`.
- `com/thingclips/sdk/bluetooth/pbbpdbb.smali`:
  vendor status to schema/DP report translator.

## Dispatcher and parser behavior (direct evidence)

In `bpqqqpp.smali`, `bdpdqbp(AccessMessage)` extracts the first
parameter byte into `v1` (unsigned) before examining opcode. For opcode
`0xcd`:

1. Type `0x82` maps to `GroupDeviceGetStatus`.
2. Types `0x01` and `0x05` map to
   `ThingVendorModelStatus` (the test is the `if-eq v1, v0` /
   `const/4 v0, 0x5` branch ending near smali line 351).
3. Several other explicit types map to specialized statuses.
4. The fallback constructs `VendorModelStatus` and logs
   “Pass through vendor success” (near lines 376–380).

Consequently, a command type `0x00` takes the generic fallback. It is
not passed to the ordinary DP parser by this dispatcher.

`ThingVendorModelStatus.parseStatusParameters()` reads parameter byte
zero into `commandType`, copies bytes after it into its DP data buffer,
then parses DP records. This confirms that its DP decoder consumes a
leading command-type byte and only follows the dispatcher’s `01`/`05`
selection. Its parser has exception handling and may preserve a partial
list on malformed input; that is separate from dispatch of type `00`.

`ppbppqq` defines the read action with expected opcode `0xcd`.
`pbbqbqp.onEventMainThread(pbdqqdq)` handles the decoded status event.
For vendor statuses, which share the `pbddbqq` base class, it masks the
source with `0xfff8`, compares that node block with the target, and then
compares the received opcode with the action's expected opcode. The
ordinary completion branch does not inspect the concrete status class,
command type, or returned DP IDs before calling `onSuccess()` and
destroying the waiter.

## Receive and pending-callback boundary

`ppbpddp.smali` shows the received `AccessMessage` passed into
`bpqqqpp.bdpdqbp(AccessMessage)`. The dispatcher returns a decoded
status object. That establishes the receive-to-parser path and shows
that early and later messages are independently decoded according to
their first parameter byte.

`pbbpdbb.bdpdqbp(String, qpppdqq)` branches on
`instanceof ThingVendorModelStatus` before it calls `getDpList()` and
iterates datapoints (around smali lines 270–281). Its selector likewise
chooses this vendor translator only for `ThingVendorModelStatus`; a
generic status uses the general SIG Mesh parser. The vendor translator
looks up a `DeviceBean` by source address and consults its schema while
converting enum/raw values. This proves that ordinary `01`/`05` status
can enter the DP report translation path, while `00` does not enter
that typed-DP path.

`ppbpddp.bdpdqbp(qpppdqq)` processes every decoded status before
publishing the status event used by the command callback. Its parser
selector chooses the DP-to-schema translator only for
`ThingVendorModelStatus`. Nonempty translated DPs are sent through the
device update path. At the end, `ppbbqbb.bdpdqbp(qpppdqq)` publishes a
`pbdqqdq` status event. A later `01`/`05` message therefore still updates
the device-property path even if an earlier generic `00` event already
completed and destroyed the pending action callback.

The report translator evidence also does not prove a global cache write
independent of request completion. It establishes schema lookup and DP
conversion, not whether parsed reports update a device cache before,
after, or separately from the pending read callback. Thus the source
does not currently support either claim that an early `00` completes
the waiter or that a later `01`/`05` necessarily updates global cache
independently of it.

## What follows, and what does not

**Proven:** Olight's opcode/parameter dispatcher recognizes `01` and
`05` as ordinary DP status and sends `00` through the generic vendor
status path. The ordinary DP parser is therefore not the parser that
interprets a leading `00`.

**Proven:** The command completion matcher is loose enough for an early
generic `00` status to complete an ordinary `0xcd` read action. The
property-report path is independent and processes every later decoded
status, selecting only `ThingVendorModelStatus` for ordinary DP schema
translation.

**Not established:** That command type `00` precedes `01`/`05` in one
specific Sphere transaction. The local runtime observed `00`, but no
authenticated capture currently records both messages with timing.

For comparison, the local controller behavior and its risk are
documented in `docs/research/mesh/vendor_type00_runtime_audit.md`;
that controller's source/opcode-only waiter is separate evidence and
must not be attributed to the phone app.
