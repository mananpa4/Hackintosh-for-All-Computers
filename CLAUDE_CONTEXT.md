# CLAUDE_CONTEXT.md — live project memory

> Update this file whenever important tasks complete or relevant decisions are
> taken. Stable knowledge lives in `CLAUDE.md`; this file is the session
> state. Last updated: 2026-09-09.

## Current state

The repository is the upstream project plus one local feature batch
(implemented 2026-09-09, verified, not yet pushed): CPU-generation detection
fix + Tiger Lake laptop support. Docs trio (`CLAUDE.md`, this file,
`SKILLS.md`) created; `README.md` extended. Working tree status beyond that
is upstream-clean — check `git status` before assuming.

Toolchain notes: `python3` on this machine is 3.14.3 (engine targets 3.11+,
worked fine). Visual Studio lives at
`F:\Program Files\Microsoft Visual Studio\18\Enterprise` (dotnet CLI remains
the primary GUI build path). External sources consulted live outside the
repo — see "External resources" below.

## Implemented (this session, verified)

- **CPU detector fix** (`tools/detect.py`): laptop generation 11 now maps to
  `tiger-lake` instead of `ice-lake` (Ice Lake is 10th-gen mobile with G
  suffix; it shadowed Tiger Lake). Comment added explaining that laptop
  12th gen+ intentionally maps to nothing (no profile yet).
- **New profile** `profiles/cpu/laptop/tiger-lake.toml`: modeled on
  `ice-lake.toml` (generic SSDTs, vendored kexts only, placeholder
  `hex:12345678`, SMBIOS `MacBookPro16,2` per the MOD). Header comment
  documents the NootedGreen manual path instead of referencing the kext.
- **Labels/tables**: `CPU_LABELS['tiger-lake']` in `setup.py`;
  `IGPU_GENERATIONS['Tiger Lake/Rocket Lake']` in `gputable.py` now
  `['rocket-lake', 'tiger-lake']` with the matching `data/gpu.toml` row
  hand-edited (next `--refresh` must reproduce it — generator and file were
  changed together per convention 6).
- **Verification run**: `build.py --platform laptop --cpu tiger-lake` →
  `ocvalidate clean`; `kextorder.py` → no ordering problems; `verify.py` →
  **179/179 catalogue hashes intact**; detector spot-checks:
  `i7-1165G7`/`i5-1135G7` → `tiger-lake`, `i7-1065G7` → `ice-lake`,
  desktop `i7-11700` → `rocket-lake`, laptop `i7-12700H` → `None`.

## In progress

- Nothing open. Next session: pick from "Pending".

## Local release build (2026-09-09, Windows x64, NOT tagged/pushed)

