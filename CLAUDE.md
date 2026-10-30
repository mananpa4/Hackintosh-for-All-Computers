# CLAUDE.md — Hackintosh-for-All-Computers

> Permanent project knowledge. Stable architecture, tech, layout, commands and
> working rules. **Before any important change, read `CLAUDE_CONTEXT.md` first**
> — it holds the live state (what is done, what is in progress, what was
> decided and why). After finishing important work, update it.

## What this is

A program that installs macOS on PC hardware. It reads the machine in front
of it, works out which OpenCore EFI folder that machine needs, writes it,
fetches Apple's recovery installer (~700 MB), formats the USB stick and
copies both onto it. No .NET/Python runtime needed by end users: each release
carries the window (Avalonia GUI) plus the builder (frozen Python engine).

Upstream: `yusufklncc/Hackintosh-for-All-Computers`. OpenCore pinned per
release (currently 1.0.7). 179 published configs generated from 62 profiles;
766 devices in 8 categories; 42 vendored kexts from 19 projects.

## Architecture — one source of truth

```
Python engine (tools/, profiles/, data/, EFI/, vendor/)
   ^  everything is decided here; stdlib only, offline by default
   |
C# GUI (gui/) — draws what the engine answers, holds no hardware rules
Docs site (guide/) — MkDocs Material, EN + TR, built from guide/
```

The GUI never reimplements logic: it spawns the engine (`EFIBuilderEngine`
beside the binary, `../Resources/EFIBuilderEngine` on macOS, or
`tools/setup.py` + python3 in a checkout) and speaks a JSON-lines protocol
(`hello/text/ask/prompt/built/done/fatal`). A screen can never say something
the console would not.

Config generation is layered merging (`tools/ocgen.py`):

```
Sample.plist (vendor/opencore/<ver>/)
 -> profiles/base.toml
 -> profiles/platform/<desktop-intel|desktop-amd|laptop>.toml
 -> profiles/cpu/<plat>/<gen>.toml [+ <gen>.<n>core.toml override]
 -> profiles/overlay/chipset.<v>.toml -> overlay/oem.<v>.toml
 -> overlay/variant.<v>.toml (fixed order: chipset, oem, variant)
 -> overlay/combo/<tag>.toml -> config/<slug>.toml (residuals)
```

Dicts merge recursively; lists/scalars replace wholesale. Identity
(serial/MLB/UUID/ROM) is never stored — `macserial` mints it per build.
`profiles/catalogue.toml` pins the hash of every published config; the
equivalence gate (`tools/verify.py`) proves a profile change did not alter
what anyone downloads.

## Tech stack

| Part | Stack |
|---|---|
| Engine | Python 3.11+, **standard library only** (`tomllib`, `plistlib`, …). No `requirements.txt` in `tools/` |
| GUI | C# .NET 8, Avalonia 11 (Fluent), trimmed single-file self-contained publish |
| Docs site | `mkdocs-material==9.7.7` + `mkdocs-static-i18n==1.3.1` (`guide/requirements.txt`) |
| Formats | TOML profiles, OpenCore plist (`hex:`/`date:` tags in `ocgen.py`), JSON reports |
| Detection | OS-native, no deps, no admin: PowerShell CIM (Win), `lspci`+sysfs (Linux), `system_profiler`+`sysctl` (macOS) |

## Layout

```
tools/        engine: setup.py (guided flow), build.py, detect.py, ocgen.py,
              verify.py, recovery.py, stick.py, gpu.py, audio.py, smbios.py,
              *_table.py (regenerate data/*.toml), selftest.py, … (51 files)
              tools/README.md is the engine manual — read it before touching tools/
profiles/     base.toml, catalogue.toml (179 hashes), support.toml,
              platform/ (3), cpu/desktop-intel/ (22), cpu/desktop-amd/ (2),
              cpu/laptop/ (14, incl. tiger-lake.toml), overlay/ (14+combo),
              config/ (6 residuals)
data/         generated tables: hardware/gpu/audio/framebuffer/network/smbios/
              … + field.toml (boot observations) + candidates.toml (not shipped)
EFI/          payload: BOOT/, OC/ (ACPI, Drivers, Kexts, Tools, Resources)
vendor/       pinned: opencore/<ver>/ (Sample.plist, ocvalidate/macserial/
              macrecovery+boards.json ×3 hosts), kexts.lock, tools.lock,
              tools/ (SSDTTime, iasl, USBToolBox/Windows.exe)
gui/          Shell.csproj, Program.cs, App.axaml, Engine/ (Builder, Session,
              Machine, Inventory, Reveal), Views/ (9 panes + MainWindow)
guide/        14 pages × EN/TR (suffix pairs: bios.md + bios.tr.md) + assets/
docs/         maintainer notes (RELEASING.md — OpenCore bump + publish flow)
macOS/        release icons for GUI/web tiles (no installers, ever)
.github/      workflows: validate, gui, windows-exe, release, refresh, guide
SKILLS.md     reusable external techniques (gibMacOS catalogs, USB enumeration,
              Tiger Lake MOD parameters, CPU detector rules)
```

CPU coverage (laptop): up to Tiger Lake (11th gen mobile). Laptop Alder
Lake/12th gen+ has intentionally no profile yet — detection returns nothing
rather than a guess. Desktop covers through Raptor Lake.

