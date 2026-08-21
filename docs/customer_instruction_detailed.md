# Sterling LWB5+ (SDIO/UART M.2) on NXP i.MX8M Plus EVK — Detailed Build Guide

A step-by-step, fork-free guide to bringing up **WiFi (`wlan0`) over SDIO** and
**Bluetooth (`hci0`) over UART** on the NXP **i.MX8M Plus EVK** with an
Ezurio/Summit **Sterling LWB5+** (`CYW4373A0 / BCM4373`) M.2 module, built with
**Yocto/Poky** and **Summit's backports stack** (not mainline drivers).

This document uses **only public repositories**:

- NXP BSP manifest: `https://github.com/nxp-imx/imx-manifest` (branch `imx-6.12.49-2.2.0`)
- Ezurio radio layer: `https://github.com/Ezurio/meta-summit-radio` (branch `lrd-14.8.0.x`)

No private or forked repository is required. The LWB5+ integration files
(custom device tree, kernel/U-Boot bbappends, image recipe) are created by hand
from the listings in §D.

Each section states **what** to do and **why** it is necessary, so the build can
be reproduced and debugged without external references.

> For a condensed version, see [`customer_instruction.md`](customer_instruction.md).
> For the fastest path (using a pre-integrated fork), see
> [`quick_instruction.md`](quick_instruction.md).

---

## 1. Scope and Hardware

| Item | Value |
|------|-------|
| SoC / Board | NXP **i.MX8M Plus** EVK (`imx8mpevk`) |
| Radio | Ezurio/Summit **Sterling LWB5+**, **CYW4373A0 (BCM4373)** chipset |
| Form factor | **M.2** module in the EVK's M.2 SDIO-capable slot |
| WLAN interface | **SDIO** → `usdhc1` (mmc@30b40000) → `wlan0` |
| Bluetooth interface | **UART** → `uart1` → `hci0` |

**Why the M.2 SDIO/UART variant.** The LWB5+ is available as a USB dongle and as
an M.2 module. This guide targets the **M.2 SDIO/UART** module because the EVK's
M.2 slot can route SDIO (not just PCIe), and NXP ships a device tree for it
(`imx8mp-evk-usdhc1-m2.dts`). WLAN goes over the high-throughput SDIO bus; BT goes
over a dedicated UART — the standard split for this chipset family.

**Why not the USB dongle path here.** The USB path can additionally suffer from
power/enumeration issues on `usb@38100000`; the SDIO/UART path is the cleaner
route on this EVK.

## 2. Software Stack

| Component | Version |
|-----------|---------|
| Yocto / Poky | 5.2 ("walnascar", DISTRO_VERSION 5.2.2) |
| DISTRO | `fsl-imx-wayland` |
| Machine | `imx8mpevk` |
| Kernel | `linux-imx` 6.12.x (NXP BSP) |
| U-Boot | `u-boot-imx` 2025.04 |
| Summit Connectivity Stack | **LRD-REL-14.8.0.x** |

**Why Summit backports instead of mainline drivers.** The LWB5+ (CYW4373A0) is
supported by Ezurio/Summit's **backports** package, which ships its own
`cfg80211`, `mac80211`, `brcmfmac` (WLAN) and `hci_uart`/`btbcm`/`bluetooth` (BT)
modules matched to the firmware. Using backports means the in-kernel
cfg80211/mac80211/Bluetooth stacks must be **disabled** in the kernel (see §D.2)
to avoid symbol conflicts, while `CONFIG_CFG80211` is kept as a **module** so a
struct member the backports need stays compiled in.

**Why `fsl-imx-wayland`.** This is NXP's standard multimedia/graphics distro for
the i.MX8 family and pulls in the NXP GPU/Wayland stack. It is unrelated to WiFi/BT
but is the conventional base for an i.MX8MP EVK image.

## 3. Architecture Overview

```
                    i.MX8M Plus EVK
  ┌───────────────────────────────────────────────┐
  │  M.2 slot  ── SDIO ──► usdhc1 (mmc@30b40000) │  WLAN (wlan0)
  │      └────── UART ──► uart1                   │  BT (hci0)
  │  SD card   ── SDIO ──► usdhc2 (mmc@30b50000) │  boot device (mmcblk1)
  │  eMMC      ── SDIO ──► usdhc3 (mmc@30b60000) │  non-removable
  └───────────────────────────────────────────────┘
```

