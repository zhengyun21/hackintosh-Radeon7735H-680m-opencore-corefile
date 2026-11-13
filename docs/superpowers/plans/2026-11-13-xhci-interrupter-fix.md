# xHCI Interrupter Mouse-Latency Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans
> to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax
> for tracking.

**Goal:** Reroute normal AppleUSBXHCI transfer TRBs from interrupter 1 to
interrupter 0 so continuous wired-mouse movement is delivered promptly.

**Architecture:** Add six reversible OpenCore Kernel Patch entries to a new
`xhci-fix` profile derived from the working `usbmap-lab` profile. The patches
change only one interrupter-target byte in each of six verified AppleUSBXHCI
instruction patterns; no macOS files and no GPU kext are modified.

**Tech Stack:** Python 3 plistlib, OpenCore 1.0.8 `ocvalidate`, PowerShell
profile installer, USBToolBox map, physical macOS Recovery boot.

---

## File Structure

- Modify: `outputs/efi-profiles/build_profiles.py`
  - Owns profile derivation and the six xHCI patch definitions.
- Modify: `outputs/efi-profiles/switch-profile.ps1`
  - Allows `-Profile xhci-fix`.
- Create: `outputs/efi-profiles/test_xhci_fix.py`
  - Verifies patch count, lengths, offsets and preserved profile components.
- Generate: `outputs/efi-profiles/xhci-fix/config.plist`
  - Installable profile with marker `ocxhci1`.
- Runtime: `F:\EFI\OC\config.plist`
  - Installed copy, never source-of-truth for the repository.

## Task 1: Add a failing generated-profile test

**Files:**
- Create: `outputs/efi-profiles/test_xhci_fix.py`

- [ ] **Step 1: Write the failing validation test**