## Commands

```bash
# guided build / inspection
python3 tools/setup.py
python3 tools/setup.py --check
python3 tools/summary.py --machine machine.json
python3 tools/setup.py --machine machine.json
python3 tools/detect.py --report machine.json

# ad-hoc build (any cpu profile, no catalogue entry needed)
python3 tools/build.py --platform laptop --cpu tiger-lake
python3 tools/build.py --platform laptop --cpu kaby-lake --oem hp
python3 tools/build.py --catalogue | --list

# equivalence gate (run after ANY profile/data/vendor change)
python3 tools/verify.py --comments
python3 tools/verify.py --rehash     # rewrites catalogue hashes; the diff is the review

# kexts / tables
python3 tools/kexts.py check
python3 tools/kexts.py outdated      # network: asks GitHub what is newer
python3 tools/kextorder.py [config.plist]

# recovery (ONLY network use on a user's behalf) / stick
python3 tools/recovery.py --list
python3 tools/recovery.py --macos 12.7.6 --out /Volumes/USB
python3 tools/stick.py --list
python3 tools/stick.py --place /Volumes/USB --efi build/EFI --recovery .
python3 tools/stick.py --prepare disk4        # erases it

# full test suite (slow, thorough)
python3 tools/selftest.py

# GUI (dotnet CLI is primary; VS 2022/2026 Enterprise also installed at
# F:\Program Files\Microsoft Visual Studio\18\Enterprise if ever needed)
dotnet build gui/Shell.csproj -c Release
dotnet publish gui/Shell.csproj -r win-x64 --self-contained true \
  -p:PublishTrimmed=true -p:PublishSingleFile=true

# docs site
pip install -r guide/requirements.txt
mkdocs serve            # localhost:8000
mkdocs build --strict
python3 tools/guidecheck.py _site
```

## Conventions (non-negotiable)

1. **Offline by default.** A clone contains everything a build needs.
   Network only when asked (`recovery.py`, `itlwm` fetch, `*-table.py
   --refresh`, `kexts.py outdated`, `smbios.py --refresh`).
2. **Never preselect.** Detection is shown as `detected` beside the question;
   the person always types a number. A wrong ticked answer is never rechecked.
3. **Merge semantics.** Dicts merge, lists replace, `EMPTY_MAPS` cleared
   before merging, `hex:`/`date:` TOML tags, canonical sorted-XML bytes for
   hashing. `verify.py` distinguishes `True` vs `1` vs `1.0`.
4. **Stick safety.** The removable-disk list is the gate: `prepare()` refuses
   anything it did not offer, re-reads at erase time, boot disk excluded by
   its own identifier. `--place` (copy) and `--prepare` (erase) stay separate
   flags. Typing the disk name is anti-surprise, not the guard.
5. **Honesty over coverage.** `unknown` beats an invented verdict. Unverifiable
   claims stay out of `data/`; one-off boot facts go to `field.toml` with
   `observed_by`, and outrank generation rules because they are more specific.
   `provenance.py` labels every answer derived/quoted/measured/none.
6. **Generated tables are parsed, not retyped.** `*_table.py` read upstream
   files; regenerating must produce a reviewable diff. If you hand-edit a
   generated file (`data/*.toml`), make the identical edit in its generator
   so the next `--refresh` reproduces it.
7. **No Apple software is redistributed.** Recovery is fetched onto the
   asker's machine; `recoverycheck.py` deletes what it verifies. Same reason
   `vendor/ocbinarydata.lock` records unlicensed drivers instead of vendoring
   them.
8. **GUI holds no rules.** New capability = engine first (`--protocol` event),
   then a view. No kext names, versions or macOS lists in C#.

## Working rules for Claude

- **Always read `CLAUDE_CONTEXT.md` before important changes**, and update it
  when tasks complete or relevant decisions are taken. It is the live memory;
  this file is the stable reference — never store transient tasks here.
- Read `tools/README.md` before modifying the engine; it documents every
  subsystem (framebuffer ids, SMBIOS reach, SSDT flow, USB mapping,
  third-party licensing) better than any summary.
- After any `profiles/`, `data/` or `vendor/` change: run `verify.py
  --comments` and the representative `build.py` for the touched profile; run
  `kextorder.py` if `Kernel.Add` changed. `ocvalidate` must be clean.
- New CPU support checklist: `profiles/cpu/<plat>/<gen>.toml` (+ header
  comment with sources) → `detect.py` tables → `setup.py CPU_LABELS` →
  `gputable.py IGPU_GENERATIONS` + matching `data/gpu.toml` row → test build
  + `verify.py`. Catalogue entries are a maintainer publish decision, not
  automatic — ad-hoc/`--answers` builds work without them.
- Deterministic CI style: `tools/fixtures/no-hardware.json` + `--ids`,
  `--hda-ids`, `--nvme`, `--answers` for scripted runs; never offer SSDTTime
  or USBToolBox menus to `--answers` (they wait for keystrokes).
- Keep the bilingual docs in pairs: every `guide/*.md` needs its `*.tr.md`
  twin (`guidecheck.py` enforces fragments + pairs).
- Prefer editing existing files; do not create docs outside `docs/`,
  `guide/` and the root trio (`CLAUDE.md`, `CLAUDE_CONTEXT.md`, `SKILLS.md`,
  `README.md`) unless asked.