### 3.1 Why the Radio Does Not Appear Out of the Box

Booting the stock image with `fdtfile=imx8mp-evk.dtb` yields no `wlan0` and no
`hci0` (`iw dev` empty, `hciconfig -a` empty, `lsusb` shows only root hubs since
USB is not used for the M.2 SDIO module). Both root causes are **device-tree**, not
firmware:

1. **WLAN** — The M.2 SDIO slot (`usdhc1`) is `status = "disabled"` in the stock
   dtb, so the mmc/SDIO interface for the radio is never brought up. No SDIO
   enumeration → no `brcmfmac` bind → no `wlan0`.
2. **Bluetooth** — `uart1` is declared for BT but with `compatible =
   "nxp,88w8997-bt"` (the NXP/Marvell 88W8997, not the Broadcom CYW4373A0). The
   Broadcom `hci_uart`/`btbcm` driver cannot bind to that compatible string, so
   no `hci0` is created.

The image already contains the correct **SDIO firmware**
(`lwb5plus-sdio-div-firmware`) and the Summit **backports** WiFi/BT drivers — they
simply have no matching DT node to bind to. The custom device tree in §D.1 fixes
both, and the kernel-config alignment in §D.2 ensures the backports modules build.

## 4. Prerequisites

- 64-bit Linux build host (Ubuntu 24.04 tested), ~150 GB free.
- i.MX8MP EVK with M.2 SDIO slot, Sterling LWB5+ installed, serial console, SD card.

---

## A — Host Setup

```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential chrpath \
    socat cpio python3 python3-pip python3-venv python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 python3-subunit zstd liblz4-tool file \
    libssl-dev libgmp-dev libmpc-dev bison flex meson ninja-build pkg-config \
    bmap-tools mtools dosfstools
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+rx ~/bin/repo
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc
export PATH=~/bin:$PATH
```

**Why.** The `apt` list is the standard Yocto host dependency set (the `essential`
Yocto packages plus `gawk`, `diffstat`, `chrpath`, `python3-*`, `zstd`, `liblz4-tool`,
etc.) needed by Poky 5.2. `repo` is Google's multi-manifest tool used by NXP's BSP
to fetch many git repositories at pinned revisions. `bmap-tools` is used later to
flash the SD card efficiently; `mtools`/`dosfstools` are used to inspect the FAT
boot partition.

## B — Create the Project + Fetch BSP Sources

```bash
mkdir -p ~/Projects/yocto_projects/lwb5p-imx8mp
cd ~/Projects/yocto_projects/lwb5p-imx8mp
repo init -u https://github.com/nxp-imx/imx-manifest -b imx-6.12.49-2.2.0 -m imx-6.12.49-2.2.0.xml
repo sync -j$(nproc)
```

**Why.** `repo init` pins the NXP i.MX manifest to release `imx-6.12.49-2.2.0`
(kernel 6.12.x, U-Boot 2025.04, Poky 5.2). `repo sync` materialises the `sources/`
tree (`poky`, `meta-imx`, `meta-freescale`, `meta-openembedded`, …) at the exact
revisions the BSP was validated against. Mixing in a different manifest revision
risks a kernel/U-Boot/backports API mismatch.

## C — Clone the Official Ezurio Layer (Public, Not a Fork)

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp/sources
git clone -b lrd-14.8.0.x https://github.com/Ezurio/meta-summit-radio.git
cd ..
```

**Why.** The `lrd-14.8.0.x` branch of `meta-summit-radio` provides:

- `summit-backports` — the backports recipe that builds the LWB5+ WiFi/BT kernel
  modules (`brcmfmac`, `cfg80211`, `mac80211`, `hci_uart`, `btbcm`, `bluetooth`).
- `radio-firmware` — the SDIO firmware and `BCM4373A0*.hcd` BT firmware blobs.
- `summit-supplicant`, `summit-hostapd`, `summit-networkmanager` — Summit's
  connectivity userspace, plus `regLWB5plus-*` regulatory/reg files.

What this branch **does not** provide: an i.MX8MP LWB5+ device tree, the
U-Boot default-FDT setting, the `imx_v8_defconfig` alignment for backports, or a
sample image that copies the custom dtb onto the FAT boot partition. Those four
gaps are filled by hand in §D.

## D — Add the LWB5+ Integration Files by Hand

All paths below are relative to
`sources/meta-summit-radio/meta-summit-radio/`. Create the directories first:

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp/sources/meta-summit-radio/meta-summit-radio
mkdir -p recipes-kernel/linux/files/dts \
         recipes-bsp/u-boot/files \
         recipes-packages/images
```