```python
#!/usr/bin/env python3

import plistlib
import unittest
from pathlib import Path
from zipfile import ZipFile


ROOT = Path(__file__).resolve().parent
CONFIG = ROOT / "xhci-fix" / "config.plist"
BACKUP = (
    ROOT.parents[1]
    / "outputs"
    / "backups"
    / "F-boot-payload-EFI-and-recovery-20261110-105107.zip"
)
KC_NAME = (
    "com.apple.recovery.boot/System/Library/KernelCollections/"
    "BootKernelExtensions.kc"
)
EXPECTED = [
    ("setup", "488901c7410808004000", 8),
    ("normal", "c744080800004000c744080c21040000", 6),
    ("data", "4409f181c90000400042894c2a08", 7),
    ("event", "48c7041800000000c744180800004000", 14),
    ("status", "c744080800004000c744080c00180000", 6),
    ("sparse", "41b8000040004489443e08", 4),
]


class XhciFixProfileTest(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        with CONFIG.open("rb") as handle:
            cls.data = plistlib.load(handle)

    def test_boot_marker_and_resolution(self):
        boot = self.data["NVRAM"]["Add"][
            "7C436110-AB2A-4BBB-A880-FE41995C9F82"
        ]["boot-args"]
        self.assertTrue(boot.endswith(b"ocxhci1"))
        self.assertEqual(self.data["UEFI"]["Output"]["Resolution"], "1280x720@1")

    def test_xhci_patches_change_only_one_byte(self):
        patches = [
            item
            for item in self.data["Kernel"]["Patch"]
            if item.get("Comment", "").startswith("XHCI-FIX |")
        ]
        self.assertEqual(len(patches), len(EXPECTED))
        for patch, (site, find_hex, offset) in zip(patches, EXPECTED):
            self.assertIn(site, patch["Comment"])
            self.assertEqual(patch["Identifier"], "com.apple.driver.usb.AppleUSBXHCI")
            self.assertEqual(patch["Find"].hex(), find_hex)
            self.assertEqual(len(patch["Find"]), len(patch["Replace"]))
            changed = [
                index
                for index, (old, new) in enumerate(
                    zip(patch["Find"], patch["Replace"])
                )
                if old != new
            ]
            self.assertEqual(changed, [offset])
            self.assertEqual(patch["Find"][offset], 0x40)
            self.assertEqual(patch["Replace"][offset], 0x00)

    def test_kernel_patterns_each_occur_once(self):
        with ZipFile(BACKUP) as archive:
            kernel = archive.read(KC_NAME)
        for _, find_hex, _ in EXPECTED:
            self.assertEqual(kernel.count(bytes.fromhex(find_hex)), 1)

    def test_existing_capabilities_are_preserved(self):
        add = {
            item["BundlePath"]: item["Enabled"]
            for item in self.data["Kernel"]["Add"]
        }
        self.assertTrue(add["AirportItlwm.kext"])
        self.assertTrue(add["USBToolBox.kext"])
        self.assertTrue(add["UTBMap.kext"])
        self.assertTrue(add["AmdTscSync.kext"])
        self.assertFalse(self.data["Kernel"]["Quirks"]["XhciPortLimit"])

    def test_no_gpu_experiment_is_enabled(self):
        add = {
            item["BundlePath"]: item["Enabled"]
            for item in self.data["Kernel"]["Add"]
        }
        self.assertFalse(add.get("RembrandtGfx.kext", False))
        self.assertFalse(add.get("NootedRed.kext", False))
        self.assertFalse(add.get("RDNA3Driver.kext", False))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run it and verify it fails because the profile does not exist**

Run:

```powershell
C:\Python314\python.exe .\outputs\efi-profiles\test_xhci_fix.py
```

Expected: FAIL with `FileNotFoundError` for `xhci-fix\config.plist`.

## Task 2: Implement the profile builder

**Files:**
- Modify: `outputs/efi-profiles/build_profiles.py`
- Modify: `outputs/efi-profiles/switch-profile.ps1`

- [ ] **Step 1: Add the profile directory and patch table**

Add beside the existing profile directory constants:

```python
XHCI_FIX_DIR = OUT / "xhci-fix"
XHCI_PATCHES = (
    ("setup", "488901c7410808004000", 8),
    ("normal", "c744080800004000c744080c21040000", 6),
    ("data", "4409f181c90000400042894c2a08", 7),
    ("event", "48c7041800000000c744180800004000", 14),
    ("status", "c744080800004000c744080c00180000", 6),
    ("sparse", "41b8000040004489443e08", 4),
)
```

- [ ] **Step 2: Add an idempotent profile constructor**

Add after `make_usbmap_lab`:

```python
def make_xhci_fix(usbmap_lab: dict) -> dict:
    data = deep_copy(usbmap_lab)
    nvram = data.setdefault("NVRAM", {}).setdefault("Add", {}).setdefault(
        BOOT_UUID, {}
    )
    nvram["boot-args"] = (
        b"keepsyms=1 -no_compat_check -no_restart nvme=-1 ocxhci1"
    )
    patches = data.setdefault("Kernel", {}).setdefault("Patch", [])
    patches[:] = [
        patch
        for patch in patches
        if not patch.get("Comment", "").startswith("XHCI-FIX |")
    ]
    for site, find_hex, offset in XHCI_PATCHES:
        find = bytes.fromhex(find_hex)
        replace = bytearray(find)
        replace[offset] = 0
        patches.append(
            {
                "Arch": "x86_64",
                "Base": "",
                "Comment": f"XHCI-FIX | {site}: interrupter target 1 -> 0",
                "Count": 1,
                "Enabled": True,
                "Find": find,
                "Identifier": "com.apple.driver.usb.AppleUSBXHCI",
                "Limit": 0,
                "Mask": b"",
                "MaxKernel": "24.99.99",
                "MinKernel": "24.0.0",
                "Replace": bytes(replace),
                "ReplaceMask": b"",
                "Skip": 0,
            }
        )
    (XHCI_FIX_DIR / "profile.json").write_text(
        json.dumps(
            {
                "profile": "xhci-fix",
                "base_on": "usbmap-lab / ocusbmap1",
                "marker": "ocxhci1",
                "purpose": (
                    "Route normal AppleUSBXHCI TRBs to interrupter 0 so AMD "
                    "XHCI transfer events are delivered without ring-wrap delay."
                ),
            },
            ensure_ascii=False,
            indent=2,
        )
        + "\n",
        encoding="utf-8",
    )
    return data
