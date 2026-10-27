# Samsung Galaxy A07 4G (SM-A075F) Kernel Source

Adapted Linux kernel source tree for the **Samsung Galaxy A07 4G** (`SM-A075F`, codename `hs07` / `a07`), powered by the **MediaTek Helio G99** (`MT6789`) chipset.

This repository provides an adapted, production-ready kernel source based on **Linux 5.10.233** (Android Common Kernel `android12-5.10`, KMI generation 9), tailored for compatibility with custom software, including custom ROMs, Generic System Images (GSIs), and rooting solutions (Magisk, KernelSU, APatch).

---

## Device Specifications

| Specification | Details |
|---|---|
| **Device Model** | Samsung Galaxy A07 4G (`SM-A075F`) |
| **Board / Platform** | MediaTek Helio G99 (`MT6789` / `mt6789`) |
| **Architecture** | ARM64 (`aarch64`, little-endian) |
| **Base Kernel Version** | Linux 5.10.233 (ACK `android12-5.10`) |
| **Android Version** | Android 12+ (GKI 2.0 / ACK KMI gen 9) |
| **Target Defconfig** | `a07_00_defconfig` (with `entry_level.config` and `hs07.config`) / `a07_defconfig` |
| **Target Device Tree** | `arch/arm64/boot/dts/mediatek/mt6789_a07_eur_open_00.dtb` |

---

## Purpose & Adaptations

Stock Samsung kernels embed aggressive proprietary security mechanisms designed to prevent modifications, enforce strict Knox policies, trap unauthorized credential alterations via hypervisors, and panic upon detecting modified system partitions.

This repository adapts the kernel tree to enable full custom software support by identifying and neutralizing conflicting security subsystems:

### 1. Samsung DEFEX LSM Neutralization
- **Issue**: Samsung DEFEX (Defensive Execution) is an in-kernel LSM that monitors system calls, enforces path restrictions, and kills unauthorized root processes.
- **Modification**: Disabled `CONFIG_SECURITY_DEFEX` and all associated sub-options (`CONFIG_SECURITY_DEFEX_DSMS`, `CONFIG_SECURITY_DEFEX_PROCA_CHECK`, `CONFIG_SECURITY_DEFEX_INTEGRITY`) in defconfig.
- **Header Stubs**: Added inline no-op fallback stubs to `kernel-5.10/include/linux/defex.h` and `kernel-5.10/security/samsung/defex_lsm/include/linux/defex.h` for core symbols (`task_defex_enforce`, `task_defex_zero_creds`, `defex_syscall_enter`, `task_defex_user_exec`, `defex_load_rules`), ensuring clean compilation of referencing drivers without runtime blocking.

### 2. UH / RKP / KDP Removal
- **Issue**: Samsung Real-time Kernel Protection (RKP), Kernel Data Protection (KDP), and User-mode Hypervisor (UH) enforce read-only credentials at the hypervisor level. Modifying task credentials (UID/GID) or page tables triggers hypervisor panics or crashes.
- **Modification**: Disabled `CONFIG_UH`, `CONFIG_RKP`, `CONFIG_KDP`, and test configurations in `a07_00_defconfig` and `a07_defconfig`, eliminating read-only credential traps and hypervisor restrictions.

### 3. dm-verity Forced Panic Disarmed
- **Issue**: Samsung's custom verity debug handler (`drivers/md/dm-verity-debug.c`) invokes a hard kernel panic (`panic("dmv corrupt")`) upon encountering hash mismatches on system partitions, preventing custom ROMs and GSIs from booting.
- **Modification**: Neutralized `panic("dmv corrupt")` in `verity_handle_err_hex_debug()`, while preserving necessary debugging structures and allowing system bringup without kernel crashes on modified partitions.

### 4. Samsung DSMS Telemetry Disabled
- **Issue**: Device Security Management Service (DSMS) logs and transmits security and root-attempt telemetry.
- **Modification**: Disabled `CONFIG_SECURITY_DSMS` across defconfigs.

### 5. SELinux Development Support
- **Issue**: Strict enforcing SELinux without development overrides hinders initial custom ROM and GSI bringup.
- **Modification**: Enabled `CONFIG_SECURITY_SELINUX_DEVELOP=y` across defconfigs, permitting permissive command-line flags (`androidboot.selinux=permissive`) and runtime `setenforce 0`.