### D.1 Custom Device Tree

File: `recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts`

```dts
/*
 * Custom device tree for the i.MX8MP EVK with a Summit/Ezurio
 * Sterling LWB5+ (CYW4373A0) M.2 module in the SDIO M.2 slot.
 *
 * Based on NXP's imx8mp-evk-usdhc1-m2.dts (enables the M.2 SDIO
 * wireless slot on usdhc1) and overrides the Bluetooth HCI node on
 * uart1 so that the Summit backports brcm/hci_uart driver binds to
 * the LWB5+ CYW4373A0 BT instead of the (wrong) NXP 88W8997.
 *
 * SPDX-License-Identifier: (GPL-2.0 OR MIT)
 */
/dts-v1/;

#include "imx8mp-evk-usdhc1-m2.dts"

/* Bluetooth is on UART1. The LWB5+ is a Broadcom/Cypress CYW4373A0,
 * so declare the matching compatible so hci_uart/btbcm bind and load
 * the BCM4373A0*.hcd firmware.
 */
&uart1 {
	bluetooth {
		compatible = "cypress,cyw4373a0-bt";
	};
};
```

**Why.** The DT `#include`s NXP's `imx8mp-evk-usdhc1-m2.dts`, which already does
the heavy lifting for WLAN on `usdhc1`:

- enables `usdhc1` for SDIO;
- adds `reg_usdhc1_vmmc` (WLAN_EN on GPIO2_IO06) and `usdhc1_pwrseq`
  (reset on GPIO2_IO10) so the module is powered and reset at the right time;
- adds `wifi_wake_host` and `mmc-pwrseq-simple`;
- disables `reg_pcie0`, `pcie`, `pcie_phy` (the SDIO-vs-PCIe mux on the M.2 slot).

That leaves only the **Bluetooth** node to fix. Stock `uart1` carries
`compatible = "nxp,88w8997-bt"`; the LWB5+ is a Broadcom/Cypress CYW4373A0, so the
node is overridden to `cypress,cyw4373a0-bt`. That exact compatible string is what
the Summit-backported `hci_bcm.c` binds to for the LWB5+ (with
`.no_uart_clock_set = true`), which then loads the `BCM4373A0*.hcd` firmware over
the UART. This single dtb therefore enables **both** WiFi (SDIO) and Bluetooth
(UART).

### D.2 Kernel Recipe (Build the DTB and Align the Defconfig)

File: `recipes-kernel/linux/linux-imx_%.bbappend`

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# Custom LWB5+ device trees to install into the kernel source
LWB_DTS_FILES = "dts/imx8mp-evk-lwb5plus.dts"

SRC_URI += "file://dts/imx8mp-evk-lwb5plus.dts"

# Build the custom dtb for this machine
KERNEL_DEVICETREE:append:imx8mpevk = " freescale/imx8mp-evk-lwb5plus.dtb"

