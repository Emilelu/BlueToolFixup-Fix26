# BlueToolFixup-Fix26

A **BlueToolFixup 2.7.3** build with **macOS 26 (Tahoe) compatibility fixes** and an
**AirPods A2DP fix**.

Used on: HP 15s-dy0xxx / macOS Tahoe 26.7.2, together with a Realtek RTL8821CE
Bluetooth controller (`0BDA:B00A`).

---

## What BlueToolFixup does

`BlueToolFixup` is a Lilu plugin from the **acidanthera/BrcmPatchRAM** family. Since macOS 12,
the Bluetooth stack moved from kernel space into user space (`bluetoothd`), so patching it
requires modifying the signed binary pages in memory. BlueToolFixup does this through
`_cs_validate_page`, patching `/usr/sbin/BlueTool` and `/usr/sbin/bluetoothd` on the fly —
the files on disk are never changed.

Its purpose is to make **third-party (non-Apple) Bluetooth controllers** acceptable to
`bluetoothd`, so the system can boot with a working Bluetooth stack instead of falling back
to the UART placeholder.

### Upstream patch set (9 patches)

| # | Target | Purpose |
|---|---|---|
| 1 | BlueTool | Skip automatic firmware update check |
| 2 | BlueTool | board-id string replacement (legacy adapters) |
| 3 | bluetoothd | Skip the "vendor must be Broadcom (`0x0A5C`)" check |
| 4 | bluetoothd | Accept chipset `0` = `THIRD_PARTY_DONGLE` (< 13.3) |
| 5 | bluetoothd | Chipset range check (Ventura 13.3+) |
| 6 | bluetoothd | Chipset range check (Sequoia 15.4+ / Tahoe 26) |
| 7 | bluetoothd | Internal-controller NVRAM check (`-btlfxnvramcheck`) |
| 8 | bluetoothd | board-id patch (`-btlfxboardid`, macOS 14+) |
| 9 | bluetoothd | Skip the "two controllers share the same address" check (`-btlfxallowanyaddr`) |

---

## What this build adds on top

### 1. macOS 26 (Tahoe) pattern correctness fix

The upstream `kBadChipsetCheckOriginal` 5-byte pattern, when applied on macOS 15.4+/26,
can match **three unrelated locations** and corrupt code. This build:

- applies the legacy pattern only on `< Sequoia`
- uses a dedicated pattern for 15.4+ / 26

### 2. `-btlfxa2dpcheck`

Forces the FastConnect A2DP "Read Clock" gate down the success path.

**Symptom it fixes:** AirPods (or other A2DP headphones) connect, but there is **no sound** —
the link is up but the A2DP stream is never established because the Read Clock step fails.

This is **not** related to BLE HID; for BLE HID keyboards see
[Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher).

---

## Version note

The latest **official** release of acidanthera/BrcmPatchRAM is **2.7.2**.
The `2.7.3` version string in this repository is a **local/third-party build** that carries
the two additions above. If you do not need them, use the official 2.7.2 release.

---

## Required NVRAM (not boot arguments)

```
bluetoothExternalDongleFailed      = 00                       (1 byte)
bluetoothInternalControllerInfo    = 00 x 14                  (exactly 14 bytes!)
```

> ⚠️ `bluetoothInternalControllerInfo` **must be exactly 14 bytes**.
> 13 bytes → `bluetoothd` crashes with `EXC_GUARD namespc 18 reason 2` in a loop.
> 29 bytes (28 hex characters parsed as ASCII) → `CFDataGetLength(...) != 14` error.
> See [Hackintosh-EFI-HP-15s-dy0xxx](https://github.com/Emilelu/Hackintosh-EFI-HP-15s-dy0xxx) for the full write-up.

---

## Boot arguments

| Argument | Meaning |
|---|---|
| `-btlfxoff` | Disable the plugin entirely |
| `-btlfxdbg` | Debug logging (DEBUG build only) |
| `-btlfxbeta` | Allow beta macOS |
| `-btlfxboardid` | Force board-id patch (macOS 14+) |
| `-btlfxallowanyaddr` | Skip the duplicate controller address check — **not** a BLE/RPA switch |
| `-btlfxnvramcheck` | Skip the internal-controller NVRAM check (macOS ≤ 14) |

---

## Important: Lilu debug logging on macOS 26

`DBGLOG` in Lilu plugins is compiled out entirely unless **both Lilu and the plugin are
DEBUG builds**. The official Lilu release binary (~531 KB) contains **zero** `DBG` strings,
so `-liludbg` has no effect with it.

To see plugin logs you need:
1. Lilu **DEBUG** build
2. Plugin **DEBUG** build
3. `-liludbg` (or `-liludbgall`)

Even then, early-boot `IOLog` output can be lost from the kernel log ring buffer because it is
overwritten by later traffic. For reliable state reporting, publish values as **IORegistry
properties** instead — that is what `RealtekBTPatcher` does.

---

## Install

1. `Kernel → Add`: `BlueToolFixup.kext` after Lilu.
2. Add the NVRAM entries above.
3. Reboot.

---

## Credits

- **[acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM)** — upstream `BlueToolFixup`; the entire patch set and `_cs_validate_page` technique are theirs. Original authors: **lvs1974**, **vit9696** and contributors.
- **[acidanthera/Lilu](https://github.com/acidanthera/Lilu)** — plugin framework and `_cs_validate_page` hooking infrastructure.
- **[acidanthera/bugtracker](https://github.com/acidanthera/bugtracker)** — issue discussions on third-party controller acceptance and macOS 26 pattern changes.
- **[thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware)** — Realtek controller firmware loader used alongside this plugin.
- **[Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher)** — companion BLE HID fix for RTL8821CE.

If your project should be credited and is missing, please open an issue.

---

## Disclaimer

Provided as-is, no warranty. Patching `bluetoothd` in memory is version-sensitive; a wrong
pattern can crash `bluetoothd` in a loop. Keep a bootable backup EFI and know `-btlfxoff`.
