# First Successful Build Review (RAXE500 Repo)

This document is a practical preflight and troubleshooting guide for getting a **first successful firmware build** in this repository.

## 1) Understand the build entrypoint

- Top-level `make` uses the root `Makefile` and defaults `PROFILE` to `RAX220`.
- The root build copies `prebuilt/*` into profile directories, then calls `build/Makefile` targets in order:
  1. main build
  2. `pre_buildimage`
  3. `gpl`
  4. `acos`
  5. `just_buildimage`

Because of this sequencing, a failed first build may come from either compile stages or late image assembly.

## 2) Critical prerequisites

1. Install the required cross-toolchain under `/opt/toolchains` (README example points at `aarch64-gcc-9.2-linux-4.19-glibc-2.30-binutils-2.32`).
2. Install host packages listed in `README.md` (`build-essential`, `gawk`, `autoconf`, `automake`, `autotools-dev`, `libtool`, `liblzo2-dev`).
3. Ensure Git LFS artifacts are available. This repo tracks `prebuilt/**` and multiple binary/archive extensions through LFS.

## 3) Why first-build failures are common in this repo

- **Profile coupling:** many files are selected by profile name (`config_<PROFILE>.mk`, `ambitCfg_WW_<PROFILE>.h`).
- **Toolchain coupling:** `make.common` hardcodes Linux 4.19 ARM/AARCH64 toolchain patterns under `/opt/toolchains`.
- **Large mixed source tree:** kernel, drivers, RDP, and userspace packages are built with legacy and modern rules together.
- **Image assembly is separate:** rootfs creation and packaging happen in `targets/buildFS` and later image targets, so compile success does not guarantee image success.

## 4) Preflight checks before first full build

Run these from repo root:

```bash
# profile and orchestrator sanity
make -n | head -n 40

# verify toolchain directories expected by make.common exist
ls -d /opt/toolchains/*4.19* || true

# confirm prebuilt/profile files are present and not unresolved LFS pointers
ls prebuilt/
file prebuilt/RAX220 || true

# quick check that profile-dependent userspace config exists
ls userspace/project/acos/config_RAX220.mk
ls userspace/project/gpl/config_RAX220.mk
```

## 5) First build strategy (recommended)

1. Start with explicit profile and controlled jobs:

```bash
make PROFILE=RAX220 BRCM_MAX_JOBS=8
```

2. If it fails, isolate stage by stage:

```bash
make -f build/Makefile PROFILE=RAX220 kernelbuild
make -f build/Makefile PROFILE=RAX220 modbuild
make -f build/Makefile PROFILE=RAX220 gpl
make -f build/Makefile PROFILE=RAX220 acos
make -f build/Makefile PROFILE=RAX220 just_buildimage
```

3. If reproducibility is poor, retry with single-job builds to avoid race-related failures:

```bash
make PROFILE=RAX220 BRCM_MAX_JOBS=1
```

## 6) Where to look when it fails

- **Top orchestration:** `Makefile`, `build/Makefile`.
- **Profile/toolchain resolution:** `make.common`.
- **Userspace package failure:** `userspace/ap/gpl/Makefile` and the failing package subdir.
- **Rootfs/image failure:** `targets/buildFS` and `build/Makefile` image targets.

## 7) What to learn next after first success

- Learn the profile file under `targets/<PROFILE>/<PROFILE>` and the generated `.config` flow.
- Learn how `acos_link.sh` and `gpl_link.sh` map profile-specific configs into build paths.
- Learn `targets/buildFS` rootfs assembly details (library layout, init scripts, module handling).
- Learn cleanup paths (`clean1`, `clean`, `cleanall`) to recover from partial builds.

