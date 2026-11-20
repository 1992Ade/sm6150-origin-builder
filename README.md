# Redmi Note 10 Pro (sweet) Kernel Builder with OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/sm6150-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sm6150-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/sm6150-origin-builder/releases/latest)

> [!IMPORTANT]
> **This is the recommended kernel for sweet — OriginSU `v4.3.0`, the final OriginSU release.**
> OriginSU has been renamed and rebuilt as **[KazeSU](https://github.com/KazeSU/KazeSU)**, which targets modern (GKI 5.10+) kernels and **does not build on this Linux 4.14 kernel** (its SELinux code needs the 5.10+ layout). OriginSU `v4.3.0` still supports it fully (non-GKI hooks, SuSFS, built-in module mounting), so this builder stays on it: the root solution is frozen, but the builder is still maintained by hand for kernel updates and optimizations.
> **4.14 is also the newest Android kernel for this SoC** (Snapdragon 732G / SM7150): Qualcomm never released a newer one. A mainline Linux port exists, but only for Linux distributions such as postmarketOS, not for Android ROMs.
>
> KazeSU support for sweet is **on hold** and will be looked at again if KazeSU supports older kernels again. The OriginSU `v4.3.0` manager APK is attached to every release (also on the KazeSU repository: [tag `v4.3.0`, release "legacy-v4.3.0"](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)). Note that KazeSU `v1.0.0` works with this kernel too but new root hiding method is not implemented.

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` and `dtbo` partitions.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.14** kernel for the **Redmi Note 10 Pro / Pro Max** (`sweet` / `sweetin`, Snapdragon 732G). It is based on the **Spiteful kernel** by [tbyool](https://github.com/tbyool/android_kernel_xiaomi_sm6150) (branch `17.0`, forked from crDroid), the kernel shipped by the [Project Infinity-X](https://projectinfinity-x.com/downloads/sweet) ROM. Check out **[riarumoda](https://github.com/riarumoda/perf_neon-builder)** for the original sm6150 work with BakaSU, other devices and ROMs.

Every build integrates the **[OriginSU v4.3.0](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** kernel-based root solution and **[SuSFS](https://gitlab.com/simonpunk/susfs4ksu)** (4.14 backport by [JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)) — plus **Baseband Guard**, **NoMount**, **Droidspaces**, **TCP BBR + fq** and a few [optimizations](#-optimizations) (see [Included features](#-included-features)) — and is packaged as a recovery-flashable `.zip` (TWRP / OrangeFox) with AnyKernel3. The matching **OriginSU manager APK** is attached to each release.

**Manual builds only:** OriginSU will not get new releases, so the weekly check was removed. Running the workflow by hand rebuilds OriginSU `v4.3.0` on the latest Spiteful kernel, with the current optimizations, and replaces the release.

The whole build lives in a single workflow file: this repository contains no copy of the kernel, OriginSU or SuSFS, they are downloaded from their official repositories at build time.

---

## 🧩 Included Features

On top of OriginSU, every build enables the same hardening/usability modules as the base [perf_neon-builder](https://github.com/riarumoda/perf_neon-builder) for the `sweet` device:

| Feature | Upstream | What it does | Kernel config |
|---|---|---|---|
| **SuSFS** | [simonpunk](https://gitlab.com/simonpunk/susfs4ksu) | Hides root/mounts from detection (`-susfs` variant only) | `CONFIG_KSU_SUSFS_*` |
| **Baseband Guard** | [vc-teahouse](https://github.com/vc-teahouse/Baseband-guard) | LSM that blocks writes to the modem/baseband partitions | `CONFIG_BBG=y` |
| **NoMount** | [maxsteeel](https://github.com/maxsteeel/nomount) | Mounts modules without real mounts (VFS path redirection, nothing in `/proc/mounts`) — driven by OriginSU's *OriginMount* screen, see [Modules](#-modules) | `CONFIG_NOMOUNT=y` |
| **Droidspaces** | [ravindu644](https://github.com/ravindu644/Droidspaces-OSS) | Namespace/cgroup isolation to run isolated Android/Linux spaces | `CONFIG_NAMESPACES`, `CONFIG_*_NS`, cgroups, netfilter… |
| **TCP BBR + fq** | Linux mainline | BBR is the default TCP congestion control, on the `fq` qdisc (see [TCP BBR](#-tcp-bbr)) | `CONFIG_DEFAULT_BBR=y`, `CONFIG_DEFAULT_FQ=y` |
| **Optimizations** | this repo | Power-efficient workqueues, `vm.swappiness` up to 200 (see [Optimizations](#-optimizations)) | `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y` |

> ### Droidspaces — what works and what doesn't on this kernel
> **Droidspaces works** (containers, namespaces, overlayfs and networking are all enabled). Two of its *optional* configs are off in the stock Spiteful defconfig and don't build as-is on this 4.14 tree:
>
> | Config | Status | What it gives Droidspaces |
> |---|---|---|
> | `CONFIG_MEMCG` | **Fixed, optional** (`droidspaces_memcg` input, off by default; zips get a `-memcg` suffix). Two compile errors were fixed: `SLAB_ACCOUNT` used `slab_flags_t`, a 4.15 type this tree lacks, and `mem_cgroup_is_root()` was defined twice. Everything that uses memcg builds with it. Experimental: memcg adds a small accounting overhead and changes how Android sets up memory cgroups, so test it before daily use | Per-container **memory limit** and **swap limit** |
> | `CONFIG_CFS_BANDWIDTH` | **Still off.** The CFS-bandwidth code in `kernel/sched/fair.c` is an incomplete backport mixing three scheduler generations (undeclared `qcfs_rq`, a loop closed too early, tasks counted twice when a group is unthrottled, a field from Linux 6.18). Fixing it means porting the whole throttling path: a scheduler bug there can freeze the phone, not worth it | Per-container **CPU limit** (`--cpus`) |
>
> Droidspaces' own docs list both as optional: *"without these two options everything else works; `--cpus` and the memory/swap limits are skipped with a warning."* Everything else — namespaces, the **pids** (`--pids-limit`), freezer, devices and net cgroups, overlayfs and NAT/macvlan networking — is enabled. `CONFIG_CGROUP_PIDS` is kept on safely because this kernel is monolithic (no loadable vendor modules, so no symbol-CRC bootloop risk).
>
> The cgroup patch applies as-is (this tree uses the `kernel/cgroup/cgroup.c` layout; a fallback also handles single-file `kernel/cgroup.c` trees). If you use Droidspaces with a systemd-based Linux space, mask the `NetworkManager` and `wpa_supplicant` services inside it (as noted upstream).

### 🌐 TCP BBR

The stock Spiteful config uses **BBRplus** (an out-of-tree BBR variant) on the **cake** qdisc, and does not build mainline BBR or `fq`. These builds switch the defaults to **mainline BBR on `fq`**, the pairing BBR is designed for:

- `fq` sends packets at the rate BBR computes (`sk_pacing_rate`) and tells TCP not to pace by itself, which is more precise and cheaper on the CPU than TCP's per-connection pacing timer used with cake.
- `cake` is built for routers that shape a known link speed; on a phone it does not use BBR's rate.
- Mainline BBR is the version maintained in Linux and run by Google in production; BBRplus is a third-party variant.

**BBRplus and cake are still built in**, so you can switch back at runtime (root shell, until reboot):

```sh
cat /proc/sys/net/ipv4/tcp_congestion_control                   # bbr
cat /proc/sys/net/core/default_qdisc                            # fq
cat /proc/sys/net/ipv4/tcp_available_congestion_control         # ... bbr bbrplus
echo bbrplus > /proc/sys/net/ipv4/tcp_congestion_control        # Spiteful's original choice
echo cake    > /proc/sys/net/core/default_qdisc                 # applies to interfaces brought up afterwards
```

The build fails if the final `.config` does not have `CONFIG_DEFAULT_TCP_CONG="bbr"` and `CONFIG_DEFAULT_NET_SCH="fq"`.

### ⚡ Optimizations

Spiteful is already well tuned: ThinLTO, Polly, Cortex-A76 code tuning, zstd for ZRAM, the IPv4 `TTL` / IPv6 `HL` targets, WireGuard, Simple LMK and an updated LZ4 with arm64 assembly are all in the tree. This builder only adds what is missing (low risk; the build fails if they are not in the final `.config`):

| Change | Effect | How to use / check |
|---|---|---|
| `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y` | Background kernel work runs on CPUs that are already awake instead of waking an idle core | `cat /sys/module/workqueue/parameters/power_efficient` → `Y` |
| `vm.swappiness` max 100 → 200 | Lets you make the kernel prefer swapping to ZRAM over dropping app caches. The default value set by the ROM is unchanged | `echo 160 > /proc/sys/vm/swappiness` (values up to 200 accepted) |

Not included on purpose: MGLRU (too large to backport to 4.14) and the TEO cpuidle governor (Qualcomm's own idle governor takes priority on this SoC).

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
5. Install the **OriginSU manager** APK attached to the same release (`OriginSU_v4.3.0_…-arm64-v8a-release.apk`, also on the [legacy-v4.3.0 page](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)). It must match the kernel: do **not** install the KazeSU v1.x manager with this kernel.
6. **Pick a module mount mode** in the manager's **OriginMount** screen — otherwise modules that change `/system` are not mounted (see [Modules](#-modules)).

The zip replaces the kernel and the `dtb` inside `boot` and flashes `dtbo`; your ROM's ramdisk is kept. It refuses to install on any device other than `sweet` / `sweetin`.

> **Updating the ROM?** An OTA update replaces `boot`, so the OriginSU kernel is removed. Flash the zip again after each ROM update.

### 🧱 Modules

OriginSU mounts modules itself. Choose how in the manager's **OriginMount** screen (the choice applies from the next boot):

| Mode | What it does |
|---|---|
| `auto` (default) | Uses a metamodule if one is installed, otherwise Magic Mount if you enabled it, otherwise **no mounting** |
| `nomount` | Uses the **NoMount** driver built into this kernel: module files are injected without real mounts, so nothing shows in `/proc/mounts` |
| `builtin` | OriginSU's built-in Magic Mount |
| `metamodule` | Delegates to an installed metamodule (e.g. meta-overlayfs) |
| `off` | No module mounting (scripts, `sepolicy.rule` and `system.prop` still work) |

NoMount is already in the kernel and OriginSU talks to it directly: **do not install the separate NoMount module zip** on top. If your modules are installed but their files don't show up, you are probably in `auto` mode with nothing to mount them: pick a mode.

### 🔍 Checking the kernel after flashing

Root shell (`adb shell` then `su`, or a terminal app):

```sh
uname -r                                              # 4.14.x-Spiteful-OriginSU
cat /proc/sys/net/ipv4/tcp_congestion_control         # bbr
cat /proc/sys/net/core/default_qdisc                  # fq
cat /sys/block/zram0/comp_algorithm                   # [lz4] ... zstd
cat /proc/net/ip_tables_targets | grep TTL            # TTL
cat /proc/net/ip6_tables_targets | grep HL            # HL
cat /sys/module/workqueue/parameters/power_efficient  # Y
dmesg | grep -iE "nomount|baseband|susfs" | head      # NoMount, Baseband Guard, SuSFS lines
ls /proc/self/ns                                      # user net pid uts ipc ... (Droidspaces)
```

In the manager: OriginSU **v4.3.0**, the SuSFS version (`-susfs` zip), and NoMount shown as available in **OriginMount**. Then check Wi-Fi, mobile data and calls, camera, Bluetooth, fingerprint, audio, charging and overnight battery drain. If something is wrong, save the logs right after reproducing it: `dmesg > /sdcard/dmesg.txt` and `logcat -b all -d > /sdcard/logcat.txt`.

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

1. **check** — selects the OriginSU tag (`v4.3.0` by default; only `v4.x` OriginSU tags are accepted) and makes sure its release ships the **manager APK**. It also pins the commit of JackA1ltman's SuSFS patches so both variants use the same one.
2. **build** (×2, in parallel)
   - Clones tbyool's Spiteful kernel (`17.0`) and downloads the latest **Neutron Clang** toolchain.
   - Generates the config from `sweet_defconfig` (ThinLTO, EROFS, dm-verity…, as in perf_neon's sweet build) with `CONFIG_LOCALVERSION="-Spiteful-OriginSU"`.
   - Integrates OriginSU at the exact tag with its own `setup.sh` from that tag, cloned from the KazeSU repository (the build stops if the tag cannot be checked out), plus a 4.14 fix (`<linux/ratelimit.h>`).
   - **susfs:** applies `susfs_patch_to_4.14.patch` and the SuSFS inline hooks (`susfs_inline_hook_patches.sh`), with perf_neon's 4.14 fixes (uname spoof, `fs/stat.c`, `fs/open.c`).
   - **manual:** applies the manual hooks (`syscall_hook_patches.sh`) for `exec`, `faccessat`, `stat` and `reboot` only; OriginSU hooks `setuid`, `init.rc` and input by itself on 4.14.
   - Applies the [optimizations](#-optimizations) and enables Baseband Guard, NoMount, Droidspaces and TCP BBR + `fq` (see [Included features](#-included-features)); the build stops if the NoMount driver speaks an API older than the one OriginSU requires (22).
   - Checks that every hook OriginSU expects is in place, exports the static SELinux symbols, compiles `Image.gz` + `dtb.img` + `dtbo.img` and packages them with [riarumoda's AnyKernel3](https://github.com/riarumoda/AnyKernel3).
3. **release** — downloads the matching OriginSU manager APK (arm64-v8a) and publishes a GitHub Release `originsu-<tag>` with both zips, their SHA-256 sums, the configs, the manager and release notes (OriginSU and SuSFS versions, kernel commit, kernel version, toolchain, build date).

### Schedule

| Trigger | Behaviour |
|---|---|
| Manual (`Run workflow`) | Builds OriginSU `v4.3.0` (or another `v4.x` tag); you can toggle KPM and choose whether to publish a release |

There is no scheduled build any more (OriginSU is discontinued).

A manual run for a version that is already released **replaces** that release (useful to pick up kernel or SuSFS updates).

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled and **Workflow permissions** allow *Read and write* (needed to publish releases).
3. Run **Build sweet Kernel (Spiteful) with OriginSU** once manually to create the first release.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | `v4.3.0` | OriginSU tag to build (`v4.x` only) |
| `kpm` | `false` | Enable `CONFIG_KPM` |
| `droidspaces_memcg` | `false` | Enable `CONFIG_MEMCG` for Droidspaces memory/swap limits (experimental, zips get a `-memcg` suffix) |
| `publish` | `true` | Publish a GitHub Release (otherwise only build artifacts) |

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| *Integrate OriginSU* fails with "SuSFS patch did not apply" | JackA1ltman's 4.14 patch no longer matches the kernel — check `build-susfs.log`; the `-manual` variant still builds |
| Compile fails in `mm/memcontrol.c` or on `slab_flags_t` | `CONFIG_MEMCG` was enabled without the two fixes — use the `droidspaces_memcg` input instead of adding it by hand |
| Compile fails in `kernel/sched/fair.c` (`qcfs_rq`, `pelt_clock_throttled`) | `CONFIG_CFS_BANDWIDTH` was enabled: this tree's CFS-bandwidth code is an incomplete backport, keep it off |
| Build fails with `optimizations` / `… missing from the final config` | A config this builder relies on was dropped by Kconfig — check the *Optimizations* lines in the compile log |
| Modules installed but their files don't show up | Pick a mode in **OriginMount** (`auto` mounts nothing without a metamodule or Magic Mount) — see [Modules](#-modules) |
| *Integrate OriginSU* fails with "hook … missing" | A hook script no longer matches the kernel code — check the run log |
| Build fails with "You lost … hook" / "incompatible hook" | A new OriginSU version requires a different hook — re-run with an older `originsu_ref` |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| Device stuck on boot logo | Flash the `-manual` variant, or restore your `boot`/`dtbo` backups |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict — see [Coming from another root solution?](#️-coming-from-another-root-solution) below |
| The manager says "not installed" | Check `uname -r` (must contain `-Spiteful-OriginSU`) and the manager version |
| Build fails with a BBR / fq error | The kernel config no longer selects BBR or `fq` by default — check the *TCP congestion control* lines in the compile log |

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
* **[OriginSU](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** (now [KazeSU](https://github.com/KazeSU/KazeSU)) — For the kernel-based root solution.
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