### 6. Build Script Python 3 Modernization
- **Issue**: Upstream configuration merger `kernel-5.10/scripts/gen_build_config.py` contained Python 2 `print` statements, causing syntax errors on modern build systems.
- **Modification**: Updated all print statements to Python 3 syntax (`print(...)`), ensuring compatibility with modern Linux distributions.

---

## Directory Structure

```
/
├── build_kernel.sh          # Primary kernel build script
├── kernel/                  # Google ACK build framework (build/build.sh)
├── kernel-5.10/             # Linux 5.10.233 source tree
│   ├── arch/arm64/configs/  # Device defconfigs (a07_00_defconfig, a07_defconfig)
│   ├── drivers/md/          # Neutralized dm-verity debug handler
│   ├── include/linux/       # DEFEX fallback header stubs
│   └── scripts/             # Python 3 modernized build configuration generator
├── vendor/                  # MediaTek external kernel modules and drivers
├── .github/workflows/       # GitHub Actions CI build workflow (build-kernel.yml)
├── README.md                # Repository documentation
└── .gitignore               # Build output and archive exclusion rules
```

---

## Building Locally

### Prerequisites

On Ubuntu 20.04/22.04 LTS or Debian 11/12, install required build tools and cross-compilers:

```bash
sudo apt-get update
sudo apt-get install -y \
  bc bison build-essential ccache curl flex git \
  libelf-dev libncurses5-dev libssl-dev lz4 \
  python3 python-is-python3 rsync \
  gcc-aarch64-linux-gnu gcc-arm-linux-gnueabi \
  device-tree-compiler clang lld llvm
```

### Toolchain Setup

The kernel utilizes Google Android GKI Clang `clang-r416183b` (Android 12 release):

```bash
# Create toolchain destination
mkdir -p prebuilts-master/clang/host/linux-x86/clang-r416183b

# Download and extract official prebuilt Clang toolchain
curl -fsSL "https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/android12-release/clang-r416183b.tar.gz" \
  | tar -xz -C prebuilts-master/clang/host/linux-x86/clang-r416183b

# Symlink prebuilts for kernel build runner
ln -sfn ../prebuilts-master kernel/prebuilts-master
```

### Build Execution

Set the required environment flags and launch the build:

```bash
# Ensure scripts have execute permissions
chmod +x build_kernel.sh kernel/build/build.sh kernel/build/_setup_env.sh

# Export non-hermetic toolchain and timestamp configuration
export HERMETIC_TOOLCHAIN=0
export BUILDTOOLS_PREBUILT_BIN=""
export SOURCE_DATE_EPOCH=$(date +%s)

# Start build process
./build_kernel.sh
```

### Build Outputs

Compiled artifacts are produced in `out/target/product/a07/obj/KERNEL_OBJ/`:
- **Kernel Image**: `out/target/product/a07/obj/KERNEL_OBJ/kernel-5.10/arch/arm64/boot/Image.gz`
- **Kernel Modules**: `out/target/product/a07/obj/KERNEL_OBJ/*.ko`
- **Build Configuration**: `out/target/product/a07/obj/KERNEL_OBJ/build.config`
- **Kconfig Symbol Configuration**: `out/target/product/a07/obj/KERNEL_OBJ/include/config/auto.conf`

---

## Continuous Integration (GitHub Actions)

This repository includes automated CI via **GitHub Actions** (`.github/workflows/build-kernel.yml`).

### Workflow Details
- **Triggers**:
  - `push` to branch `main`
  - `pull_request` to branch `main`
  - Manual dispatch via `workflow_dispatch`
- **Runner**: `ubuntu-22.04`
- **Pipeline Steps**:
  1. Checks out repository source tree (`actions/checkout@v4`).
  2. Provisions build packages and cross-compilers via `apt-get`.
  3. Downloads and unpacks Android GKI Clang `clang-r416183b`.
  4. Executes `./build_kernel.sh` with `HERMETIC_TOOLCHAIN=0`.
  5. Packages and uploads build artifacts using `actions/upload-artifact@v4`.

### Artifacts in CI
Workflow runs automatically publish artifacts under the name `samsung-a07-kernel-build`, containing:
- `Image*` kernel binaries
- Out-of-tree and in-tree `.ko` kernel modules
- Build configuration logs and symbols

---

## License

- The Linux kernel source is licensed under the terms of the **GNU General Public License version 2 (GPLv2)**.
- MediaTek and Samsung driver components are subject to their respective upstream licenses.
