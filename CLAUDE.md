# CLAUDE.md — wimb-whyred Kernel

This file documents the codebase structure, build workflows, and conventions for AI assistants working on this repository.

## Project Overview

**wimb-whyred** is a custom Android Linux kernel (version **4.4.222**, codename "Blurry Fish Butt") targeting the **Xiaomi Redmi Note 5** (device codename: `whyred`). It is based on the Qualcomm CAF (Code Aurora Forum) kernel for the **Snapdragon 636 / SDM660** SoC and targets the **AOSPExtended** custom ROM (`CONFIG_LOCALVERSION="-aospextended"`).

The kernel includes several upstream backports and custom features beyond the stock Qualcomm CAF tree, such as the Dynamic SchedTune Boost framework, WireGuard VPN, NTFS/ExFAT write support, and KASLR.

---

## Repository Structure

```
wimb-whyred/
├── AndroidKernel.mk          # Android build system integration (AOSP make)
├── Makefile                  # Top-level Linux kernel Makefile (version 4.4.222)
├── Kbuild / Kconfig          # Top-level build/config entry points
├── backported-features       # Documents LSK (Linaro Stable Kernel) backports
├── build.config.*            # Build configs for various targets (aarch64, x86_64, goldfish, cuttlefish)
├── verity_dev_keys.x509      # dm-verity development signing key
│
├── arch/
│   └── arm64/                # Primary architecture (all device work lives here)
│       ├── configs/          # Kernel defconfigs (one per device)
│       │   ├── whyred_defconfig      # PRIMARY — Redmi Note 5 (whyred)
│       │   ├── vimb_defconfig        # Full merged config (very large, 122KB)
│       │   ├── sdm660_defconfig      # Generic SDM660 debug config
│       │   ├── sdm660-perf_defconfig # SDM660 performance config
│       │   ├── lavender_defconfig    # Redmi Note 7
│       │   ├── wayne_defconfig       # Mi A2
│       │   ├── jason_defconfig       # Mi 6X
│       │   ├── tulip_defconfig       # Redmi Note 6 Pro
│       │   └── msmcortex_defconfig   # Generic MSM8998
│       └── boot/
│           └── dts/
│               └── qcom -> (symlink to vendor/qcom DTS source)
│
├── drivers/                  # Kernel drivers (Qualcomm, touchscreen, fingerprint, etc.)
├── kernel/                   # Core kernel (sched, printk, etc.)
├── mm/                       # Memory management
├── net/                      # Networking (includes WireGuard)
├── fs/                       # Filesystems (ext4, f2fs, ntfs, exfat, sdfat)
├── security/                 # SELinux, seccomp
├── sound/                    # ALSA/ASoC audio
├── include/                  # Kernel headers
├── scripts/                  # Build and utility scripts
├── android/
│   └── configs/              # Android-specific Kconfig fragment files
└── Documentation/            # Linux kernel documentation
```

---

## Target Device

| Property | Value |
|---|---|
| Device | Xiaomi Redmi Note 5 |
| Codename | `whyred` |
| SoC | Qualcomm Snapdragon 636 (SDM660) |
| Architecture | ARM64 (aarch64) |
| CPU cores | 8 (Kryo 260, HMP scheduler) |
| Primary defconfig | `arch/arm64/configs/whyred_defconfig` |
| Kernel version | 4.4.222 |
| Local version string | `-aospextended` |
| Target ROM | AOSPExtended |

---

## Build System

### Standalone / Out-of-tree build

The standard Linux Kbuild system is used. For ARM64 cross-compilation:

```sh
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-android-
# or for Clang:
export CC=clang
export CLANG_TRIPLE=aarch64-linux-gnu-
export CROSS_COMPILE=aarch64-linux-androidkernel-

# Generate config
make whyred_defconfig

# Build kernel + modules
make -j$(nproc)

# Build only the image
make Image.gz-dtb -j$(nproc)
```

Output artifact: `arch/arm64/boot/Image.gz-dtb` (appended DTB is the default, controlled by `CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE=y`).

### Android AOSP tree build (via AndroidKernel.mk)

The file `AndroidKernel.mk` integrates this kernel into an Android build tree. It reads these variables from the Android `BoardConfig.mk`:

| Variable | Purpose |
|---|---|
| `TARGET_KERNEL_ARCH` | Target arch (default: `arm`, override to `arm64`) |
| `KERNEL_DEFCONFIG` | Defconfig file name |
| `TARGET_KERNEL_CROSS_COMPILE_PREFIX` | Toolchain prefix |
| `TARGET_KERNEL_MAKE_ENV` | Extra env vars passed to make |
| `TARGET_KERNEL_APPEND_DTB` | `true` = Image.gz-dtb, `false` = Image.gz + dtb.img |

Module outputs land in `$(PRODUCT_OUT)/system/lib/modules/`.

### Build configs (for build.sh / Bazel-style builds)

- `build.config.common` — Sets `BRANCH=android-4.4-p`, uses Clang + GCC 4.9 prebuilts
- `build.config.aarch64` — ARM64 target: `ARCH=arm64`, outputs `Image.gz` + `vmlinux` + `System.map`
- `build.config.x86_64` — x86_64 target
- `build.config.goldfish.*` / `build.config.cuttlefish.*` — Emulator targets

---

## Key defconfig: `whyred_defconfig`

Highlight of non-default or noteworthy settings:

| Config | Purpose |
|---|---|
| `CONFIG_ARCH_SDM660=y` | Qualcomm SDM660 platform |
| `CONFIG_MACH_XIAOMI_WHYRED=y` | whyred machine definition |
| `CONFIG_SCHED_HMP=y` | Heterogeneous Multi-Processing scheduler |
| `CONFIG_CGROUP_SCHEDTUNE=y` | SchedTune CGroup (basis for Dynamic Boost) |
| `CONFIG_RANDOMIZE_BASE=y` | KASLR enabled |
| `CONFIG_WIREGUARD=y` | WireGuard VPN |
| `CONFIG_NTFS_FS=y` + `NTFS_RW=y` | NTFS with write support |
| `CONFIG_EXFAT_FS=y` + `SDFAT_FS=y` | ExFAT filesystem |
| `CONFIG_FB_MSM_MDSS_KCAL_CTRL=y` | Display color calibration |
| `CONFIG_SOUND_CONTROL=y` | Audio gain control |
| `CONFIG_MODULE_SIG_FORCE=y` | Signed modules only |
| `CONFIG_CC_WERROR=y` | Warnings are errors |
| `CONFIG_FPC_FINGERPRINT=y` | FPC fingerprint sensor |
| `CONFIG_GOODIX_FINGERPRINT=y` | Goodix fingerprint sensor |
| `CONFIG_F2FS_FS=y` | F2FS filesystem |
| `CONFIG_DM_VERITY=y` + `DM_ANDROID_VERITY=y` | dm-verity |
| `CONFIG_PREEMPT=y` | Full kernel preemption |
| `CONFIG_HZ_300=y` | 300Hz timer tick |

---

## Notable Custom Features

### Dynamic SchedTune Boost

Located in `kernel/sched/tune.c`. A slot-based system (up to 5 slots per SchedTune group) for stacking multiple concurrent boost requests. Key functions:

- `do_stune_sched_boost()` — activate a boost slot
- `reset_stune_boost()` — release a boost slot
- Integrates with `/proc/sys/kernel/sched_boost` (re-introduced from HMP) to trigger top-app boosting
- Tunable: `/dev/stune/*/schedtune.sched_boost`

Every `do_stune_sched_boost()` call **must** be paired with a `reset_stune_boost()` to maintain `stune_boost_count` correctly.

### msm_performance

- CPU frequency min/max setting via msm_performance is intentionally disabled
- Works alongside Dynamic SchedTune Boost for integrated boosting
- Hotplug management removed (depends on removed `msm_core.c`)
- `CONFIG_MSM_CORE` is disabled in defconfig

### LSK Backported Features (`backported-features`)

1. **KASLR** — `v4.4/topic/mm-kaslr` and `v4.4/topic/mm-kaslr-pax_usercopy`
2. **Coresight + OpenCSD** — Juno board perf tool support
3. **OPTEE** — Based on LSK mainline (not in AOSP mainline)

---

## Commit Message Conventions

This repository follows **Linux kernel commit style**:

