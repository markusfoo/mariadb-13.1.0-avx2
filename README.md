<div align="center">

<img src="docs/logo.svg" width="560" alt="MariaDB Windows Builds" />

# MariaDB Windows Builds

**Always-fresh MariaDB binaries for Windows x64 — compiled in CI from every new upstream release,<br>in three instruction-set flavours: AVX2, AVX and a universal baseline.**

[![Build MariaDB](https://github.com/markusfoo/mariadb-windows-builds/actions/workflows/build.yml/badge.svg)](https://github.com/markusfoo/mariadb-windows-builds/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/markusfoo/mariadb-windows-builds?display_name=tag&include_prereleases&sort=semver)](https://github.com/markusfoo/mariadb-windows-builds/releases/latest)
[![Downloads](https://img.shields.io/github/downloads/markusfoo/mariadb-windows-builds/total?color=teal)](https://github.com/markusfoo/mariadb-windows-builds/releases)
[![Platform](https://img.shields.io/badge/platform-Windows%2010%2B%20x64-6c757d)](#-which-build-should-i-pick)
[![Binaries license](https://img.shields.io/badge/binaries-GPL--2.0-brightgreen)](https://mariadb.org/about/licensing/)

[Releases](https://github.com/markusfoo/mariadb-windows-builds/releases) · [How it works](#-how-it-works) · [XAMPP guide](#-installing-into-xampp) · [Troubleshooting](#-fixing-incorrect-definition-of-table-mysqlevent-xampp) · [FAQ](#-faq)

</div>

---

## ✨ What this repository does

This repository turns the **latest MariaDB source release** into ready-to-run **Windows x64 binaries**, automatically:

- 🔄 **Version-agnostic, never goes stale.** A version checker resolves the newest upstream release *at run time* (rolling/preview tarballs included — those are not even git-tagged upstream). When MariaDB 13.2, 14, 15 … ships, this repo builds it **without a single change**.
- 🧮 **Three codegen flavours per release** — `AVX2`, `AVX` and a universal SSE2 baseline, so there is a build for every CPU made since 2003.
- 📦 **Portable ZIP packages** — an unpacked, ready-to-run tree (`bin/`, `data/`, `share/`, `lib/` …). No installer needed; drop-in friendly for XAMPP. An MSI can be produced on demand.
- ✅ **Smoke-tested before publishing** — every package is booted in CI, queried, and shut down cleanly. A build that doesn't start never reaches the Releases page.
- 🔐 **SHA-256 verified sources & published checksums** — tarballs are checked against MariaDB's official checksum API; every release ships `SHA256SUMS.txt`.
- 🗓️ **Fully automatic** — a weekly check (Mon 02:30 UTC) picks up brand-new upstream releases; manual dispatch can build any series (`11.8`, `12.3`, …) or exact version at any time.
- 🩺 **Debug symbols included** — `*-debugsymbols.zip` with PDBs accompanies every flavour.

> [!NOTE]
> **Unofficial builds.** Binaries are compiled from pristine upstream sources, but this project is not affiliated with the MariaDB Foundation. Preview/RC series are marked as pre-releases here — for production prefer the [LTS releases](https://github.com/markusfoo/mariadb-windows-builds/releases) (e.g. MariaDB 11.8 / 12.3).

---

## 🧭 Which build should I pick?

| Asset suffix | Instruction set | Works on | MSVC flag |
|---|---|---|---|
| `avx2` | AVX2 + FMA + BMI (VEX-encoded SIMD) | Intel Haswell (2013)+ · AMD Zen+ · all modern servers | `/arch:AVX2` |
| `avx` | AVX1 | Intel Sandy/Ivy Bridge (2011–2013) · some VPS | `/arch:AVX` |
| `generic` | x86-64 baseline (SSE2 only) | **any** x64 CPU or VM, incl. pre-2011 hardware | *(none)* |

<details>
<summary><strong>How do I check what my CPU supports?</strong></summary>

Download [Sysinternals Coreinfo](https://learn.microsoft.com/en-us/sysinternals/downloads/coreinfo) and run:

```
coreinfo -v
```

Look for `AVX2 *` (asterisk = supported). If you see `AVX2 -` but `AVX *`, take the `avx` build. In doubt → `generic` always works. Note that a *VM* may hide host CPU features unless passthrough is enabled.
</details>

> [!IMPORTANT]
> Everything inside an AVX2/AVX package (server, clients, plugins) requires that instruction set — a binary copied to a CPU without AVX2 will crash with `0xC000001D` (illegal instruction). When in doubt, use `generic`.

---

## 📦 What's inside a release

Every release `vX.Y.Z` contains one portable ZIP per flavour, plus symbols and checksums:

| Asset | Contents |
|---|---|
| `mariadb-X.Y.Z-winx64-avx2.zip` | full unpacked distribution: `bin\mariadbd.exe` (+ `server.dll`), `mariadb.exe`, `mariadb-admin.exe`, `mariadb-dump.exe`, all storage-engine/client plugins as DLLs, `share\`, `lib\`, pristine `data\` |
| `mariadb-X.Y.Z-winx64-avx.zip` | same, `/arch:AVX` |
| `mariadb-X.Y.Z-winx64-generic.zip` | same, SSE2 baseline |
| `mariadb-X.Y.Z-winx64-*-debugsymbols.zip` | matching PDB debug symbols |
| `SHA256SUMS.txt` | checksums of every asset |
| `BUILD_INFO.txt` | exact version, source tarball, compiler, flags, runner |
| `mariadb-X.Y.Z-winx64-*.msi` *(optional)* | WiX installer — only when [built with the MSI switch](#-faq) |

Releases for **preview/RC** upstream versions (e.g. the 13.1 rolling series) are flagged as GitHub *pre-releases*; the `Latest` badge always points to the newest stable/LTS build.

---

## 🪟 Installing into XAMPP

Works with any XAMPP that ships MariaDB (e.g. `C:\xampp\mysql`). Adjust paths to your layout.

1. **Stop MySQL/MariaDB** in the XAMPP Control Panel and **back up your data directory** (`C:\xampp\mysql\data` → `data_backup`).
2. Download the flavour you want from [Releases](https://github.com/markusfoo/mariadb-windows-builds/releases) and unzip it.
3. Copy the zip's `bin`, `share`, `lib` (and `data` only for a fresh start) over `C:\xampp\mysql\` — replace existing files.
4. If your XAMPP panel expects the classic name, copy `bin\mariadbd.exe` to `bin\mysqld.exe` (both work identically).
5. Start MariaDB. If it starts clean → done. If you see `Incorrect definition of table mysql.event` errors → follow the fix below **before** using the server.

---

## 🩹 Fixing `Incorrect definition of table mysql.event` (XAMPP)

You replaced old XAMPP binaries with a new MariaDB and the log shows:

```
[ERROR] Incorrect definition of table mysql.event: expected column 'definer' at position 3 to have type varchar(, found type char(141).
[ERROR] mariadbd.exe: Event Scheduler: An error occurred when initializing system tables.
[ERROR] mariadbd.exe: An error occurred when loading data from the table mysql.event. System triggers not loaded
[ERROR] Aborting
```

**This is not a build defect.** The binaries are fine — the *data directory* is old. XAMPP ships a much older MariaDB series, whose system tables (`mysql.event`, `mysql.user`, …) predate the current schema. New MariaDB auto-upgrades system tables on first start, but when the old layout is too far behind (or partially patched), the server **refuses to start rather than risk corrupting your privileges**. Official binaries behave exactly the same.

### Option A — upgrade in place (keeps your databases, users, events)

From your MariaDB `bin` folder (e.g. `C:\xampp\mysql\bin`), with XAMPP's MySQL stopped:

```bat
:: 1) start a temporary server that tolerates the old system tables
mariadbd.exe --console --skip-grant-tables --event-scheduler=DISABLED

:: 2) in a SECOND terminal, same folder — run the system-table upgrade
mariadb-upgrade.exe --force -h 127.0.0.1 -P 3306 -u root

:: 3) shut the temporary server down
mariadb-admin.exe -h 127.0.0.1 -P 3306 -u root shutdown

:: 4) start normally (XAMPP panel or: mariadbd.exe --console)
```

`mariadb-upgrade.exe` ships in every package published here. It rewrites `mysql.*` tables to the current schema, creates the missing `event` columns, installs the system triggers and repairs `mysql.gtid_slave_pos` ("doesn't exist in engine").

### Option B — fresh data directory (guaranteed, when A fails)

1. Rename `C:\xampp\mysql\data` → `data_old`.
2. Copy the **pristine `data\` folder from the release ZIP** into `C:\xampp\mysql\`.
3. Start MariaDB — it boots instantly with a clean system database.
4. Re-import your data: `mariadb.exe -u root < your_dump.sql` (if you have a dump from the old install; otherwise boot the *old* XAMPP binaries against `data_old` once and `mariadb-dump` it).

> [!TIP]
> Jumping many major series at once (e.g. XAMPP's 10.x → 13.x) is outside MariaDB's supported in-place upgrade path. For anything precious: dump with the old binaries first, then load into a fresh directory (Option B). This repo's zips always include a ready-to-use `data\` for exactly that.

---

## ⚡ What to expect performance-wise

Honest numbers, no marketing:

- **OLTP / general queries:** ±0–3% — InnoDB's hot paths (CRC32C checksums via PCLMUL, memcpy) are *runtime-dispatched* already and don't change.
- **Hash & SIMD-friendly paths:** the real AVX2 payoff — xxHash (XXH3/XXH64/XXH128) compiles to a true AVX2 backend, helping hash joins, GROUP BY hashing, bulk loads, checksums.
- **Haswell/Broadwell (2013–2015):** minor AVX2 turbo-clock penalty possible under sustained SIMD load; typically still net-positive.
- Whole-package consistency: server *and* all plugins/clients share one instruction set — no mixed-flavour crashes.
- Want more? PGO+LTCG is on the roadmap (see [FAQ](#-faq)) — benchmark *your* workload before committing.

---

## 🏗️ How it works

```
 ┌──────────────┐   ┌───────────────────────────────┐   ┌──────────────────┐
 │   resolve    │──▶│  build (3-way matrix, ~25 m)  │──▶│     release      │
 │              │   │                               │   │                  │
 │ mirror index │   │ download+verify source        │   │ tag vX.Y.Z       │
 │ rest-api     │   │ bison bridge · cmake · MSVC   │   │ notes+checksums  │
 │ dedupe check │   │ /arch:AVX2|AVX|— (RwDInfo)    │   │ upload assets    │
 └──────────────┘   │ win_package → ZIPs            │   └──────────────────┘
                    │ flags audit · smoke boot      │
                    └───────────────────────────────┘
```

1. **`resolve`** — the *version checker*: scrapes the MariaDB mirror index (rolling releases like 13.1.x exist only as tarballs there, **not** as git tags) and the [downloads.mariadb.org REST API](https://downloads.mariadb.org/rest-api/mariadb/) for status, release name and the official SHA-256. If a release for the newest version already exists here (and `force` is not set), the run exits green without wasting compute.
2. **`build`** — a 3-job matrix (AVX2 / AVX / generic) on `windows-latest`: downloads and checksum-verifies the source, sets up the Bison toolchain, configures with CMake (Visual Studio generator auto-detected, x64), compiles `RelWithDebInfo` with the flavour's `/arch:` flag, runs the official `win_package` target, audits the recorded compiler flags, then **boots the packaged server and answers a query** before anything is uploaded.
3. **`release`** — creates tag `vX.Y.Z` and a GitHub Release with all flavours, `*-debugsymbols.zip`, `SHA256SUMS.txt` and `BUILD_INFO.txt`; Preview/RC versions are flagged as pre-releases.

**Triggers:**

| Event | Behaviour |
|---|---|
| `schedule` — Mondays 02:30 UTC | builds whenever a brand-new upstream release appears |
| `workflow_dispatch` | manual: `auto`, a series (`11.8`, `12.3`, …), or an exact version; options `force` (rebuild) and `build_msi` |
| `push` to the workflow file | re-validates the pipeline (deduped → no-op) |

---

## 🔁 Build it yourself (locally)

Prerequisites: Visual Studio 2022+ with *Desktop development with C++*, CMake ≥ 3.25, [Chocolatey](https://chocolatey.org).

```powershell
# Bison (MariaDB's Windows build needs it; winflexbison + a name bridge)
choco install -y winflexbison3
$t = 'C:\ProgramData\chocolatey\lib\winflexbison3\tools'
Copy-Item "$t\win_bison.exe" "$t\bison.exe"; Copy-Item "$t\win_flex.exe" "$t\flex.exe"

# Source (adjust version)
$v = '13.1.0'
curl.exe -fLo src.tar.gz "https://archive.mariadb.org/mariadb-$v/source/mariadb-$v.tar.gz"
tar -xzf src.tar.gz; mv mariadb-$v src; cd src

# Configure + build (pick your flavour flag: /arch:AVX2, /arch:AVX, or none for generic)
cmake -S . -B bld -A x64 `
  -DCMAKE_BUILD_TYPE=RelWithDebInfo `
  "-DCMAKE_C_FLAGS_RELWITHDEBINFO=/O2 /Ob1 /DNDEBUG /arch:AVX2" `
  "-DCMAKE_CXX_FLAGS_RELWITHDEBINFO=/O2 /Ob1 /DNDEBUG /arch:AVX2" `
  "-DBISON_EXECUTABLE=$t\bison.exe" "-DFLEX_EXECUTABLE=$t\flex.exe"
cmake --build bld --config RelWithDebInfo --parallel
cmake --build bld --config RelWithDebInfo --target win_package
```

`win_package` leaves two zips in `bld\` — the portable distribution and debug symbols.

---

## ❓ FAQ

<details><summary><b>Is this an official MariaDB distribution?</b></summary>

No. Sources are pristine upstream tarballs (SHA-256 verified against MariaDB's checksum API), but the compilation, packaging and hosting happen in this repo's CI. For vendor-supported binaries use [mariadb.com downloads](https://mariadb.com/downloads).
</details>

<details><summary><b>Why is 13.1.x marked as a pre-release?</b></summary>

Upstream status: the 13.1 series is *Preview* (rolling), 13.0 is *RC* — the release checker reads this from the MariaDB downloads API and flags accordingly. The GitHub `Latest` marker therefore always points at a stable/LTS build (11.8 / 12.3).
</details>

<details><summary><b>How do I force a rebuild or build an older/other version?</b></summary>

*Actions → Build MariaDB for Windows x64 → Run workflow*: set **source_ref** to `auto`, a series like `11.8`, or an exact version like `13.1.0`, and tick **force** to rebuild a version that already has a release. Assets are then replaced (`--clobber`).
</details>

<details><summary><b>Can I get an MSI installer?</b></summary>

Yes — run the workflow manually with **build_msi** ticked. It is best-effort (WiX 3) and uploaded alongside the ZIPs when it succeeds. ZIPs remain the recommended format (XAMPP-friendly, no install needed).
</details>

<details><summary><b>Why does <code>mariadbd --version</code> say “Source distribution”?</b></summary>

That's the standard label for any build compiled from the source tarball rather than packaged by MariaDB's own release engineering. It is cosmetic; the CI smoke test verifies the actual server behavior.
</details>

<details><summary><b>Will there be AVX-512 / PGO builds?</b></summary>

Possibly — the matrix is one line per flavour. AVX-512 needs both MSVC support and enough real CPUs to matter; PGO+LTCG (`/GL /LTCG`) is the more promising next step for single-digit percentage gains.
</details>

<details><summary><b>My antivirus flags the download…</b></summary>

Unofficial unsigned binaries occasionally trip heuristics. Every asset has its SHA-256 in `SHA256SUMS.txt` — verify with <code>certutil -hashfile &lt;file&gt; SHA256</code> and compare.
</details>

---

## ⚖️ License, sources & support

- MariaDB is **GPL-2.0**; binaries built here inherit it. Sources for every release: the exact upstream tarball, linked by version and checksum in each release's notes (e.g. `archive.mariadb.org/mariadb-<version>/source/`).
- This repository's own scripts and docs are MIT (see [LICENSE](LICENSE)).
- No warranty. Preview series (13.x) are development rolling releases — treat them as such.
- Bugs in MariaDB itself → [MariaDB Jira](https://jira.mariadb.org); build/pipeline issues here → [Issues](https://github.com/markusfoo/mariadb-windows-builds/issues).

---

<div align="center">

**⭐ Star this repo if the builds are useful — it helps others find them.**

Built with ❤️ on GitHub Actions · [Releases](https://github.com/markusfoo/mariadb-windows-builds/releases) · [Pipeline](https://github.com/markusfoo/mariadb-windows-builds/actions/workflows/build.yml)

</div>
