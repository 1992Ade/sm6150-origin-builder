# Redmi Note 10 Pro (sweet) Kernel Builder with OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/sm6150-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sm6150-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/sm6150-origin-builder/releases/latest)

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` and `dtbo` partitions.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.14** kernel for the **Redmi Note 10 Pro / Pro Max** (`sweet` / `sweetin`, Snapdragon 732G). It is based on the **Spiteful kernel** by [tbyool](https://github.com/tbyool/android_kernel_xiaomi_sm6150) (branch `17.0`, forked from crDroid), the kernel shipped by the [Project Infinity-X](https://projectinfinity-x.com/downloads/sweet) ROM. Check out **[riarumoda](https://github.com/riarumoda/perf_neon-builder)** for the original sm6150 work with BakaSU, other devices and ROMs.

Every build integrates the **[OriginSU](https://github.com/samakshkambxj/OriginSU)** kernel-based root solution and **[SuSFS](https://gitlab.com/simonpunk/susfs4ksu)** (4.14 backport by [JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)) — plus **Baseband Guard**, **NoMount** and **Droidspaces** (see [Included features](#-included-features)) — and is packaged as a recovery-flashable `.zip` (TWRP / OrangeFox) with AnyKernel3.

**Fully automated:** a GitHub Actions workflow checks every week (Sunday) for a new OriginSU release. As soon as a new version is out **with its manager APK**, the kernel is rebuilt with that exact OriginSU version and a new GitHub Release is published automatically — no manual action needed.

The whole build lives in a single workflow file: this repository contains no copy of the kernel, OriginSU or SuSFS, they are downloaded from their official repositories at build time.

---

## 🧩 Included Features

On top of OriginSU, every build enables the same hardening/usability modules as the base [perf_neon-builder](https://github.com/riarumoda/perf_neon-builder) for the `sweet` device:

| Feature | Upstream | What it does | Kernel config |
|---|---|---|---|
| **SuSFS** | [simonpunk](https://gitlab.com/simonpunk/susfs4ksu) | Hides root/mounts from detection (`-susfs` variant only) | `CONFIG_KSU_SUSFS_*` |
| **Baseband Guard** | [vc-teahouse](https://github.com/vc-teahouse/Baseband-guard) | LSM that blocks writes to the modem/baseband partitions | `CONFIG_BBG=y` |
| **NoMount** | [maxsteeel](https://github.com/maxsteeel/nomount) | Prevents unwanted mounts used for root detection | `CONFIG_NOMOUNT=y` |
| **Droidspaces** | [ravindu644](https://github.com/ravindu644/Droidspaces-OSS) | Namespace/cgroup isolation to run isolated Android/Linux spaces | `CONFIG_NAMESPACES`, `CONFIG_*_NS`, cgroups, netfilter… |

> **Droidspaces note:** it enables `CONFIG_CFS_BANDWIDTH=y`, applies two non-GKI patches and adds the namespace/cgroup configs. This Spiteful tree already uses the `kernel/cgroup/cgroup.c` layout, so the cgroup patch applies as-is; a fallback also handles single-file (`kernel/cgroup.c`) trees. If a patch ever stops fitting, it is skipped with a `::warning` and the build continues with the configs enabled. If you use Droidspaces with a systemd-based Linux space, mask the `NetworkManager` and `wpa_supplicant` services inside it (as noted upstream).

---

## ✅ Supported Devices & ROMs

| Device | Codename |
|---|---|
| Redmi Note 10 Pro (global) | `sweet` |
| Redmi Note 10 Pro Max (India) | `sweetin` |

**Requirement:** a ROM that runs on this sm6150 4.14 kernel (Spiteful / crDroid `17.0`). Confirmed working on:

| ROM | Link |
|---|---|
| **Project Infinity-X** (ships the Spiteful kernel) | [projectinfinity-x.com/sweet](https://projectinfinity-x.com/downloads/sweet) |
| **VoltageOS** | [voltageos.com/sweet](https://www.voltageos.com/devices/download/sweet) |

Other AOSP-based ROMs on the same Spiteful / crDroid `17.0` kernel should work too. MIUI / HyperOS are **not** supported.

---

## 📦 Which zip should I download?

Each release ships two variants:

| File | Hook mode | Recommended for |
|---|---|---|
| `Spiteful-OriginSU-<version>-susfs-sweet-<date>.zip` | SuSFS inline hook | Most users — root + SuSFS root hiding |
| `Spiteful-OriginSU-<version>-manual-sweet-<date>.zip` | Manual hook, no SuSFS | Fallback if the SuSFS build doesn't boot |

The SHA-256 sum and the full kernel `.config` of each variant are attached to the release for reference.

---

## 🔧 Installation

0. **Coming from Magisk, APatch, BakaSU or another KernelSU?** Read [this warning](#️-coming-from-another-root-solution) first.
1. Back up your stock `boot` and `dtbo` partitions (from TWRP/OrangeFox, or keep the `boot.img`/`dtbo.img` of your ROM).
2. Download the zip from the [latest release](https://github.com/1992Ade/sm6150-origin-builder/releases/latest).
3. Reboot to recovery and flash it (or `adb sideload <file>.zip`).
4. Reboot to system.
5. Install the **OriginSU manager** APK (`arm64-v8a` or `universal`) from the [OriginSU releases](https://github.com/samakshkambxj/OriginSU/releases) (use the same version as the kernel).

The zip replaces the kernel and the `dtb` inside `boot` and flashes `dtbo`; your ROM's ramdisk is kept. It refuses to install on any device other than `sweet` / `sweetin`.

> **Updating the ROM?** An OTA update replaces `boot`, so the OriginSU kernel is removed. Flash the zip again after each ROM update.

### ↩️ Restore the stock kernel

```sh
adb shell su -c 'rm -rf /data/adb/*'    # remove root data first
adb reboot bootloader
fastboot flash boot boot_backup.img
fastboot flash dtbo dtbo_backup.img
fastboot reboot
```

---

## ⚙️ How the Build Works

The workflow lives in [`.github/workflows/build.yml`](.github/workflows/build.yml) and runs three jobs:

```
check  ──►  build (susfs)  ──┐
       └─►  build (manual) ──┴──►  release
