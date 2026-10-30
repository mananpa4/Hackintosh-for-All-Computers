# SKILLS.md — reusable external techniques

> Distilled from sources outside this repository so future sessions don't
> re-derive them. Ideas and parameters are recorded here; before copying code,
> check each source's license (noted per section).

## 1. gibMacOS catalog technique (full macOS installers)

Source: `E:\Usuarios\Tony\Downloads\Programs\gibMacOS-master` (corpnewt,
**MIT License**) — `gibMacOS.py` + `Scripts/downloader.py`.

What it does differently from this repo's `recovery.py`: instead of Apple's
recovery protocol (`macrecovery` → ~700 MB BaseSystem via `boards.json`),
it reads Apple's **software-update catalogs** and downloads **full
installers** (`InstallAssistant.pkg`, 12+ GB, Big Sur and later; older
releases as product packages through Catalina for Windows USB builds).

Reusable pieces:

- **Catalog URL construction** (`gibMacOS.py: build_url`): 
  `https://swscan.apple.com/content/catalogs/others/index-{}.merged-1.sucatalog`
  where `{}` is the reversed, dash-joined macOS version list between the
  minimum and the target, with a per-channel suffix (`catalog_suffix`:
  `publicrelease` → `""`, `public` → `beta`, `customer` →
  `customerseed`, `developer` → `seed`). Channels are selectable at runtime.
- **Product filtering** (`get_catalog_data` / `ApplePrefs`): parse the
  catalog plist, keep products whose `ExtendedMetaInfo /
  InstallAssistantPackageIdentifiers` has `OSInstall ==
  "com.apple.mpkg.OSInstall"` or `SharedSupport` starting with
  `"com.apple.pkg.InstallAssistant"`. That predicate is the whole
  installer-vs-noise filter.
- **Local catalog cache**: `Scripts/sucatalog.plist` + `prod_cache.plist`
  allow `--savelocal` / offline reuse — the same shape this repo already
  uses with `boards.json`; a future `tools/fullinstaller.py` could mirror
  the flags (`--latest`, `--catalog`, `--local-catalog`).
- **Downloader** (`Scripts/downloader.py`): chunked download with progress,
  multiprocessing-friendly; `BuildmacOSInstallApp.py` shows the
  post-download step (open `InstallAssistant.pkg` on macOS → install app in
  `/Applications`).

Constraints if integrated: opt-in and network-only-when-asked (offline
rule); never bundle or publish what it fetches (no-redistribution rule);
full images don't fit FAT32, so the stick flow would need the two-partition
treatment the guide already describes for whole images.

## 2. WinToGo USB enumeration technique

Source: `E:\Usuarios\Tony\OneDrive\Documents\GitHub\WinToGoCreator`
(C/Win32, **GPL-3.0** — ideas portable, code copy needs a license check) —
`src/usb.c`, `src/disk.c`.

What is worth borrowing for `tools/stick.py --list` on Windows:

- **Enumeration** (`USB_Enumerate`, `src/usb.c`): iterate
  `\\.\PhysicalDrive<N>`, `DeviceIoControl(IOCTL_STORAGE_QUERY_PROPERTY)`
  → keep only `RemovableMedia == TRUE` **or** `BusType == BusTypeUsb`;
  `IOCTL_DISK_GET_DRIVE_GEOMETRY_EX` for size; friendly name via
  `SetupDiGetClassDevsW(GUID_DEVCLASS_DISKDRIVE)`. This is the same
  "removable/external/physical only" gate `stick.py` already enforces —
  a native equivalent for the Windows path.
- **Letter allocation**: prefer letters from the end of the alphabet
  (S/Z/Y… for EFI, W/… for data) to minimize conflicts.
- **Elevation pattern**: admin-only operations up front with a clear
  message instead of failing halfway through a disk.

What is NOT applicable: the whole partitioning flow (GPT + 100 MB FAT32
EFI + 128 MB MSR + NTFS + fixed VHDX via diskpart → VirtDisk →
`wimlib-imagex apply` → DISM drivers → `bcdboot`). It targets Windows-To-Go
(WIM/BCD/VHDX); this repo's stick is FAT32 + `EFI/` +
`com.apple.recovery.boot/`. Different target, different safety model —
`stick.py`'s list-is-the-gate + re-read-at-erase + `--place`/`--prepare`
split stays.

## 3. Tiger Lake MOD parameters (11th-gen mobile)

Source: `E:\PROGRAMAS\DISCOS\Mac_OS_X\Nuevo\EFI.Opencore.NoteBook.TigerLake.MOD`
(third-party working EFI; parameter source only — its merged `MaLd0n.aml`
  was deliberately not imported).

- **iGPU (Intel Xe, e.g. 8086:9A49)**: driver = **NootedGreen.kext** (not
  WhateverGreen, not vendored here). `DeviceProperties`
  `PciRoot(0x0)/Pci(0x2,0x0)`: `device-id = 499A0000`
  (`hex:499a0000`), `AAPL,ig-platform-id = 0000499A` (`hex:0000499a`),
  plus `AAPL,GfxYTile`, `complete-modeset`, `force-online`, `rps-control`,
  `hda-gfx = onboard-1`.
- **boot-args**: `-v alcid=12 watchdog=0 npci=0x2000 -wegnoegpu keepsyms=1
  debug=0x100` + NootedGreen `-ngreen*` flags (`-allow3d
  -disablegfxfirmware -ngreenexp`, log flags per its docs).
- **Kext set**: Lilu, VirtualSMC (+SMCProcessor/SMCBattery/SMCLight/
  SMCSuperIO), AppleALC, ECEnabler, NootedGreen, NVMeFix, IntelMausi,
  RealtekRTL8111, RTL812xLucy, AtherosE2200Ethernet, FeatureUnlock,
  RestrictEvents, BrightnessKeys, USBInjectAll, XHCI-unsupported,
  VoodooI2C(+HID)+VoodooPS2Controller. No WhateverGreen alongside
  NootedGreen.
- **Quirks worth noting**: `AppleXcpmCfgLock`, `DisableIoMapper`,
  `LapicKernelPanic`, `ProvideCurrentCpuInfo`, `XhciPortLimit` = true
  (first two already in `profiles/platform/laptop.toml`); Booter
  `DevirtualiseMmio` + `SetupVirtualMap` = true.
- **SMBIOS**: `MacBookPro16,2`; Drivers: HfsPlus/OpenCanopy/OpenRuntime/
  ResetNvramEntry.

## 4. CPU detector rules (as implemented in `tools/detect.py`)

- Intel Core `i[3579]-DDDD[Gx]` / 5-digit SKUs: first digit(s) = generation;
  5-digit or G-suffix SKUs take the first two digits (2600K → 2nd gen,
  1065G7 → 10th).
- 10th-gen mobile split by G suffix: `10510U` → `comet-lake`, `1065G7` →
  `ice-lake`.
- 11th-gen mobile (`1135G7`, `1165G7`, `1185G7`…) → **`tiger-lake`**;
  11th-gen desktop (`11700`…) → `rocket-lake`.
- Ryzen/Threadripper/Epyc → `ryzen-threadripper`; FX/A-series → 
...[truncated 615 chars]