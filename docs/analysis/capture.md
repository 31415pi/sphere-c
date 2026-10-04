# Sphere C capture analysis

This is the original app-network capture analysis. Later local
provisioning and authenticated control did not decrypt that old
network's traffic. Its proposed next experiments are historical;
see [the report](../../REPORT.md) and
[current reference](../../SPHERE-C-BT.md) for subsequent results.

## Scope and provenance

`TASK.md` was not present in this directory. `docs/project/original-task.md` was
used as the task brief instead.

The original `old/captures/sphere-c.btsnoop` was not modified.

- Format: BTSnoop v1, HCI UART (H4)
- Frames: 3,391
- Duration: 346.676597 seconds
- Capture time: 2026-09-26 10:57:01.452540 through
  11:02:48.129137 PDT
- SHA-256:
  `7fa0332b793b9445f6556d0a04dd5bad1846a32cfb2f093265e214e26fb3401b`
- TShark used: 4.6.6

## Main conclusion

This is standards-based Bluetooth Mesh traffic carried through a
Mesh Proxy GATT connection. The capture does not contain the Mesh
NetKey or AppKey. It also contains no Bluetooth SMP exchange and no
link-encryption setup, which agrees with the observation that the
device was not conventionally bonded.

The mesh payloads cannot be decoded into light commands from this
capture alone. They have valid-looking network-layer encryption and
authentication. Replaying captured PDUs is not a useful substitute:
Bluetooth Mesh sequence numbers and replay protection are designed
to reject old messages.

The most valuable next input is the NetKey and AppKey stored by the
Olight app, or access to a device reset/reprovisioning experiment in
which Linux chooses those keys.

## Connection and GATT map

The phone completed a BLE connection at frame 351, relative time
30.400783 seconds:

- Peer: `dc:23:52:09:a8:cf`
- Address type: public
- Connection handle: `0x0040`
- Interval: 18.75 ms
- Latency: 0
- Supervision timeout: 5 seconds

The peer advertised the standard Mesh Proxy service and this
service data:

```text
UUID 0x1828
00 1d 49 7f d7 60 45 01 69
```

The leading `00` is the Network ID advertising type. The remaining
eight bytes are Network ID `1d497fd760450169`.

Its scan response contained Tuya manufacturer data:

```text
company 0x07d0 (Hangzhou Tuya Information Technology)
cf a8 09 52 23 dc 10 00 00 00 00 00 00 00 00 00
01 02 03 04 05 06 07 08 09 0a 0b
```

The first six data bytes are the peer address in little-endian
order. This and the `0xfd50` GATT service are strong evidence of a
Tuya-based Bluetooth Mesh implementation.

Discovered services relevant to control:

| Service | Out/notify | CCCD | In/write command |
|---|---:|---:|---:|
| Mesh Provisioning `0x1827` | `0x0013` | `0x0015` | `0x0017` |
| Mesh Proxy `0x1828` | `0x001c` | `0x001e` | `0x0020` |
| Alternate proxy `0x7fdd` | `0x0025` | `0x0027` | `0x0029` |
| Tuya `0xfd50` | `0x0037` | `0x0039` | `0x003b` |

The Tuya characteristic UUIDs normalize to:

```text
notify: 00000002-0000-1001-8001-00805f9b07d0
write:  00000001-0000-1001-8001-00805f9b07d0
```

Only the standard Mesh Proxy pair was used. At frame 692 the phone
wrote `01 00` to CCCD handle `0x001e`. The first proxy notification
was a Secure Network Beacon at frame 711. Phone writes then used
write-without-response on handle `0x0020`; peer notifications used
handle `0x001c`.

## Mesh network facts

All proxy messages have SAR `0`, so each GATT value contains one
complete proxy message. There is no proxy-layer fragmentation.

The Secure Network Beacon is stable throughout the capture:

- Flags: `0x00` (no key refresh, no IV update)
- Network ID: `1d497fd760450169`
- IV Index: `0`
- Authentication value: `f58aa355815dc50c`
- Network NID: `0x30`

There are 251 Mesh Network PDUs through the proxy:

- Phone to proxy: 70
- Proxy to phone: 181

PDU length distributions:

| Bytes | Phone to proxy | Proxy to phone |
|---:|---:|---:|
| 20 | 32 | 0 |
| 21 | 4 | 27 |
| 22 | 6 | 27 |
| 24 | 16 | 4 |
| 25 | 5 | 47 |
| 26 | 0 | 9 |
| 27 | 5 | 0 |
| 28 | 0 | 67 |
| 29 | 2 | 0 |

For an ordinary unsegmented access message using four-byte network
and transport MICs, a 20-byte Network PDU corresponds to a two-byte
access payload. The common 20-byte request / 21-byte response shape
is therefore compatible with a two-byte SIG-model Get opcode and a
three-byte Status opcode plus one byte of state. This is only a
structural inference until keys allow authentication and decoding.

Activity occurs in 14 bursts when a one-second idle gap is used:

| Burst | Relative seconds | Frames | Tx | Rx |
|---:|---:|---:|---:|---:|
| 1 | 33.005-34.917 | 735-949 | 15 | 45 |
| 2 | 88.631-88.735 | 1738-1745 | 1 | 1 |
| 3 | 92.231-92.341 | 1886-1893 | 1 | 1 |
| 4 | 106.044-108.283 | 2577-2741 | 13 | 33 |
| 5 | 126.422-126.516 | 3107-3111 | 2 | 2 |
| 6 | 128.783-129.879 | 3112-3117 | 2 | 2 |
| 7 | 134.691-134.803 | 3119-3124 | 2 | 2 |
| 8 | 136.050-136.121 | 3125-3126 | 1 | 1 |
| 9 | 155.546-155.620 | 3132-3133 | 1 | 1 |
| 10 | 161.382-161.470 | 3135-3136 | 1 | 1 |
| 11 | 169.640-169.710 | 3141-3142 | 1 | 1 |
| 12 | 181.217-183.909 | 3152-3249 | 16 | 51 |
| 13 | 285.445 | 3279 | 0 | 1 |
| 14 | 344.390-346.676 | 3309-3390 | 14 | 39 |

The repeated one-request/one-response bursts are good candidates
for individual UI control actions. The larger response fans are
consistent with status discovery or group-address requests answered
by several mesh nodes. Exact identities remain encrypted.

## Olight `0xfcf1` advertisements

The capture includes 386 advertisements carrying service UUID
`0xfcf1`. Three stable service-data payloads are present:

```text
04 37 91 13 67 e4 d1 2d 31 0f 67 3f 62 b9 f6 13 f8 2a d3
04 96 97 13 2c 20 50 33 84 b1 c8 c0 42 70 03 35 70 e9 67 40
04 f3 45 dc b1 4d 76 26 cd da d8 c1 02 6c c2 bd 6f 3f 68 8b b5
```

The 21-byte third payload appeared under two addresses:

```text
25:73:b3:f1:0f:c4  from 30.225 to 41.403 seconds
01:fc:fb:b8:68:83  from 79.552 to 115.622 seconds
```

This directly confirms that an Olight advertising identity can
survive a random BLE address change. The service-data bytes, not the
address, are the stable correlation value in this capture. The
example service data in `docs/project/original-task.md` is not present byte-for-byte.

The other two stable payloads each stayed on one address during the
recorded scan interval. The capture alone does not identify which
payload belongs to the Sphere under test.

## What is not in the capture

- No Bluetooth SMP packets
- No HCI encryption-change event for this connection
- No Mesh Provisioning PDU exchange
- No NetKey, AppKey, or DeviceKey in cleartext
- No plaintext mesh source, destination, sequence, opcode, or model
  parameters
- No user-action labels

The cleartext GATT transport does not weaken Mesh security. The
Network PDU inside GATT is independently encrypted and authenticated.
The observed 64-bit Network ID is derived from the 128-bit NetKey;
it is useful for validating a candidate key but not for recovering
the key by practical brute force.

## Reproducible artifacts

`src/sphere_mesh/capture.py` uses local TShark for extraction. With no keys:

```sh
PYTHONPATH=src python3 -m sphere_mesh.capture --no-pdus
```

To export the exact proxy PDUs:

```sh
PYTHONPATH=src python3 -m sphere_mesh.capture \
  --no-pdus --csv old/analysis/proxy_pdus.csv
```

If keys are recovered, validate and decrypt them locally:

```sh
PYTHONPATH=src python3 -m sphere_mesh.capture \
  --net-key NETKEY_HEX \
  --app-key APPKEY_HEX
```

The script implements Bluetooth Mesh `s1`, `k1`, `k2`, `k3`, and
`k4`, network deobfuscation/decryption, and unsegmented AppKey access
decryption. It reports segmented and DeviceKey messages but does not
yet reassemble/decrypt them. `tests/test_capture.py` checks key
derivation against Bluetooth Mesh Profile vectors.

`old/analysis/proxy_pdus.csv` is the generated exact PDU inventory. It contains
frame number, relative time, direction, length, and encrypted
Network PDU bytes.

`docs/analysis/compatibility.md` compares this evidence with the original Sphere,
Tuya's published model mapping, and the public Sphere C DP profile.
`scripts/infer_mesh_models.py` reproduces the request/response length analysis
used in that comparison.

## Recommended next experiment

The least disruptive useful experiment is key extraction from the
Olight app's private data while the existing network remains intact.
Look for 16-byte NetKey and AppKey values in its database, preferences,
logs, or live process memory. Any NetKey candidate can be confirmed
immediately because `k3(NetKey)` must equal
`1d497fd760450169`; its `k2` NID must also equal `0x30`.

A capture with labeled UI actions would improve correlation but
would not remove the cryptographic block. Capturing a fresh standard
provisioning session also does not by itself reveal keys to a passive
sniffer because provisioning uses ECDH.

If app key extraction is unavailable, the definitive path is a
factory reset followed by provisioning from Linux with keys chosen
and retained by the Linux provisioner. That is disruptive and should
only be attempted with explicit approval and a recovery plan.
