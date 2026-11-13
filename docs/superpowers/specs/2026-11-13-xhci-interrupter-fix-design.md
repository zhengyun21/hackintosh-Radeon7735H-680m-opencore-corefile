# xHCI Interrupter Mouse-Latency Fix

## Goal

Remove the continuous pointer-motion delay on the Rembrandt laptop without
introducing the Radeon 680M driver yet.

The observable target is:

- click response remains immediate;
- continuous pointer motion no longer waits for periodic USB ring wrap;
- USB keyboard, storage, Type-C and future AX210 Bluetooth continue to work;
- no macOS system files are modified.

## Evidence

The working RembrandtGfx reference contains a dedicated AppleUSBXHCI fix:

- macOS prepares normal USB transfer TRBs for interrupter 1.
- The AMD XHCI exposes one usable MSI message.
- Transfer events on interrupter 1 are not delivered promptly; its ring is
  drained when an event on interrupter 0 runs the shared I/O handler.
- Rerouting normal TRBs to interrupter 0 restores prompt delivery.

All six reference byte sequences were found exactly once in the current
`BootKernelExtensions.kc`.

## Scope

Create a new OpenCore profile named `xhci-fix` based on
`usbmap-lab / ocusbmap1`.

Enable six Kernel Patch entries for:

```text
Identifier: com.apple.driver.usb.AppleUSBXHCI
Arch: x86_64
Base: empty
Count: 1
Skip: 0
Limit: 0
MinKernel: 24.0.0
MaxKernel: 24.99.99
```

Each patch changes only the `0x40` interrupter-target byte to `0x00`.

| Site | Find |
|---|---|
| setup | `488901c7410808004000` |
| normal | `c744080800004000c744080c21040000` |
| data | `4409f181c90000400042894c2a08` |
| event | `48c7041800000000c744180800004000` |
| status | `c744080800004000c744080c00180000` |
| sparse | `41b8000040004489443e08` |

The replacement has the same length and preserves every other byte.

The profile keeps:

- USBToolBox and UTBMap;
- AirportItlwm for the future AX210;
- `SSDT-EC-USBX` and `SSDT-PLUG-ALT-16CPU`;
- AmdTscSync;
- the standard AMD kernel patch set;
- the 1280x720 display request;
- boot marker `ocxhci1`.

It does not enable RembrandtGfx, NootedRed, RDNA3Driver, or a GPU identity spoof.

## Validation

Before installation:

- parse the generated plist;
- run OpenCore `ocvalidate`;
- verify each find/replace has equal length;
- confirm only the intended six patches are added.

Physical acceptance test:

1. Boot `F206 (external) (dmg)` with the mouse on the known-good controller.
2. Move the pointer continuously for at least ten seconds.
3. Confirm clicks remain immediate and pointer motion tracks without a visible
   backlog.
4. Confirm installer USB and mouse still enumerate.
5. Confirm the system reaches the Recovery menu without a new hang.

## Failure Handling

The patch only changes the in-memory kernel collection loaded by OpenCore.
It does not write to the macOS volume.

If input or USB storage regresses:

1. select the previous `usbmap-lab / ocusbmap1` config;
2. reboot;
3. leave the six interrupter patches disabled.

If pointer latency is unchanged, the remaining leading explanation is the
unaccelerated display/cursor composition path, and the next separate project is
the RembrandtGfx bring-up.