# Summit/Ezurio backports (kernel-module-lwb-if-backports) ships its own
# cfg80211, mac80211 and Bluetooth modules. Its checks.h only errors out when
# these are BUILT INTO the kernel (=y); a module (=m) is fine (CONFIG_X_MODULE,
# not CONFIG_X, is defined, so the #if is false). Keeping CONFIG_CFG80211=m is
# also required so net_device.ieee80211_ptr (guarded by IS_ENABLED(CONFIG_CFG80211))
# stays present for the backports' own cfg80211 to compile.
#
# linux-imx bakes imx_v8_defconfig in via do_copy_defconfig (which overwrites
# configme fragments) and DELTA_KERNEL_DEFCONFIG's merge_config.sh cannot force
# =y symbols to 'not set'. So we patch imx_v8_defconfig itself here.
do_patch:append() {
    if [ -d "${S}/arch/arm64/boot/dts/freescale" ]; then
        for i in ${LWB_DTS_FILES}; do
            if [ -f "${UNPACKDIR}/${i}" ]; then
                bbnote "Installing LWB5+ device tree ${i}"
                install -m 0644 "${UNPACKDIR}/${i}" "${S}/arch/arm64/boot/dts/freescale/"
            fi
        done
    fi
    # Align imx_v8_defconfig with the LWB5+ radio bring-up guidance:
    #   CFG80211   -> module (=m)  : backports WLAN needs ieee80211_ptr present; =m dodges checks.h
    #   MAC80211   -> off          : Summit backports provides its own
    #   WLAN       -> off          : no in-tree wireless LAN drivers
    #   BT         -> off          : Summit backports provides its own BT stack (lwb config)
    #   FW_LOADER_USER_HELPER_FALLBACK -> off
    #   IMX_SDMA   -> module (=m)
    DEF="${S}/arch/arm64/configs/imx_v8_defconfig"
    if [ -f "$DEF" ]; then
        bbnote "LWB5+: aligning imx_v8_defconfig with LWB5+ backports guidance"
        sed -i \
            -e '/^CONFIG_CFG80211=y$/d' \
            -e '/^CONFIG_CFG80211=m$/d' \
            -e '/^# CONFIG_CFG80211 is not set$/d' \
            -e '/^CONFIG_MAC80211=y$/d' \
            -e '/^CONFIG_CFG80211_WEXT=y$/d' \
            -e '/^CONFIG_CFG80211_REQUIRE_SIGNED_REGDB=y$/d' \
            -e '/^CONFIG_CFG80211_USE_KERNEL_REGDB_KEYS=y$/d' \
            -e '/^CONFIG_CFG80211_DEFAULT_PS=y$/d' \
            -e '/^CONFIG_CFG80211_CRDA_SUPPORT=y$/d' \
            -e '/^CONFIG_MAC80211_LEDS=y$/d' \
            -e '/^CONFIG_MAC80211_HAS_RC=y$/d' \
            -e '/^CONFIG_MAC80211_RC_MINSTREL=y$/d' \
            -e '/^CONFIG_MAC80211_RC_DEFAULT_MINSTREL=y$/d' \
            -e '/^CONFIG_MAC80211_RC_DEFAULT=/d' \
            -e '/^CONFIG_WLAN=y$/d' \
            -e '/^CONFIG_BT=y$/d' \
            -e '/^CONFIG_BT=m$/d' \
            -e '/^CONFIG_FW_LOADER_USER_HELPER_FALLBACK=y$/d' \
            -e '/^CONFIG_IMX_SDMA=y$/c\CONFIG_IMX_SDMA=m' \
            "$DEF"
        # Force the symbols to our desired state (append, last match wins in kconfig)
        printf 'CONFIG_CFG80211=m\n' >> "$DEF"
        printf '# CONFIG_MAC80211 is not set\n' >> "$DEF"
        printf '# CONFIG_WLAN is not set\n' >> "$DEF"
        printf '# CONFIG_BT is not set\n' >> "$DEF"
        printf '# CONFIG_FW_LOADER_USER_HELPER_FALLBACK is not set\n' >> "$DEF"
    fi
}
```

**Why — three jobs in one bbappend.**

1. **Install the custom DTS** into the kernel source tree
   (`arch/arm64/boot/dts/freescale/`) during `do_patch`, and add it to
   `KERNEL_DEVICETREE` so the kernel build actually compiles
   `freescale/imx8mp-evk-lwb5plus.dtb`.
2. **Align `imx_v8_defconfig`** with the backports requirements (see the table
   and pitfall below).
3. Keep `CONFIG_IMX_SDMA=m` because the i.MX SDMA engine is used by the BT UART
   path; a module is sufficient and avoids forcing it built-in.

**Why `CONFIG_CFG80211` must be `=m` (module), specifically.** The Summit
backports provide their own `cfg80211`/`mac80211`/Bluetooth. Their `checks.h`
refuses to build when these are **built into** the kernel:

```c
#if defined(CONFIG_CFG80211) && defined(CPTCFG_CFG80211_MODULE)
#error "You must not have cfg80211 built into your kernel if you want to enable it"
#endif
```

- With `CONFIG_CFG80211=m`, the kernel defines `CONFIG_CFG80211_MODULE`
  (not `CONFIG_CFG80211`), so `defined(CONFIG_CFG80211)` is **false** and the
  `#error` does **not** fire. ✓