```

- [ ] **Step 3: Wire the profile into `main`**

Add directory creation:

```python
XHCI_FIX_DIR.mkdir(parents=True, exist_ok=True)
```

Add profile construction after `usbmap_lab`:

```python
xhci_fix = make_xhci_fix(usbmap_lab)
```

Add serialization after the `usbmap-lab` config:

```python
with (XHCI_FIX_DIR / "config.plist").open("wb") as handle:
    plistlib.dump(xhci_fix, handle, sort_keys=False)
```

Add output:

```python
print(f"Wrote {XHCI_FIX_DIR / 'config.plist'}")
```

- [ ] **Step 4: Allow the profile in the installer**

Add `xhci-fix` to the `ValidateSet` in
`outputs/efi-profiles/switch-profile.ps1`.

- [ ] **Step 5: Run the test**

Run:

```powershell
C:\Python314\python.exe .\outputs\efi-profiles\build_profiles.py
C:\Python314\python.exe .\outputs\efi-profiles\test_xhci_fix.py
```

Expected: `Ran 5 tests ... OK`.

- [ ] **Step 6: Record generated hashes**

```powershell
Get-FileHash `
  .\outputs\efi-profiles\xhci-fix\config.plist `
  -Algorithm SHA256
```

Expected: print one deterministic SHA-256 for the generated profile. The
workspace is not a Git repository; the sanitized final checkpoint is archived
to the GitHub repository after the physical test.

## Task 3: Validate generated configuration

**Files:**
- Validate: `outputs/efi-profiles/xhci-fix/config.plist`

- [ ] **Step 1: Parse and validate**

Run:

```powershell
C:\Python314\python.exe .\outputs\efi-profiles\test_xhci_fix.py
.\work\ocvers\1.0.8\Utilities\ocvalidate\ocvalidate.exe `
  .\outputs\efi-profiles\xhci-fix\config.plist
```

Expected:

```text
Ran 5 tests ... OK
No issues found.
```

The test already confirms that each of the six patterns has exactly one hit in
the backed-up kernel collection.

## Task 4: Install to F when the USB volume is connected

**Files:**
- Modify: `F:\EFI\OC\config.plist`

- [ ] **Step 1: Verify F is the intended removable FAT32 volume**

Run:

```powershell
Get-Volume -DriveLetter F
```

Expected: `FileSystem=FAT32`, label `F206`, removable device.

- [ ] **Step 2: Install with automatic rollback backup**

Run:

```powershell
powershell -ExecutionPolicy Bypass `
  -File .\outputs\efi-profiles\switch-profile.ps1 -Profile xhci-fix
```

Expected:

```text
Installed profile 'xhci-fix'.
```

- [ ] **Step 3: Validate F**

Run:

```powershell
.\work\ocvers\1.0.8\Utilities\ocvalidate\ocvalidate.exe `
  F:\EFI\OC\config.plist
```

Expected: `No issues found.`

## Task 5: Physical acceptance test

**Files:** None.

- [ ] **Step 1: Boot `F206 (external) (dmg)` with the mouse on the known-good XHCI port.**
- [ ] **Step 2: Confirm the macOS Recovery menu is reached.**
- [ ] **Step 3: Move the pointer continuously for at least ten seconds.**

Expected: pointer motion tracks the physical mouse without visible multi-second
backlog.

- [ ] **Step 4: Click Disk Utility and confirm click response remains immediate.**
- [ ] **Step 5: If pointer latency remains unchanged, revert to `usbmap-lab` and classify the remaining cause as display/cursor composition.**

## Task 6: Archive the tested result

**Files:**
- Copy sanitized config to:
  `docs/config-backups/config-xhci-fix-2026-11-13.plist`
- Copy tested config to:
  `EFI/OC/config.plist`

- [ ] **Step 1: Clear `SystemSerialNumber`, `MLB`, `SystemUUID` and `ROM` in both repository copies.**
- [ ] **Step 2: Parse both plist files and assert equality.**
- [ ] **Step 3: Run `ocvalidate` against `EFI/OC/config.plist`.**
- [ ] **Step 4: Run `git diff --check`.**
- [ ] **Step 5: Commit and push `record: add tested xHCI interrupter latency fix`.**
