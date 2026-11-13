# Spiteful + OriginSU + SuSFS — Redmi Note 10 Pro (`sweet`)

[![Build](https://github.com/1992Ade/sm6150-origin-builder/actions/workflows/build.yml/badge.svg)](https://github.com/1992Ade/sm6150-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sm6150-origin-builder?label=release&logo=github)](https://github.com/1992Ade/sm6150-origin-builder/releases/latest)

> **Your warranty is now void.** Flashing a custom kernel can prevent your
> phone from booting. You alone are responsible for what you flash: **always**
> back up your `boot` and `dtbo` partitions first.

## 📱 About

This repository **automatically** builds (GitHub Actions) a flashable
[AnyKernel3](https://github.com/osm0sis/AnyKernel3) zip for the
**Redmi Note 10 Pro / Pro Max** (`sweet` / `sweetin`, Snapdragon 732G).

- **Kernel:** the **Spiteful kernel** by [tbyool](https://github.com/tbyool/android_kernel_xiaomi_sm6150)
  (Linux **4.14**, branch `17.0`), the kernel shipped by the
  [Project Infinity-X](https://projectinfinity-x.com/downloads/sweet) ROM.
- **Root:** **[OriginSU](https://github.com/samakshkambxj/OriginSU)**, a
  kernel-based root solution (KernelSU ← ReSukiSU ← SukiSU-Ultra fork),
  **without KPM**.
- **Root hiding:** **[SuSFS](https://gitlab.com/simonpunk/susfs4ksu)**
  (simonpunk), backported to 4.14 by
  [JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd).
- **Fully automated:** every Sunday the workflow checks whether a new OriginSU
  version is out; if so, it rebuilds the kernel and publishes a GitHub Release
  on its own.

This repository contains **no copy** of the kernel, OriginSU or SuSFS: they are
downloaded from their official repositories at build time, then patched,
built and packaged.

The approach is inspired by
[1992Ade/msm8937-origin-builder](https://github.com/1992Ade/msm8937-origin-builder)
and [riarumoda/perf_neon-builder](https://github.com/riarumoda/perf_neon-builder)
(which already builds this same sm6150 kernel for sweet).

This kernel is intended to work with this **[Project Infinity X]([https://gitlab.com/simonpunk/susfs4ksu](https://projectinfinity-x.com/downloads/sweet))** ROM that uses Spiteful kernel already.

## 📦 Which zip should I download?

Each release ships two variants:

| File | Hook mode | Recommended for |
|---|---|---|
| `Spiteful-OriginSU-<version>-susfs-sweet-<date>.zip` | SuSFS inline hook + SuSFS | **Most users** — root + SuSFS root hiding |
| `Spiteful-OriginSU-<version>-manual-sweet-<date>.zip` | Manual hook, no SuSFS | Fallback if the SuSFS build doesn't boot |

The full `.config` of each variant (`config-*.txt`) and the exact source
revisions (`SOURCE-*.txt`) are attached to the release.

**Requirement:** a ROM that runs on this sm6150 kernel (Spiteful / crDroid
`17.0`), for example Project Infinity-X.

## 🔧 Installation

0. **Coming from Magisk, APatch, KernelSU, SukiSU, BakaSU…?** Read
   [this section](#️-coming-from-another-root-solution) first.
1. Back up `boot` and `dtbo` (from OrangeFox/TWRP, or keep your ROM's
   `boot.img`/`dtbo.img`).
2. Download the zip from the [latest release](https://github.com/1992Ade/sm6150-origin-builder/releases/latest).
3. Flash it from recovery (or `adb sideload <file>.zip`).
4. Reboot.
5. Install the **OriginSU manager** (`arm64-v8a` or `universal` APK) from the
   [OriginSU releases](https://github.com/samakshkambxj/OriginSU/releases) —
   use the **same version** as the kernel.

The zip leaves the ramdisk untouched: it replaces the kernel (`Image.gz`) and
the `dtb` inside `boot`, then flashes `dtbo`. It refuses to install on any
device other than `sweet` / `sweetin`.

### ↩️ Restore the stock kernel

```sh
adb shell su -c 'rm -rf /data/adb/*'    # remove root data first
adb reboot bootloader
fastboot flash boot boot_backup.img
fastboot flash dtbo dtbo_backup.img
fastboot reboot
```

## 🚀 Setting up your repository

1. Create the repository and push these files:
   ```bash
   git init -b main
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/1992Ade/sm6150-origin-builder.git
   git push -u origin main
   ```
2. **Settings → Actions → General**: make sure Actions are enabled and that
   **Workflow permissions** is set to **Read and write permissions** (needed
   to publish releases).
3. **Actions** tab → *Build sweet kernel (Spiteful + OriginSU + SuSFS)* →
   **Run workflow** once to create the first release.

> GitHub **disables scheduled workflows after 60 days without repository
> activity**. If automatic releases stop, re-enable the workflow in the
> Actions tab or push a commit.

### Schedule and update policy

| Trigger | Behaviour |
|---|---|
| Every Sunday at 04:23 UTC | Builds **only** if a new OriginSU version warrants it (see `UPDATE_POLICY`) |
| Manual (*Run workflow*) | Always builds; an existing release for the same version is **replaced** (useful to pick up kernel or SuSFS fixes) |

`UPDATE_POLICY`, at the top of [`.github/workflows/build.yml`](.github/workflows/build.yml):

| Value | Rebuilds when OriginSU goes from… |
|---|---|
| `major` | v4.x.x → v5.0.0 (major versions only) |
| `minor` *(default)* | v4.3.x → v4.4.0 or v5.0.0 (major and minor versions) |
| `all` | v4.3.0 → v4.3.1 (every version, patch releases included) |

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | *(latest release)* | OriginSU tag to build |
| `variants` | `both` | `both`, `susfs` or `manual` |
| `toolchain` | `system` | `system` = Ubuntu 24.04 Clang 18, or the URL of a Clang `.tar.gz` |
| `publish` | ✅ | Publish (or replace) the GitHub Release |

## ⚙️ How the build works

```text
check ──► build (susfs)  ──┐
      └─► build (manual) ──┴──► release
```

1. **check** — reads the latest OriginSU release, compares it with this
   repository's latest `originsu-<tag>` release according to
   `UPDATE_POLICY`, and pins the SuSFS patch repository commit for the whole run.
2. **build** (×2 in parallel), through `scripts/build.sh`:
   1. **sync** — clones tbyool's kernel (`17.0`) and applies your optional patches from `patches/`.
   2. **originsu** — clones OriginSU at the requested tag and runs its official
      `kernel/setup.sh` (`drivers/kernelsu` link, `Makefile`, `Kconfig`).
   3. **susfs** *(susfs variant)* — applies `susfs_patch_to_4.14.patch`
      (adds `fs/susfs.c`, `include/linux/susfs*.h` and the VFS changes); the
      build stops if any hunk is rejected.
   4. **hooks** — `scripts/patch_originsu_hooks.py` inserts the hooks for the
      selected mode, backports `path_umount()`, fixes OriginSU for 4.14 (see
      below), then repeats the checks done by OriginSU's `Kbuild`.
   5. **config** — `sweet_defconfig` + `configs/originsu-common.config` +
      `configs/hook-<mode>.config`; fails if `CONFIG_KSU`, the hook mode or the
      absence of KPM are not honoured.
   6. **kernel** — full LLVM build (`LLVM=1 LLVM_IAS=1`, ThinLTO).
   7. **zip** — AnyKernel3 with `anykernel/anykernel.sh`, licences and `SOURCE.txt`.
3. **release** — publishes `originsu-<tag>` with both zips, their SHA-256
   sums, the `.config` files and release notes (OriginSU and SuSFS versions,
   kernel commit, date). It is published even if only one variant built.

### Inserted hooks

| File | **susfs** variant (SuSFS inline hook) | **manual** variant |
|---|---|---|
| `fs/exec.c` | `ksu_handle_execveat[_sucompat]()` + post-exec | `ksu_handle_execveat()` |
| `fs/open.c` | `ksu_handle_faccessat()` (through `filename_lookup`) | `ksu_handle_faccessat()` |
| `fs/stat.c` | `ksu_handle_stat()` in `vfs_statx`, `ksu_handle_vfs_fstat()` | `ksu_handle_stat()`, `ksu_handle_newfstat_ret()`, `ksu_handle_fstat64_ret()` |
| `fs/read_write.c` | `ksu_handle_sys_read()` | — (automatic LSM hook) |
| `drivers/input/input.c` | `ksu_handle_input_handle_event()` | — (automatic `input_handler`) |
| `kernel/sys.c` | `ksu_handle_setresuid()` | — (automatic LSM hook) |
| `kernel/reboot.c` | `ksu_handle_sys_reboot()` | `ksu_handle_sys_reboot()` |
| `fs/namei.c` | `filename_lookup()` made non-`static` | — |
| `fs/namespace.c` | `path_umount()` backport (Linux 5.9) | same |

The SuSFS hooks follow the logic of JackA1ltman's `susfs_inline_hook_patches.sh`
(hooks by simonpunk and backslashxx), rewritten in Python so they anchor on the
code instead of line numbers, and aligned with OriginSU's exact prototypes.

### OriginSU ↔ Linux 4.14 fixes

OriginSU is mostly tested on GKI kernels (5.10+). On this 4.14 kernel the
script automatically fixes (only when needed):

| Problem | Fix |
|---|---|
| `pr_*_ratelimited()` does not compile (4.14's `printk.h` does not pull `<linux/ratelimit.h>`) | add the include |
| Link error `undefined symbol: ksu_preinit_release` (`preinit.h` only included in GKI hook modes) | include `preinit/preinit.h` unconditionally |
| Functions called without a declaration (`is_init`, `is_zygote`, `mnt_want_write`, the SuSFS API…), hidden by `-Wno-implicit-function-declaration` | add the missing headers / prototypes |

## 🖥️ Building locally

```bash
sudo apt install bc bison flex cpio curl git zip patch python3 libssl-dev libelf-dev \
                 clang lld llvm binutils-aarch64-linux-gnu binutils-arm-linux-gnueabi
./scripts/build.sh                       # susfs variant, all steps
HOOK_MODE=manual ./scripts/build.sh      # manual variant
./scripts/build.sh config kernel         # only some steps
```

Every variable at the top of `scripts/build.sh` can be overridden
(`ORIGINSU_REF=v4.3.0`, `NONGKI_REF=<commit>`, `SUSFS_PATCH=<file or URL>`,
`TOOLCHAIN=<url>`, `JOBS=8`, `SHIP_DTBO=0`…). Zips end up in `dist/`.

## 🗂️ Repository layout

```text
.
├── .github/workflows/build.yml     # check → build (susfs, manual) → release
├── anykernel/anykernel.sh          # AnyKernel3 script for sweet (non-A/B boot + dtbo)
├── configs/
│   ├── originsu-common.config      # OriginSU, multi-manager, KPM disabled
│   ├── hook-susfs.config           # SuSFS inline hook + SuSFS options
│   └── hook-manual.config          # Manual hook (fallback)
├── patches/README.md               # your optional kernel patches (*.patch)
├── scripts/
│   ├── build.sh                    # single build script (CI and local)
│   └── patch_originsu_hooks.py     # susfs/manual hooks + OriginSU 4.14 fixes
├── LICENSE                         # GPL-2.0
├── NOTICE.md                       # third-party components and licences
└── README.md
```

Zip contents: `Image.gz`, `dtb`, `dtbo.img`, `anykernel.sh`, `META-INF/` and
`tools/` (AnyKernel3), `LICENSE` (AnyKernel3), `COPYING.kernel`,
`LICENSE.OriginSU`, `LICENSE.OriginSU-kernel` and `SOURCE.txt`.

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| *Apply the SuSFS kernel patch* step fails | The SuSFS patch changed and no longer applies: re-run with `variants: manual`, or pin `NONGKI_REF` to a commit that worked |
| `[hooks] … FAILED: anchor not found` | The kernel code changed: adapt the anchor in `patch_originsu_hooks.py` or add a patch in `patches/` |
| `You lost … hook in your kernel` | A new OriginSU version requires an extra hook: compare with the [hooks guide](https://resukisu.github.io/guide/manual-hooks.html), or re-run with an older `originsu_ref` |
| Stuck on the boot logo | Flash the `-manual` variant, or restore your `boot`/`dtbo` backups |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict: see below |
| The manager says "not installed" | Check `uname -r` (must contain `-Spiteful-OriginSU`) and the manager version |
| No new automatic release | The scheduled workflow may have been disabled (60-day rule) |

Build logs (`build-log-*`) are attached to every run, even when it fails.
Pressing *volume down* at least 3 times during boot enables KernelSU safe mode
(modules disabled).

### ⚠️ Coming from another root solution?

Modules installed with Magisk, APatch, KernelSU, SukiSU, BakaSU… stay in
`/data/adb/modules` and will be loaded by OriginSU. OriginSU ships its own
Zygisk: another Zygisk (ReZygisk, NeoZygisk, Zygisk Next) or modules that rely
on one (LSPosed…) can conflict with it — typical symptom: the phone boots but
has **no Wi-Fi, no mobile network or no camera**.

Uninstall all modules, then reinstall them one by one:

```sh
adb shell
su
for m in /data/adb/modules/*/; do touch "$m/remove"; done
rm -rf /data/adb/modules_update/*
reboot
```

## 🙏 Credits

**Kernel**

- [Linus Torvalds](https://github.com/torvalds/linux) and all **Linux kernel** contributors.
- **Qualcomm / CodeLinaro (formerly CodeAurora)** for the msm-4.14 base, **Xiaomi** for the sweet kernel sources.
- The [**crDroid**](https://github.com/crdroidandroid/android_kernel_xiaomi_sm6150) team, whose repository tbyool's kernel is forked from.
- [**tbyool**](https://github.com/tbyool/android_kernel_xiaomi_sm6150) for the **Spiteful kernel**.
- [**Project Infinity-X**](https://projectinfinity-x.com/) and the ROM's sweet maintainer.

**Root and SuSFS**

- [**OriginSU**](https://github.com/samakshkambxj/OriginSU) — samakshkambxj.
- [**SuSFS / susfs4ksu**](https://gitlab.com/simonpunk/susfs4ksu) — simonpunk.
- [**NonGKI_Kernel_Build_2nd**](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) — JackA1ltman:
  SuSFS backport to 4.14 and the SuSFS inline hook script.
- [backslashxx](https://github.com/backslashxx) — scope-minimized hooks (backslashxx/KernelSU #5).
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU) (OriginSU's base, [hooks guide](https://resukisu.github.io/guide/manual-hooks.html)),
  [SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra),
  [KernelSU](https://github.com/tiann/KernelSU) (tiann/weishu, [non-GKI guide](https://kernelsu.org/guide/how-to-integrate-for-non-gki.html)),
  [MKSU](https://github.com/5ec1cff/KernelSU), [RKSU](https://github.com/rsuntk/KernelsU),
  [KernelPatch](https://github.com/bmax121/KernelPatch),
  [Kernel-Assisted Superuser](https://git.zx2c4.com/kernel-assisted-superuser/about/),
  [Magisk](https://github.com/topjohnwu/Magisk), [genuine](https://github.com/brevent/genuine/),
  [Diamorphine](https://github.com/m0nad/Diamorphine), [WildKSU](https://github.com/WildKernels/Wild_KSU),
  [Sui](https://github.com/RikkaApps/Sui), [Shizuku-API](https://github.com/RikkaApps/Shizuku-API),
  [HMA-OSS](https://github.com/frknkrc44/HMA-OSS), [Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist),
  [NoMount](https://github.com/maxsteeel/nomount), [ZeroMount](https://github.com/Enginex0/zeromount)
  (credits taken from OriginSU's README).

**Tools and inspiration**

- [**1992Ade/msm8937-origin-builder**](https://github.com/1992Ade/msm8937-origin-builder) — workflow structure (weekly check, susfs/manual variants, automatic release).
- [**riarumoda/perf_neon-builder**](https://github.com/riarumoda/perf_neon-builder) — SuSFS integration on sweet's sm6150 4.14 kernel.
- [**AnyKernel3**](https://github.com/osm0sis/AnyKernel3) — Chris Renshaw (osm0sis @ xda-developers).
- **LLVM/Clang** and **GitHub Actions**.

## 📜 Licences

Details in [`NOTICE.md`](NOTICE.md).

- **This repository** (scripts, configs, `anykernel.sh`, workflow): **GPL-2.0-only**
  ([`LICENSE`](LICENSE)), because it injects code into the kernel (hooks,
  `path_umount()` backport).
- **Linux kernel / Spiteful**: GPL-2.0. **SuSFS** (kernel code): GPL-2.0.
- **OriginSU**: `kernel/` under GPL-2.0-only, everything else under GPL-3.0-or-later.
- **NonGKI_Kernel_Build_2nd** (4.14 SuSFS patch, downloaded at build time): repository under GPL-3.0.
- **AnyKernel3**: BSD-style licence; its `LICENSE` stays inside every zip.

**Redistributing the zips (GPL):** every zip contains `SOURCE.txt` with the
exact commits of the kernel, OriginSU, the SuSFS patch and this repository.
Keep this repository public and do not rewrite the history used by a
published release.

"Xiaomi", "Redmi", "Qualcomm" and "Snapdragon" are trademarks of their
respective owners. Unofficial project, not affiliated with them, nor with
tbyool, Project Infinity-X, simonpunk, JackA1ltman or the OriginSU developers.
