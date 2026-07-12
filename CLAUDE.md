# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build system overview

OpenWrt uses a GNU Make build system with Kconfig-based configuration (same frontend as the Linux kernel). The build produces a complete embedded Linux firmware image: it downloads sources, builds a cross-compilation toolchain, builds the Linux kernel, and builds user-space packages.

### Quickstart

```sh
./scripts/feeds update -a       # fetch all feed sources
./scripts/feeds install -a      # create symlinks into package/feeds/
make menuconfig                 # interactive configuration
make -j$(nproc)                 # full build
```

### Common build targets

| Command | Purpose |
|---------|---------|
| `make menuconfig` | Interactive target/package/option selection |
| `make -j$(nproc)` | Full parallel build |
| `make V=s` | Verbose build (full compiler/linker output) |
| `make V=sc` | Show commands but suppress warnings |
| `make defconfig` | Apply default configuration |
| `make oldconfig` | Update .config for new Kconfig options |
| `make kernel_menuconfig` | Configure Linux kernel options |
| `make package/<name>/compile V=s` | Build one package (verbose) |
| `make package/<name>/clean` | Clean a single package |
| `make package/<name>/prepare` | Download, unpack, and patch a package (stops before compile) |
| `make package/<name>/install` | Install a package's build output into staging |
| `make target/linux/compile` | Build just the kernel and its modules |
| `make clean` | Remove build_dir, staging_dir, bin |
| `make dirclean` | Also remove staging_dir/host and host build dirs |
| `make distclean` | Remove everything including .config, feeds, dl |

### Config options

- `CONFIG_USE_APK=y` — use APK package manager (new default) instead of OPKG
- `CONFIG_TARGET_PER_DEVICE_ROOTFS=y` — generate per-device root filesystems (needed for device-specific images)
- `CONFIG_DEVEL=y` — developer options (enable package source dir overrides)
- `CONFIG_BUILDBOT=y` — defaults for automated builds (auto-clean, per-device rootfs, etc.)
- `CONFIG_EXTERNAL_TOOLCHAIN` — use an external toolchain instead of building one

### Debugging builds

- `make package/<name>/compile V=s` — verbose output for one package
- `make -j1 V=s` — sequential verbose build (useful when parallel build hides errors)
- Check `logs/` directory if `CONFIG_BUILD_LOG=y` is set
- Use `make package/<name>/prepare` then inspect `build_dir/` for the unpacked source
- `make package/<name>/compile QUILT=1` — enable quilt patch refresh mode
- `make -j$(nproc) 2>&1 | tee build.log` — capture full build output

## Repository architecture

### Build pipeline order

tools → toolchain → target (kernel + modules) → packages → firmware images

This ordering is enforced by `include/toplevel.mk` and `rules.mk`. Each stage builds on the previous: tools provide host-side utilities (cmake, quilt, etc.), the toolchain provides cross-compilers for the target architecture, and so on.

### Core build system (`include/`)

Every Makefile in the repo includes `$(TOPDIR)/rules.mk` (or `$(INCLUDE_DIR)/...`). The key include files:

| File | Purpose |
|------|---------|
| `toplevel.mk` | Top-level targets: config, menuconfig, build ordering |
| `rules.mk` | Global variables: cross-compiler paths, ARCH, BOARD, SUBTARGET, flags, host tools |
| `target.mk` | Per-target defaults: DEFAULT_PACKAGES, DEVICE_TYPE, subtarget resolution |
| `package.mk` | Package build framework: PKG_BUILD_DIR, download, unpack, patch, configure, compile, install |
| `kernel.mk` | Kernel build framework: LINUX_DIR, kernel config, module compilation |
| `kernel-build.mk` | Kernel download, prepare, configure, compile pipeline with stamp tracking |
| `kernel-defaults.mk` | Default implementations of Kernel/Prepare, Kernel/Configure, Kernel/Compile |
| `image.mk` | Firmware image generation: IMG_PREFIX, KDIR, DTS_DIR, per-device rootfs |
| `image-commands.mk` | Image build primitives: append-dtb, append-kernel, append-rootfs, sysupgrade metadata |
| `host-build.mk` | Host-side tool build rules (for tools/ and host dependencies) |
| `download.mk` | Source tarball download, git cloning, hash verification |
| `feeds.mk` | Feed repository management, package index generation for opkg/apk |
| `hardening.mk` | Security hardening flags (PIE, RELRO, stack protector, FORTIFY_SOURCE) |
| `version.mk` | Release version templates (currently 25.12-SNAPSHOT) |
| `rootfs.mk` | Root filesystem assembly, mklibs, device table, init script packing |
| `package-bin.mk` | IPK/APK binary package creation rules |
| `scan.mk` | Metadata scanning: walks package/ and target/ directories to build Kconfig |
| `depends.mk` | Dependency resolution framework for build stamps |
| `quilt.mk` | Kernel and package patch management (quilt integration) |

