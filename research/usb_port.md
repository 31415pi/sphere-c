# Sphere C USB-C investigation

The manual describes the USB-C port as a charging input, but that
does not prove the data pins are absent or unused.

A later, non-destructive experiment should connect the Sphere C to
Linux with a known-good USB data cable and record USB enumeration,
kernel events, and power behavior in these states:

1. Powered normally.
2. Light switched off.
3. Connected while the bottom button is held.
4. Connected after entering Bluetooth pairing mode.

Useful outcomes include a USB device descriptor, serial port, HID,
DFU interface, vendor interface, or an explicit absence of any USB
enumeration. The test must not inject signals, probe internal pads,
or open the enclosure without separate approval.

## Non-destructive enumeration result: 2026-09-26

The Sphere C was connected to Linux with a known-good USB data cable
while a live kernel and udev USB monitor was active. A USB inventory
was recorded before and after connection. The following states were
tested:

- connected while powered on;
- connected while switched off, with the charging LED illuminated;
- unplugged and reconnected while the bottom button was held;
- several additional button/connect combinations.

No kernel or udev USB event appeared in any state. The USB inventory
was unchanged, and the kernel journal contained no connect,
descriptor, reset, or enumeration-error message. Charging did work.

This is strong evidence that the exposed USB-C port is charge-only in
normal and tested button-held states. It does not prove that the USB-C
data pins are physically absent: they could be left disconnected,
routed only to inaccessible test circuitry, or gated by an unknown
factory condition. Further progress would require continuity/board
inspection or undocumented boot-mode knowledge, rather than more
ordinary host enumeration attempts.