```

1. **check** — reads the latest stable OriginSU release, makes sure it ships the **manager APK**, and looks for a release named `originsu-<tag>` in this repo. If it already exists (or the manager is not published yet), the run stops there. It also pins the commit of JackA1ltman's SuSFS patches so both variants use the same one.
2. **build** (×2, in parallel)
   - Clones tbyool's Spiteful kernel (`17.0`) and downloads the latest **Neutron Clang** toolchain.
   - Generates the config from `sweet_defconfig` (ThinLTO, EROFS, dm-verity…, as in perf_neon's sweet build) with `CONFIG_LOCALVERSION="-Spiteful-OriginSU"`.
   - Integrates OriginSU at the exact release tag with its official `setup.sh`, plus a 4.14 fix (`<linux/ratelimit.h>`).
   - **susfs:** applies `susfs_patch_to_4.14.patch` and the SuSFS inline hooks (`susfs_inline_hook_patches.sh`), with perf_neon's 4.14 fixes (uname spoof, `fs/stat.c`, `fs/open.c`).
   - **manual:** applies the manual hooks (`syscall_hook_patches.sh`) for `exec`, `faccessat`, `stat` and `reboot` only; OriginSU hooks `setuid`, `init.rc` and input by itself on 4.14.
   - Checks that every hook OriginSU expects is in place, exports the static SELinux symbols, compiles `Image.gz` + `dtb.img` + `dtbo.img` and packages them with [riarumoda's AnyKernel3](https://github.com/riarumoda/AnyKernel3).
3. **release** — publishes a GitHub Release `originsu-<tag>` with both zips, their SHA-256 sums, the configs and release notes (OriginSU and SuSFS versions, kernel commit, kernel version, toolchain, build date).

### Schedule

| Trigger | Behaviour |
|---|---|
| Every Sunday at 04:23 UTC | Builds **only** if a new OriginSU release (with its manager) is out |
| Manual (`Run workflow`) | Always builds; you can pick an OriginSU tag, toggle KPM and choose whether to publish a release |

A manual run for a version that is already released **replaces** that release (useful to pick up kernel or SuSFS updates).

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled and **Workflow permissions** allow *Read and write* (needed to publish releases).
3. In the **Actions** tab, enable scheduled workflows on the fork.
4. Run **Build sweet Kernel (Spiteful) with OriginSU** once manually to create the first release.

> GitHub disables scheduled workflows after **60 days without repository activity**. If automatic releases stop, re-enable the workflow in the Actions tab or push a commit.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | *(latest release)* | OriginSU tag to build |
| `kpm` | `false` | Enable `CONFIG_KPM` |
| `publish` | `true` | Publish a GitHub Release (otherwise only build artifacts) |

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| *Integrate OriginSU* fails with "SuSFS patch did not apply" | JackA1ltman's 4.14 patch no longer matches the kernel — check `build-susfs.log`; the `-manual` variant still builds |
| Compile fails with `use of undeclared identifier 'slab_flags_t'` | Triggered by Droidspaces' `CONFIG_MEMCG` on this pre-`slab_flags_t` 4.14 base. The *Extra features* step adds the missing typedef automatically; if it ever reappears, drop `CONFIG_MEMCG=y` from the Droidspaces block |
| *Integrate OriginSU* fails with "hook … missing" | A hook script no longer matches the kernel code — check the run log |
| Build fails with "You lost … hook" / "incompatible hook" | A new OriginSU version requires a different hook — re-run with an older `originsu_ref` |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| No build although OriginSU released | The release has no manager APK yet; the next weekly run will pick it up |
| Device stuck on boot logo | Flash the `-manual` variant, or restore your `boot`/`dtbo` backups |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict — see [Coming from another root solution?](#️-coming-from-another-root-solution) below |
| The manager says "not installed" | Check `uname -r` (must contain `-Spiteful-OriginSU`) and the manager version |
| No new release after an OriginSU update | Check that the scheduled workflow is still enabled (60-day rule above) |

Build logs (`build-log-*`) are attached to every run, even when it fails. Pressing *volume down* at least 3 times during boot enables KernelSU safe mode (modules disabled).

### ⚠️ Coming from another root solution?

If you previously used **Magisk, APatch, BakaSU, KernelSU or SukiSU**, your old modules are still in `/data/adb/modules` and will be loaded by OriginSU.

OriginSU ships its **own Zygisk (OriginZygisk)**. Old modules — especially another Zygisk implementation (ReZygisk, NeoZygisk, Zygisk Next) or Zygisk-based modules such as LSPosed — can conflict with it. The typical symptom is a phone that **boots fine but has no Wi-Fi, no mobile network and/or no camera**.

**Fix: uninstall all modules, then reinstall them one by one.**

From a PC (use the interactive shell — it also works in Windows PowerShell):

```sh
adb shell
su
for m in /data/adb/modules/*/; do touch "$m/remove"; done
rm -rf /data/adb/modules_update/*
reboot
```

Each module's `uninstall.sh` runs on the next boot. If some folders remain afterwards:

```sh
adb shell
su
rm -rf /data/adb/modules/*
reboot
```

Then:

- Reinstall your modules **one at a time**, checking Wi-Fi, network and camera after each reboot.
- Use **only one Zygisk**: keep OriginZygisk and don't install another Zygisk module.
- Reinstall Zygisk-based modules (e.g. LSPosed) last.
- Optional cleanup of leftovers from other root solutions, e.g. APatch: `rm -rf /data/adb/ap /data/adb/apd`.

---

## 🙏 Credits & Acknowledgments

This project relies on the incredible work of the following open-source projects and developers:

* **[tbyool](https://github.com/tbyool/android_kernel_xiaomi_sm6150)** — For the Spiteful kernel (branch `17.0`).
* **[crDroid](https://github.com/crdroidandroid/android_kernel_xiaomi_sm6150)** — For the sm6150 kernel tree Spiteful is forked from.
* **[Project Infinity-X](https://projectinfinity-x.com/)** — For the ROM and its sweet maintainer.
* **[OriginSU](https://github.com/samakshkambxj/OriginSU)** — For the kernel-based root solution.
* **[riarumoda](https://github.com/riarumoda/perf_neon-builder)** — For perf_neon-builder, whose sweet workflow this build is based on, and for the [AnyKernel3 fork](https://github.com/riarumoda/AnyKernel3) used for packaging.
* **[osm0sis](https://github.com/osm0sis/AnyKernel3)** — For the original AnyKernel3.
* **[JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)** — For the 4.14 SuSFS backport and the non-GKI hook scripts.
* **[backslashxx](https://github.com/backslashxx)** — For the scope-minimized hooks those scripts are based on.
* **[simonpunk](https://gitlab.com/simonpunk/susfs4ksu)** — For SuSFS.
* **[vc-teahouse](https://github.com/vc-teahouse/Baseband-guard)** — For Baseband Guard.
* **[maxsteeel](https://github.com/maxsteeel/nomount)** — For NoMount.
* **[ravindu644](https://github.com/ravindu644/Droidspaces-OSS)** — For Droidspaces.
* **[Neutron Toolchains](https://github.com/Neutron-Toolchains)** — For the Clang toolchain.
* **[tiann/KernelSU](https://github.com/tiann/KernelSU)** — The project OriginSU is built upon.

## 📄 License

Kernel sources are licensed under **GPL-2.0**. See each upstream project for its own license.

"Xiaomi", "Redmi", "Qualcomm" and "Snapdragon" are trademarks of their respective owners. Unofficial project, not affiliated with them, nor with tbyool, Project Infinity-X, simonpunk, JackA1ltman or the OriginSU developers.