### Directory layout

- `package/` — all core packages organized by category: base-files, boot (uboot), kernel (mac80211, mt76, button-hotplug), libs, network (config/ipv6/services/utils), system (uci, ubus, procd, opkg), utils, firmware
- `target/linux/<target>/` — per-SoC target definitions
- `target/linux/generic/` — shared kernel config, patches, and files across all targets
- `toolchain/` — cross-compilation toolchain (binutils, gcc, musl/glibc, kernel-headers, gdb, mold)
- `tools/` — host build tools (cmake, bison, quilt, patch, libtool, etc.)
- `scripts/` — helper scripts (feeds, config, metadata, image generation, checkpatch, kernel bump)
- `config/` — high-level Kconfig files (Config-build.in, Config-devel.in, Config-images.in, Config-kernel.in)
- `feeds.conf.default` — external package feed definitions (packages, luci, routing, telephony, video)

### Target architecture

Each target under `target/linux/` represents a supported SoC/platform. Each contains:

| Path | Purpose |
|------|---------|
| `Makefile` | Defines ARCH, BOARD, SUBTARGETS, FEATURES, KERNEL_PATCHVER, DEFAULT_PACKAGES |
| `config-<ver>` | Kernel .config fragment for this target |
| `patches-<ver>/` | Platform-specific kernel patches |
| `base-files/` | Board-specific init scripts (uci-defaults, preinit) |
| `image/Makefile` | Device definitions and image generation rules |
| `image/<subtarget>.mk` | Per-subtarget device image definitions |
| `dts/` | Device tree source files (.dts and .dtsi) |
| `modules.mk` | Kernel module package definitions specific to this target |

For example, `target/linux/mediatek/` has subtargets filogic, mt7622, mt7623, mt7629 each with their own `<subtarget>.mk` image definitions.

#### Adding support for a new board