Artifacts in `C:\Users\Win\AppData\Local\Temp\opencode\release-dist\`
(deliberately outside the repo — `build/`, `gui/bin/`, `gui/obj/` are
gitignored, working tree shows only the intended source changes):

| Artifact | Size | Check |
|---|---|---|
| `EFI-base.zip` | 35 MB | `release.py --out` |
| `configs.zip` | 0.8 MB | 179/179 configs |
| `engine/EFIBuilderEngine/` | — | PyInstaller 6.22.2 (pip-installed), `--check` smoke OK |
| `HackintoshEFIBuilder-win-x64.zip` | 76.1 MB | exe (36.7 MB, trimmed single-file) + `EFIBuilderEngine\` beside it, per RELEASING.md layout |

Notes: `verify.py --comments` → 179/179 before building. dotnet publish ran
on SDK 11.0.100-preview (SDKs 10/11 only on this machine; net8.0 target
resolved via NuGet) — warnings only, all pre-existing (Avalonia CS0108
`Content`/`Bounds` hiding, obsolete `DragEventArgs.Data`, IL2026 COM trim).
First publish attempt was interrupted mid-restore; re-run completed. GUI
window itself not smoke-tested (needs a display; CI uses xvfb + `--render`).
No tag created, nothing pushed — publishing/republishing (`v1.0.7` per
RELEASING.md, OpenCore hasn't moved) is a separate explicit decision.

## Pending tasks

1. **Tiger Lake catalogue entries** (maintainer publish decision): profile
   exists and builds ad hoc, but `catalogue.toml` has 0 Tiger Lake configs.
   Publishing = add entries + `verify.py --rehash`, review the diff.
2. **Boot verification on real Tiger Lake hardware**: the MOD parameters were
   read off a third-party EFI, never booted here. A successful boot report
   would justify a `data/field.toml` `igpu_id` entry (8086:9A49 →
   fake 499A0000 + platform 0000499A, `observed_by` whoever boots it).
3. **NootedGreen vendoring decision**: needed for accelerated Xe; licensing +
   size call, same as the RealtekCardReader precedent (documented, not
   shipped). Until decided, the manual 3-step path in the profile header
   stands. If vendored, profile gains the kext + real `device-id`.
4. **Laptop Alder Lake (12th) / Raptor Lake (13th–14th) / Core Ultra**:
   detection returns `None` by design; profiles + detector rows + labels +
   gpu rows when sources exist. Desktop side already covers Alder/Raptor.
5. **Full-installer downloads (gibMacOS technique)**: optional
   `tools/fullinstaller.py` reusing the sucatalog method (see SKILLS.md).
   Must stay opt-in/network-on-request per the offline rule; never bundle
   Apple images.
6. **USB enumeration hardening (WinToGo technique)**: borrow the
   SetupAPI + `IOCTL_STORAGE_QUERY_PROPERTY` BusTypeUsb/RemovableMedia
   pattern for `stick.py --list` on Windows (see SKILLS.md). Do NOT port the
   GPT/MSR/VHDX/bcdboot partitioning flow — wrong target for a FAT32
   EFI+recovery stick.

## Decisions taken

- **NootedGreen stays out of the tree** for now: unvendored third-party kext
  with no license/size review done; profile references vendored kexts only
  (a missing `BundlePath` only warns, but shipping a config that silently
  lacks acceleration contradicts the honesty rule). Manual steps documented
  in the profile header instead.
- **MOD's `MaLd0n.aml` NOT copied**: single merged SSDT written against one
  machine's tables; generics + SSDTTime flow kept per the SSDT policy.
- **`field.toml` untouched**: nothing here booted Tiger Lake; an entry
  without `observed_by` + specific observation would be exactly the invented
  verdict the repo refuses.
- **gpu verdict for `tiger-lake` = `unsupported`** (Dortania's wording,
  models listed): builder consequently offers the no-accel fallback — the
  correct behavior until NootedGreen is vendored.
- **SKILLS.md created at root** per request ("guarda los SKILLS.md"): holds
  the reusable external techniques so future sessions don't re-derive them.

## Known issues

- Catalogue has no Tiger Lake entries → guided/setup ad-hoc builds only.
- Laptop 12th gen+ (and Core Ultra, Xeon, Pentium, Apple silicon) detect as
  nothing — by design, but users will ask; the pending profiles above are
  the answer.
- `EFI.Opencore.NoteBook.TigerLake.MOD` contains `__MACOSX/` + `.DS_Store`
  noise and a config without `ToggleSipEntry.efi`; used as parameter source
  only, never merged.

## External resources (outside the repo, do not move)

| Path | What it is | Verdict |
|---|---|---|
| `E:\PROGRAMAS\DISCOS\Mac_OS_X\Nuevo\EFI.Opencore.NoteBook.TigerLake.MOD` | Working Tiger Lake laptop EFI (OC, SMBIOS MBP16,2, NootedGreen, device-id 499A0000, platform 0000499A, merged MaLd0n.aml) | Parameters extracted → profile header + SKILLS.md; SSDT/AML not imported |
| `E:\Usuarios\Tony\Downloads\Programs\gibMacOS-master` (corpnewt, MIT) | Full macOS installer downloader via `swscan.apple.com` sucatalogs + `InstallAssistantPackageIdentifiers` parsing | Reusable catalog technique for an opt-in full-installer tool; complements (never replaces) `macrecovery`-based recovery flow |
| `E:\Usuarios\Tony\OneDrive\Documents\GitHub\WinToGoCreator` (C/Win32, GPL-3.0) | Windows-To-Go USB creator: GPT+EFI/MSR/NTFS+VHDX-fixed via diskpart/VirtDisk/wimlib/DISM/bcdboot | Enumeration technique worth borrowing; partitioning flow NOT applicable (different target layout + GPL-3.0 vs repo licensing — check before copying code, not just ideas) |

## Recommended next steps

1. Boot-test the Tiger Lake build on real 11th-gen-mobile hardware (with
   manual NootedGreen steps) → file the `field.toml` entry.
2. Decide NootedGreen vendoring → update profile + `kexts.lock` +
   `licences.toml` + catalogue entries via `--rehash`.
3. If a full installer is wanted on Windows/Linux: prototype the gibMacOS
   catalog reader as a separate opt-in tool (network only when asked).
4. Run the full `selftest.py` before any push (this session ran build +
   verify + kextorder only).