- With `CONFIG_CFG80211=y` (built-in), `CONFIG_CFG80211` **is** defined, so the
  `#error` fires and backports aborts. ✗
- With `# CONFIG_CFG80211 is not set`, the kernel compiles out
  `net_device.ieee80211_ptr` (it is guarded by
  `IS_ENABLED(CONFIG_CFG80211)`). The backports' own `cfg80211` code requires
  that member, so `nl80211.c` fails with
  `struct net_device has no member 'ieee80211_ptr'`, the whole backports recipe
  aborts, and **the Bluetooth modules are never packaged either** → no `hci0`. ✗

So `=m` is the only setting that simultaneously keeps `ieee80211_ptr` in the
kernel **and** avoids the `checks.h` `#error`.

| Option | Setting | Reason |
|--------|---------|--------|
| `CONFIG_CFG80211`  | `=m` (module) | backports WLAN needs `net_device.ieee80211_ptr` present; `=m` dodges `checks.h` |
| `CONFIG_MAC80211`  | off (`is not set`) | Summit backports provides its own |
| `CONFIG_WLAN`      | off (`is not set`) | no in-tree wireless LAN drivers |
| `CONFIG_BT`        | off (`is not set`) | Summit backports provides its own BT stack (`lwb` defconfig) |
| `CONFIG_FW_LOADER_USER_HELPER_FALLBACK` | off | no sysfs firmware fallback |
| `CONFIG_IMX_SDMA`  | `=m` | DMA for the BT UART path |

**Why patch `imx_v8_defconfig` directly instead of using a config fragment.**
This is the critical, non-obvious part. In `linux-imx`:

- `do_copy_defconfig` runs **after** `do_kernel_configme` and **overwrites `.config`**
  with the stock `imx_v8_defconfig`. Any symbol a fragment applied during
  configme is therefore wiped before the kernel is actually built.
- `DELTA_KERNEL_DEFCONFIG` uses `merge_config.sh`, which **cannot turn a `=y`
  symbol off** — it only appends duplicate `# … is not set` lines, which kconfig
  ignores (the existing `=y` wins).

Consequently the desired settings must be written **into `imx_v8_defconfig`
itself**, which is what the `do_patch:append` does: it strips any existing
`CFG80211`/`MAC80211`/`BT`/`WLAN`/`FW_LOADER_USER_HELPER_FALLBACK` lines and then
appends the desired final states (last match wins in kconfig).

### D.3 U-Boot Config Fragment

File: `recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg`

```
# Boot the Summit LWB5+ device tree (SDIO Wi-Fi + CYW4373A0 Bluetooth) by
# default instead of the stock imx8mp-evk.dtb.
CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"
```

**Why.** `CONFIG_DEFAULT_FDT_FILE` sets the dtb U-Boot loads by default. With it
pointing at `imx8mp-evk-lwb5plus.dtb`, the board boots straight into the custom DT
with no manual `setenv fdtfile …; saveenv` at the U-Boot prompt. (Without this,
U-Boot would load the stock `imx8mp-evk.dtb` and the radio would not come up —
exactly the §3.1 problem.)

### D.4 U-Boot Recipe (Load the Config Fragment)