1. Add or reference a DTS file in `target/linux/<target>/dts/` (or upstream kernel)
2. Add a `Device/<name>` block in `target/linux/<target>/image/<subtarget>.mk` defining KERNEL, IMAGES, IMAGE/*, DEVICE_DTS, DEVICE_TITLE, DEVICE_PACKAGES
3. Add board-specific files in `target/linux/<target>/base-files/etc/uci-defaults/` if needed
4. Optionally add platform-specific kernel modules via `modules.mk`

### Package Makefile structure

A typical package Makefile under `package/`:

```makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=foo
PKG_VERSION:=1.0
PKG_RELEASE:=1
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://example.com
PKG_HASH:=abc123...

include $(INCLUDE_DIR)/package.mk

define Package/foo
  SECTION:=utils
  CATEGORY:=Utilities
  DEPENDS:=+libbar        # hard dep; +PACKAGE_libfoo:libbar for conditional dep
  TITLE:=Foo utility
  URL:=https://example.com
endef

define Package/foo/description
  Longer description text.
endef

define Build/Configure
  # Default: runs ./configure. Override for cmake, meson, or custom.
endef

define Package/foo/install
  $(INSTALL_DIR) $(1)/usr/bin
  $(INSTALL_BIN) $(PKG_BUILD_DIR)/foo $(1)/usr/bin/
endef

$(eval $(call BuildPackage,foo))
```

Key variables: `PKG_BUILD_DIR`, `PKG_INSTALL_DIR`, `PKG_BUILD_PARALLEL`, `PKG_CONFIG_DEPENDS`, `PKG_BUILD_DEPENDS` (build-time deps), `PKG_FIXUP` (autoreconf, patch-libtool).

Build variant support: `define Package/foo/description` can differ per variant, and `BuildPackage` accepts the variant name as second argument.

### Kernel build details

- Kernel version is set in `include/kernel-version.mk`. Currently kernel 6.12.
- Per-target kernel configs live in `target/linux/<target>/config-6.12`
- Generic kernel config is in `target/linux/generic/config-6.12`
- Patches are organized in:
  - `target/linux/generic/backport-6.12/` — upstream backports
  - `target/linux/generic/hack-6.12/` — OpenWrt-specific workarounds
  - `target/linux/generic/pending-6.12/` — patches pending upstream submission
  - `target/linux/<target>/patches-6.12/` — platform-specific patches
- Kernel bump script: `scripts/kernel_bump.sh`
- Kernel config format: `target/linux/<target>/kernel-6.12` files contain version and hash

### Package managers

OpenWrt now defaults to **APK** (`CONFIG_USE_APK=y`) replacing OPKG. The package index formats differ:
- APK: `packages.adb` files
- OPKG: `Packages.gz` files

The feed config is generated from `include/feeds.mk` via `FeedSourcesAppendAPK` / `FeedSourcesAppendOPKG`.

### Configuration system

Kconfig files are generated from package and target metadata at build time:
- `tmp/.config-package.in` — generated from package Makefile metadata via `scripts/package-metadata.pl`
- `tmp/.config-target.in` — generated from target metadata via `scripts/target-metadata.pl`
- `tmp/.config-feeds.in` — generated from feeds

The config is also partially applied during the build: `scripts/kconfig.pl` checks whether .config is out of sync with the generated Kconfig and warns.

## Commit and patch style

- Commit messages follow: `<area>: <short description>` (e.g., `mediatek: add support for ...`, `kernel: bump linux to ...`, `build: fix ...`)
- All commits must include `Signed-off-by:` line (Developer Certificate of Origin)
- Use `scripts/checkpatch.pl` to check patches before submission (Linux kernel style)
- Kernel patches should follow the categories: backport/ (from upstream), hack/ (OpenWrt-specific), pending/ (for upstream submission)

## Dependency syntax

Package dependencies in `DEPENDS:=`:
- `+libfoo` — hard dependency (build + install libfoo)
- `+PACKAGE_libfoo:libbar` — conditional dependency (only if libfoo is selected)
- `@TARGET_foo` — available only on specific target
- `@HAS_TESTING_KERNEL` — depends on target capability
- `!BUSYBOX_DEFAULT_foo:busybox` — conflict with BusyBox applet

## Git identity

This is a GeekWRT fork. Commits must use the following author identity:

```sh
git config user.email "geekwrt@users.noreply.github.com"
git config user.name "GeekWRT"
```

Apply per-repo when working in either openwrt main repo or feeds/luci.

## Key utilities and scripts

- `./scripts/feeds` — manage external package feeds
- `./scripts/package-metadata.pl` — generate package dependency/config info
- `./scripts/target-metadata.pl` — generate target Kconfig info
- `./scripts/config/conf` and `./scripts/config/mconf` — kconfig frontend binaries
- `./scripts/getver.sh` — compute revision string from git
- `./scripts/get_source_date_epoch.sh` — compute source date epoch for reproducible builds
- `./scripts/diffconfig.sh` — generate a minimal config diff from a full .config
- `./scripts/kernel_bump.sh` — automate kernel version bumps
- `./scripts/remote-gdb` — remote GDB debugging on target
- `./scripts/ipkg-build` — build .ipk package files
- `./scripts/gen_image_generic.sh` — generate generic disk images

## Related repositories

- **LuCI web interface**: https://github.com/openwrt/luci
- **Package feeds**: https://github.com/openwrt/packages
- **Routing packages**: https://github.com/openwrt/routing
- **Video packages**: https://github.com/openwrt/video
