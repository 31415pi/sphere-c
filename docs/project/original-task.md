# Olight Sphere C Bluetooth Reverse Engineering

We are reverse-engineering the Bluetooth protocol used by an **Olight Sphere C** so that it can ultimately be controlled directly from Linux without the Olight app.

The device is mine and is already provisioned/configured in the Olight Android app.

## Available capture

The working directory contains:

` old/captures/sphere-c.btsnoop `

This is an Android Bluetooth HCI snoop capture made while I controlled the already-provisioned Sphere C using the Olight app.

Use `tshark`, Wireshark command-line facilities, Python, and other local tools as useful. You may create scripts, decoded captures, notes, or other analysis artifacts in this directory. Do not modify the original capture.

## What we already know

Initial passive scanning from Linux found a device advertising:

- Random BLE address; observed example: `2D:D3:3E:5A:02:A7`
- Service UUID: `0000fcf1-0000-1000-8000-00805f9b34fb`
- Example service data:

```text
04 37 ea 56 77 86 02 d4 1b d7 43 fc 44 b0 17 40
46 e0 12 e9 71
```

The address appears to randomize.

The Sphere was already configured in the Olight app. Android did not report it as conventionally paired/bonded during the Linux observation, which may indicate application-layer provisioning rather than an Android BLE bond.

## Initial HCI analysis

`tshark -q -z io,phs` reported:

```text
bluetooth
  hci_h4
    bthci_cmd
    bthci_evt
      btmesh
    bthci_acl
      btl2cap
        btatt
          btmproxy
            btmesh
```

The capture contains approximately:

- 3391 total frames
- 354 ACL frames
- 352 ATT frames
- 252 Bluetooth Mesh Proxy frames
- 251 Mesh frames underneath Mesh Proxy
-