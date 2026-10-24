# HP 400 G Series · Hackintosh OpenCore EFI

[简体中文](README.md) | **English**

OpenCore boot configuration for HP 400 G series desktops (Coffee Lake / 300-series platform), SMBIOS `iMac20,1`, Intel UHD 630 iGPU.

> ### ⚠️ Read this first
> **All SMBIOS identifiers (Serial, MLB, SystemUUID, ROM) have been wiped from this repo.**
> The EFI will still boot, but **iCloud, iMessage, FaceTime and the App Store will not work**
> until you generate your own set and put it into `config.plist` — see [Serial & MLB](#serial--mlb-must-be-regenerated).

---

## Overview

| Item | Value |
|---|---|
| SMBIOS | `iMac20,1` |
| iGPU | Intel UHD 630 (`AAPL,ig-platform-id = 0x3E9B0007`, `device-id = 0x3E9B`, `framebuffer-stolenmem = 0x01300000`, custom EDID) |
| Ethernet | Intel I219-LM → `IntelMausi.kext` |
| Wi-Fi / Bluetooth | Broadcom BCM94360 family → `AirportBrcmFixup` + `BrcmPatchRAM3` + `BrcmFirmwareData` + (`IOSkywalkFamily` / `IO80211FamilyLegacy` / `AirPortBrcmNIC`) |
| Audio | `AppleALC.kext`, boot-arg `alcid=20` |
| Bootloader | OpenCore 1.0.x (original `OpenCore.efi` shipped; `config.plist` differs from 1.0.8 by 4 `HideVerbose` keys only — see [FAQ](#faq)) |
| SIP | `csr-active-config = 0x803` |
| SecureBootModel | `Disabled` |
| boot-args | `keepsyms=1 debug=0x100 igfxonln=1 amfi=0x80 ipc_control_port_options=0 alcid=20` |

## Layout

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── OpenCore.efi
    ├── config.plist
    ├── ACPI/       # SSDT-AWAC, SSDT-AWAC-HPET-RTC, SSDT-EC-USBX-DESKTOP, SSDT-PLUG, SSDT-PMC
    ├── Drivers/    # HfsPlus, OpenCanopy, OpenRuntime, ResetNvramEntry
    ├── Kexts/      # 18 kexts (CPUFriend pair disabled by default)
    ├── Resources/  # OpenCanopy theme (Acidanthera GoldenGate)
    └── Tools/      # OpenShell.efi
```

## Quick start

1. Prepare a **FAT32** USB stick and copy the whole `EFI` folder to the root of its EFI partition (keep the `EFI/BOOT` + `EFI/OC` structure).
2. In BIOS: **disable Secure Boot / Fast Boot**, enable **Above 4G Decoding**, set DVMT / VGA Share Memory to **64 MB or more**, boot mode **UEFI only (no CSM)**.
3. Boot from the stick, open the OpenCore picker, start the macOS installer, format the target disk as **APFS / GUID** in Disk Utility, then install.
4. Copy `EFI` to the **internal disk's EFI partition** and put OpenCore first in the boot order (or run `bcfg boot add 0 FS0:\EFI\OC\OpenCore.efi "OpenCore"` from `UEFI Shell` in the picker).

## Serial & MLB (must be regenerated)

In `config.plist` → `PlatformInfo` → `Generic` these four keys are placeholders:

| Key | Current | What to use |
|---|---|---|
| `SystemSerialNumber` | `XXXXXXXXXXXX` | Generate for `iMac20,1` with GenSMBIOS / macserial |
| `MLB` | `XXXXXXXXXXXXXXXXX` | Same tool, generated together with the serial |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` | Any random UUID — or keep all zeros and let OpenCore derive it from hardware |
| `ROM` | `00 00 00 00 00 00` | **Your real `en0` (Ethernet) MAC address**, 6 bytes, no separators |

Edit with ProperTree / OCC, reboot once, then sign in — if iMessage fails, try FaceTime first.

> Never publish a `config.plist` that contains your own identifiers: one machine, one set. Duplicate or rapidly changing serials trigger Apple's risk control.

## Known limitations

- **The Broadcom Wi-Fi stack targets macOS 15 Sequoia**: `IOSkywalkFamily` / `IO80211FamilyLegacy` / `AirPortBrcmNIC` are capped with `MaxKernel 25.99.99`, and `Kernel > Block` disables `com.apple.iokit.IOSkywalkFamily`.
  Before upgrading to **macOS 26 Tahoe**, swap in `AirPortBrcmNIC-Tahoe.kext` (`MinKernel 25.0.0`) — otherwise Wi-Fi disappears after the upgrade.
- `amfi=0x80` disables AMFI: **as a side effect, privacy prompts (mic / camera / screen recording) stop appearing** and apps never get a TCC entry. Remove it together with `AMFIPass.kext` if you don't need it.
- Continuity / iPhone features depend on Wi-Fi being `en0`; re-check after swapping cards or applying root patches.
- **Not included**: `EFI/HP` (HP BIOS + ME firmware) and `EFI/Microsoft` (Windows boot manager + BCD) from the original ESP — vendor firmware and machine-specific Windows boot data don't belong in a public repo. `oldConfig.plist` is excluded as well.

## FAQ

**`ocvalidate` says `Missing key HideVerbose` after upgrading OpenCore to 1.0.8+?**
1.0.8 added `HideVerbose` to every `UEFI > Drivers` entry. Add `HideVerbose = false` to all four; older builds ignore the key, so it is safe either way.

**Stuck at `[EB|#LOG:EXITBS:START]`?**
Check `Booter > Quirks`: `EnableWriteUnprotector`, `SetupVirtualMap`, `ProvideCustomSlide` must stay `true` (already set here). These are the easiest ones to lose when swapping OpenCore versions.

**No audio?**
Try other `alcid` values (`11 / 13 / 1 / 3`); HP board codecs differ, so the layout-id is not universal.

**Can I install macOS 26 with this EFI?**
It can boot, but fix the Wi-Fi entries first (see above). macOS 26 is stricter about SMBIOS and kext versions — install and validate on an older release first.

## Changelog

### 1.0.0
- First release: complete OpenCore EFI (5 ACPI tables, 18 kexts, 4 drivers, GoldenGate theme)
- All SMBIOS identifiers and ROM wiped; `EFI/HP`, `EFI/Microsoft` and `oldConfig.plist` excluded

## Credits & license

- Bootloader and drivers by [Acidanthera](https://github.com/acidanthera) (OpenCorePkg, Lilu, VirtualSMC, WhateverGreen, AppleALC, AirportBrcmFixup, BrcmPatchRAM, IntelMausi, CPUFriend, AMFIPass)
- ACPI patches generated with [CorpNewt SSDTTime](https://github.com/corpnewt/SSDTTime)
- Theme icons from [OcBinaryData](https://github.com/acidanthera/OcBinaryData)
- Configuration files and documentation in this repo are released under **MIT** (see [LICENSE](LICENSE)); bundled binaries such as `OpenCore.efi` and the kexts remain under their own authors' licenses.

## Disclaimer

Hackintosh is risky: you can lose data or brick firmware. Back up first and always keep a **known-good spare USB EFI**. Everything here is provided "as is", without warranty of any kind.