```
subsystem/component: Short imperative summary

Longer description explaining WHY the change is needed,
not what it does. Reference relevant upstream commits
with their short SHA and description in parentheses.

Change-Id: I<gerrit-id>          (optional, from CAF)
Signed-off-by: Name <email>
```

Examples of well-formed subsystem prefixes used here:
- `whyred: dts: ...` — device-specific DTS changes
- `sched/tune: ...` — scheduler tuning
- `sched/boost: ...` — scheduler boost
- `msm_performance: ...` — Qualcomm performance driver
- `defconfig: ...` — defconfig changes
- `init: ...` — init/Kconfig dependency fixes

**Always include `Signed-off-by:`** for any new code.

When cherry-picking from CAF or upstream, preserve original authorship and add your own `Signed-off-by:` at the bottom.

---

## Device Tree (DTS)

- QCOM DTS files live under `arch/arm64/boot/dts/qcom/` (symlinked)
- The kernel uses **appended DTB** by default (`Image.gz-dtb`)
- Alternatively, a standalone `dtb.img` can be built by setting `TARGET_KERNEL_APPEND_DTB=false`; the Makefile will then `cat` all `*.dtb` from the qcom DTS output
- Whyred-specific DTS edits should target the `sdm660-*-whyred*.dts` / `.dtsi` files in the qcom directory
- Recent fix: `pa_therm0` thermal zone in the whyred DTS (commit `da2480f`)

---

## Branches

| Branch | Purpose |
|---|---|
| `master` | Main development branch (default) |
| `common` | Tracks common/base kernel (LSK/CAF baseline) |
| `hyde27-patch-1` | Experimental patch branch |
| `test` | Testing branch (currently same HEAD as master) |

---

## Key Files to Know

| File | Purpose |
|---|---|
| `arch/arm64/configs/whyred_defconfig` | Primary device defconfig — edit here for feature toggles |
| `arch/arm64/configs/vimb_defconfig` | Full merged defconfig (large, may be used for build validation) |
| `AndroidKernel.mk` | Drives kernel build from AOSP make |
| `build.config.aarch64` | Clang-based build config for aarch64 |
| `build.config.common` | Shared build config (branch, compiler, flags) |
| `backported-features` | Documents what has been cherry-picked from LSK |
| `verity_dev_keys.x509` | dm-verity dev key (do not use in production signing) |
| `kernel/sched/tune.c` | Dynamic SchedTune Boost implementation |
| `drivers/soc/qcom/msm_performance.c` | MSM performance driver |

---

## Coding Conventions

- **C standard**: GNU C with Linux kernel style (`scripts/checkpatch.pl`)
- **Indentation**: Tabs (8-space equivalent), as per Linux kernel coding style
- **Line length**: 80 characters preferred, 100 max
- **No trailing whitespace**
- Run `scripts/checkpatch.pl --strict` before committing driver or core changes
- Kconfig entries: new options should have a `help` section and depend on appropriate platform guards (e.g., `depends on ARCH_SDM660`)
- DTS: follow existing node naming conventions; use `pa_therm` naming for thermal sensors
- New drivers must go into the appropriate `drivers/` subdirectory with a corresponding `Kconfig` + `Makefile` entry
- Module signing: all modules must be signed (`CONFIG_MODULE_SIG_FORCE=y`); the trusted key is `verity.x509.pem`

---

## Defconfig Workflow

When modifying kernel config options:

```sh
# 1. Start from the whyred defconfig
make ARCH=arm64 whyred_defconfig

# 2. Open menuconfig to make changes interactively
make ARCH=arm64 menuconfig

# 3. Save back to the defconfig file
make ARCH=arm64 savedefconfig
cp defconfig arch/arm64/configs/whyred_defconfig
```

Alternatively, via `AndroidKernel.mk` target:
```sh
make -f AndroidKernel.mk kernelconfig
```

This automatically saves the defconfig back to `arch/$(KERNEL_ARCH)/configs/$(KERNEL_DEFCONFIG)`.

---

## Contributors

- **hyde27** — primary maintainer (whyred-specific commits, DTS fixes)
- **laststandrighthere (Lau)** — scheduler and msm_performance improvements
- **joshuous / joshchoo** — Dynamic SchedTune Boost framework