File: `recipes-bsp/u-boot/u-boot-imx_%.bbappend`

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# Default to booting the Summit LWB5+ device tree (SDIO Wi-Fi + CYW4373A0 BT)
SRC_URI:append:imx8mpevk = " file://imx8mpevk-lwb5plus.cfg"
```

**Why.** This makes the U-Boot build pick up the `.cfg` fragment from §D.3 through
`FILESEXTRAPATHS`/`SRC_URI`, so `CONFIG_DEFAULT_FDT_FILE` is applied during the
U-Boot compile. The `:imx8mpevk` override scopes the change to this machine only.

### D.5 Custom Image Recipe

File: `recipes-packages/images/lwb5p-lrd14.bb`

```bitbake
DESCRIPTION = "Sterling LWB5+ SDIO/UART M.2 (diversity antenna) sample image"
LICENSE = "MIT"

inherit core-image

export IMAGE_BASENAME = "${PN}"

# Ensure our custom LWB5+ device tree is copied to the FAT boot partition.
# U-Boot's CONFIG_DEFAULT_FDT_FILE loads 'imx8mp-evk-lwb5plus.dtb'; without this
# entry it is built but never placed on the boot partition.
IMAGE_BOOT_FILES:append = " imx8mp-evk-lwb5plus.dtb"

IMAGE_FEATURES += "\
	ssh-server-dropbear \
	splash \
	"

IMAGE_FEATURES:remove = "\
	tools-profile \
	tools-debug \
	tools-testapps \
	"

IMAGE_INSTALL += "\
	iproute2 \
	rng-tools \
	ca-certificates \
	tzdata \
	htop \
	ethtool \
	iperf3 \
	tcpdump \
	iw \
	kernel-module-lwb-if-backports \
	lwb5plus-sdio-div-firmware \
	summit-supplicant \
	summit-networkmanager \
	summit-networkmanager-nmcli \
	libedit \
	"
```

**Why — `IMAGE_BOOT_FILES:append`.** U-Boot (via `CONFIG_DEFAULT_FDT_FILE`) can
only load a dtb that physically exists on the FAT boot partition. The kernel
build produces `imx8mp-evk-lwb5plus.dtb`, but it is **not** copied to the boot
partition unless the image recipe lists it in `IMAGE_BOOT_FILES`. Without this
one line the dtb is built yet never placed where U-Boot looks, so the board still
boots the stock dtb. This is the most easily missed of the five files.

**Why — the package set.**

- `kernel-module-lwb-if-backports` — the LWB5+ WiFi **and** Bluetooth kernel
  modules (backports). This is the single package that supplies both `wlan0`
  (via `brcmfmac`/`cfg80211`) and `hci0` (via `hci_uart`/`btbcm`/`bluetooth`).
- `lwb5plus-sdio-div-firmware` — the SDIO firmware for WLAN and the
  `BCM4373A0*.hcd` firmware for BT; without it the modules bind but fail to
  initialise.
- `summit-supplicant` / `summit-networkmanager` / `summit-networkmanager-nmcli` —
  Summit's matched userspace for managing WiFi (and the regulatory domain).
- `iw` — manual WiFi inspection; `ethtool`/`iperf3`/`tcpdump`/`htop` — basic
  bring-up/diagnostics; `rng-tools` — entropy for BT pairing; `ca-certificates`/
  `tzdata`/`iproute2` — common base utilities.

`IMAGE_FEATURES:remove` drops NXP's heavy debug/profile/testapp feature sets to
keep the image lean for a radio bring-up image.

---

## E — Create the Build Directory

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp
DISTRO=fsl-imx-wayland MACHINE=imx8mpevk source ./imx-setup-release.sh -b build-imx8mpevk-wal-lwbp
cd ~/Projects/yocto_projects/lwb5p-imx8mp
```

**Why.** `imx-setup-release.sh` is NXP's wrapper around Poky's `setup-environment`:
it bakes `DISTRO=fsl-imx-wayland` and `MACHINE=imx8mpevk` into
`conf/local.conf`, accepts an EULA, and creates the `build-imx8mpevk-wal-lwbp`
build directory with the i.MX layers pre-configured. Pinning `MACHINE=imx8mpevk`
is what later makes the `:imx8mpevk` overrides in §D.2/D.4 apply.

## F — Configure the Build

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp
tee -a build-imx8mpevk-wal-lwbp/conf/bblayers.conf <<'EOF'
BBLAYERS += "${BSPDIR}/sources/meta-summit-radio/meta-summit-radio"
EOF
cat >> build-imx8mpevk-wal-lwbp/conf/local.conf <<'EOF'

PREFERRED_PROVIDER_wpa-supplicant = "summit-supplicant"
PREFERRED_PROVIDER_wpa-supplicant-cli = "summit-supplicant-cli"
PREFERRED_PROVIDER_wpa-supplicant-passphrase = "summit-supplicant-passphrase"
PREFERRED_PROVIDER_hostapd = "summit-hostapd"
PREFERRED_PROVIDER_networkmanager = "summit-networkmanager"
PREFERRED_PROVIDER_networkmanager-cli = "summit-networkmanager-cli"
PREFERRED_RPROVIDER_wireless-regdb-static = "wireless-regdb"
LWB_REGDOMAIN = "US"
EOF
```

**Why — `BBLAYERS`.** The cloned `meta-summit-radio` layer is not added by the NXP
manifest, so it must be appended to `bblayers.conf` for BitBake to see the
backports/firmware/supplicant recipes.

**Why — `PREFERRED_PROVIDER_*`.** Poky and `meta-networking` already provide
`wpa-supplicant`, `hostapd` and `networkmanager`. The LWB5+ must use **Summit's**
versions (matched to the backports stack), so these `PREFERRED_PROVIDER` lines
force BitBake to choose the Summit recipes for the virtuals
(`wpa-supplicant`, `hostapd`, `networkmanager`) and their CLI/passphrase
companions. Without them the image would pull the generic versions, which are not
 ABI-matched to the backports modules.

**Why — `PREFERRED_RPROVIDER_wireless-regdb-static = "wireless-regdb"`.** Resolves
the regulatory-database virtual provider to the standard `wireless-regdb`
package; the LWB5+ reg files are installed separately by the firmware packages.

**Why — `LWB_REGDOMAIN = "US"`.** Sets the default 802.11 regulatory domain baked
into the image (country code for channel/tx-power rules). Change to your region
(e.g. `EU`, `JP`) as needed.

## G — Build

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp
source sources/base/setup-environment build-imx8mpevk-wal-lwbp
bitbake lwb5p-lrd14
```

**Why.** `lwb5p-lrd14` is the custom image recipe from §D.5. Bitbaking it pulls in
the kernel (with the custom DT + defconfig alignment), U-Boot (with the default-FDT
fragment), the backports modules, the firmware, and Summit's userspace, then
assembles the wic image with `IMAGE_BOOT_FILES` placing the custom dtb on the boot
partition.

Output:
```
build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.bmap
```

## H — Flash the SD Card

```bash
# Find the SD device (~15G, RM=1). Replace /dev/sdX below.
lsblk -o NAME,SIZE,RM,MOUNTPOINT

# MUST unmount before writing (writes to a mounted card are dropped).
sudo umount /dev/sdX* 2>/dev/null

IMG=build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
BMAP=build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.bmap
sudo bmaptool copy --bmap "$BMAP" "$IMG" /dev/sdX
sync

# VERIFY before ejecting — expect 48 dtbs and the lwb5plus file present.
sudo mount -o ro /dev/sdX1 /mnt
ls /mnt | grep -c '\.dtb'                 # 48
ls -l /mnt/imx8mp-evk-lwb5plus.dtb
sudo umount /mnt
sudo eject /dev/sdX
```

**Why — unmount first.** Writing to a mounted SD card is dropped by the kernel
block layer; `bmaptool`/`dd` will appear to succeed but the data won't persist.
`umount` before flashing is mandatory.

**Why — `bmaptool`.** It uses the `.bmap` to copy only the used blocks (fast) and
verifies checksums, far safer than a raw `dd`.

**Why — verify 48 dtbs + the lwb5plus file.** This is the single best check that
the write actually persisted **and** that §D.5's `IMAGE_BOOT_FILES` worked. If the
count isn't 48 or `imx8mp-evk-lwb5plus.dtb` is missing, the write didn't persist
(try another SD card — some silently drop writes) or the image recipe's
`IMAGE_BOOT_FILES:append` line is wrong/missing. Catching it here avoids a dead
boot on the EVK.

## I — Verify on the EVK

Boot from the SD card, log in as root:

```bash
iw dev            # expect wlan0
hciconfig -a      # expect hci0 UP RUNNING
dmesg | grep -iE 'brcm|hci_uart|hci0|brcmfmac'
```

Expected kernel-log markers: `brcmfmac ... BCM4373/0` (WLAN over SDIO bound and
firmware loaded), `hci_uart_bcm` and `Bluetooth: hci0: BCM4373A0` (BT over UART
bound and `BCM4373A0*.hcd` loaded).

**Why these checks.** `iw dev` confirms the SDIO `brcmfmac`→`cfg80211` path
produced a `wlan0`; `hciconfig -a` confirms the UART `hci_uart`→`btbcm` path
produced an `hci0`. If `wlan0` is missing, revisit §D.1 (usdhc1 enable) and §D.2
(`CONFIG_CFG80211=m`). If `hci0` is missing, revisit §D.1 (`cypress,cyw4373a0-bt`)
and §D.2 (`CONFIG_BT` off so backports BT is packaged) — recall that an aborted
backports WLAN build also drops the BT modules.

Manual fallback to prove the DT fix without rebuilding U-Boot:
```
setenv fdtfile imx8mp-evk-usdhc1-m2.dtb   # or imx8mp-evk-lwb5plus.dtb
saveenv
boot
```

## J — Troubleshooting and Notes

- **No `hci0` even though `wlan0` is present (or vice versa).** The backports
  recipe builds WLAN and BT together; if the WLAN build aborts (e.g.
  `nl80211.c: struct net_device has no member 'ieee80211_ptr'`), the BT modules
  are never packaged. The fix is `CONFIG_CFG80211=m` (not `is not set`), with
  `MAC80211`/`WLAN`/`BT` off — see §D.2. Earlier builds that used `CONFIG_CFG80211
  is not set` hit exactly this on kernel 6.12; `=m` resolves it and both
  `brcmfmac`/`cfg80211` (`wlan0`) and `hci_uart`/`btbcm`/`bluetooth` (`hci0`)
  build cleanly.
- **EEPROM/reg files.** These are installed by the firmware packages
  (`regLWB5plus-aarch64` etc.) and picked up at runtime; no manual step needed.
- **M.2 vs USB.** This guide uses the M.2 **SDIO/UART** module, not the USB dongle.
  The USB path can additionally suffer power/enumeration issues on
  `usb@38100000`.

---

## Appendix A — Summary of Files Added by Hand

| File | Purpose |
|------|---------|
| `recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts` | custom DT (SDIO WiFi + CYW4373A0 BT) |
| `recipes-kernel/linux/linux-imx_%.bbappend` | build the dtb; align `imx_v8_defconfig` (`CONFIG_CFG80211=m`, BT/WLAN/MAC80211 off) |
| `recipes-bsp/u-boot/u-boot-imx_%.bbappend` | load the U-Boot config fragment |
| `recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg` | `CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"` |
| `recipes-packages/images/lwb5p-lrd14.bb` | sample image; `IMAGE_BOOT_FILES:append` puts the dtb on the boot partition |

## Appendix B — Reference Documents and Upstream Sources

- Ezurio USB-dongle app note (original, for context):
  [Sterling LWB5+ USB Dongle – i.MX8M EVK Integration Guide][appnote-usb]
- Ezurio `meta-summit-radio` (branch `lrd-14.8.0.x`):
  <https://github.com/Ezurio/meta-summit-radio>
- NXP i.MX BSP manifest (`imx-6.12.49-2.2.0`):
  <https://github.com/nxp-imx/imx-manifest>
- NXP i.MX8M Plus EVK device-tree sources (inside `linux-imx`):
  - `arch/arm64/boot/dts/freescale/imx8mp-evk.dts`
  - `arch/arm64/boot/dts/freescale/imx8mp-evk-usdhc1-m2.dts` (SDIO/PCIe M.2 slot)

[appnote-usb]: https://www.ezurio.com/documentation/application-note-sterling-lwb5-usb-dongle-nxp-i-m8x-evk-integration-guide
