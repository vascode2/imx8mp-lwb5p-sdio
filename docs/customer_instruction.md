# Sterling LWB5+ (SDIO/UART M.2) on NXP i.MX8M Plus EVK — Customer Build Guide

Bring up **WiFi (`wlan0`) over SDIO** and **Bluetooth (`hci0`) over UART** on the
i.MX8MP EVK with an Ezurio/Summit **Sterling LWB5+** (`CYW4373A0`) module, using the
**official Ezurio `meta-summit-radio`** layer and **Summit's backports stack**
(not mainline drivers).

> This guide does **not** depend on any private or forked repository. It uses the
> public Ezurio layer (`https://github.com/Ezurio/meta-summit-radio`, branch
> `lrd-14.8.0.x`) and adds the LWB5+ integration files by hand. Everything needed
> to reproduce the build is contained in this document. For deeper rationale per
> step, see [`customer_instruction_detailed.md`](customer_instruction_detailed.md).

## Hardware & Scope

| Item | Value |
|------|-------|
| Board | NXP **i.MX8M Plus** EVK (`imx8mpevk`) |
| Radio | Ezurio/Summit **Sterling LWB5+**, **CYW4373A0 (BCM4373)** |
| Form factor | **M.2** module in the EVK's SDIO-capable M.2 slot |
| WLAN | **SDIO** → `usdhc1` (mmc@30b40000) → `wlan0` |
| Bluetooth | **UART** → `uart1` → `hci0` |

## Architecture

```
                    i.MX8M Plus EVK
  ┌───────────────────────────────────────────────┐
  │  M.2 slot  ── SDIO ──► usdhc1 (mmc@30b40000) │  WLAN (wlan0)
  │      └────── UART ──► uart1                   │  BT (hci0)
  │  SD card   ── SDIO ──► usdhc2 (mmc@30b50000) │  boot device (mmcblk1)
  │  eMMC      ── SDIO ──► usdhc3 (mmc@30b60000) │  non-removable
  └───────────────────────────────────────────────┘
```

Stock `imx8mp-evk.dts` keeps `usdhc1` (M.2) **disabled** and sets `uart1` BT to
`nxp,88w8997-bt` (wrong chip — NXP/Marvell 88W8997, not Broadcom CYW4373A0).
Both are fixed by the **custom device tree** added in §D.

## Software Stack

| Component | Version |
|-----------|---------|
| Yocto / Poky | 5.2 ("walnascar", DISTRO_VERSION 5.2.2) |
| DISTRO | `fsl-imx-wayland` |
| Machine | `imx8mpevk` |
| Kernel | `linux-imx` 6.12.x (NXP BSP) |
| U-Boot | `u-boot-imx` 2025.04 |
| Summit Connectivity Stack | **LRD-REL-14.8.0.x** |

---

## A — Host Setup

64-bit Linux build host (Ubuntu 24.04 tested), ~150 GB free.

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

## B — Create the Project + Fetch BSP Sources

```bash
mkdir -p ~/Projects/yocto_projects/lwb5p-imx8mp
cd ~/Projects/yocto_projects/lwb5p-imx8mp
repo init -u https://github.com/nxp-imx/imx-manifest -b imx-6.12.49-2.2.0 -m imx-6.12.49-2.2.0.xml
repo sync -j$(nproc)
```

This creates a `sources/` tree (poky, meta-imx, etc.).

## C — Clone the Official Ezurio Layer (Public, Not a Fork)

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp/sources
git clone -b lrd-14.8.0.x https://github.com/Ezurio/meta-summit-radio.git
cd ..
```

This branch contains the Summit backports, firmware, and supplicant/networkmanager
recipes — but **no** i.MX8MP LWB5+ device-tree / U-Boot / kernel-config / image
integration. You add those by hand in the next step.

## D — Add the LWB5+ Integration Files by Hand

All paths are relative to `sources/meta-summit-radio/meta-summit-radio/`.

### D.1 Custom Device Tree

`recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts`

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

### D.2 Kernel Recipe (Build the DTB and Align the Defconfig)

`recipes-kernel/linux/linux-imx_%.bbappend`

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

### D.3 U-Boot Config Fragment

`recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg`

```
# Boot the Summit LWB5+ device tree (SDIO Wi-Fi + CYW4373A0 Bluetooth) by
# default instead of the stock imx8mp-evk.dtb.
CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"
```

### D.4 U-Boot Recipe (Load the Config Fragment)

`recipes-bsp/u-boot/u-boot-imx_%.bbappend`

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# Default to booting the Summit LWB5+ device tree (SDIO Wi-Fi + CYW4373A0 BT)
SRC_URI:append:imx8mpevk = " file://imx8mpevk-lwb5plus.cfg"
```

### D.5 Custom Image Recipe

`recipes-packages/images/lwb5p-lrd14.bb`

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

> Tip — create them in one shot:
> ```bash
> cd ~/Projects/yocto_projects/lwb5p-imx8mp/sources/meta-summit-radio/meta-summit-radio
> mkdir -p recipes-kernel/linux/files/dts \
>          recipes-bsp/u-boot/files \
>          recipes-packages/images
> # then paste each block above into the corresponding file
> ```

---

## E — Create the Build Directory

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp
DISTRO=fsl-imx-wayland MACHINE=imx8mpevk source ./imx-setup-release.sh -b build-imx8mpevk-wal-lwbp
cd ~/Projects/yocto_projects/lwb5p-imx8mp
```

## F — Configure the Build

Add the layer and prefer Summit's supplicant/hostapd/networkmanager:

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

## G — Build

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp
source sources/base/setup-environment build-imx8mpevk-wal-lwbp
bitbake lwb5p-lrd14
```

Output:
```
build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
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

> If the count isn't 48 / the file is missing, the write didn't persist: make sure
> the card is unmounted before flashing, and/or try another SD card (some cards
> silently drop writes and report success).

## I — Verify on the EVK

Boot from the SD card, log in as root:

```bash
iw dev            # expect wlan0
hciconfig -a      # expect hci0 UP RUNNING
dmesg | grep -iE 'brcm|hci_uart|hci0|brcmfmac'
```

Expected log markers: `brcmfmac ... BCM4373/0`, `hci_uart_bcm`, `Bluetooth: hci0: BCM4373A0`.

---

## Appendix — Why These Changes Are Needed

**WLAN** — Stock `imx8mp-evk.dts` leaves the M.2 SDIO slot (`usdhc1`) `status =
"disabled"`, so the radio never enumerates. The custom DTS includes NXP's
`imx8mp-evk-usdhc1-m2.dts`, which enables `usdhc1`, adds `reg_usdhc1_vmmc`
(WLAN_EN, GPIO2_IO06), `usdhc1_pwrseq` (reset GPIO2_IO10), `wifi_wake_host`, and
disables the PCIe controller/phy.

**Bluetooth** — Stock `uart1` declares `compatible = "nxp,88w8997-bt"` (NXP/Marvell),
so Broadcom's `hci_uart`/`btbcm` won't bind. Overriding it to
`cypress,cyw4373a0-bt` makes the Summit-backported `hci_bcm.c` bind and load the
`BCM4373A0*.hcd` firmware.

**Kernel config — `CONFIG_CFG80211` must be `=m` (module).**
- Built-in `=y` trips the backports `checks.h` `#error` ("You must not have cfg80211
  built into your kernel").
- `# CONFIG_CFG80211 is not set` compiles `net_device.ieee80211_ptr` out of the
  kernel (it's guarded by `IS_ENABLED(CONFIG_CFG80211)`), which the backports'
  own cfg80211 requires — the WLAN build then fails (`nl80211.c: struct net_device
  has no member 'ieee80211_ptr'`) and the whole backports recipe aborts before the
  Bluetooth modules are packaged → **no `hci0` either**.
- `=m` is the correct setting: a module keeps `ieee80211_ptr` in the kernel **and**
  avoids the `checks.h` `#error`.

| Option | Setting | Reason |
|--------|---------|--------|
| `CONFIG_CFG80211`  | `=m` | backports WLAN needs `ieee80211_ptr`; `=m` dodges `checks.h` |
| `CONFIG_MAC80211`  | off | Summit backports provides its own |
| `CONFIG_WLAN`      | off | no in-tree wireless LAN drivers |
| `CONFIG_BT`        | off | Summit backports provides its own BT stack (`lwb` defconfig) |
| `CONFIG_FW_LOADER_USER_HELPER_FALLBACK` | off | no sysfs firmware fallback |
| `CONFIG_IMX_SDMA`  | `=m` | DMA for the BT UART path |

**Why patch `imx_v8_defconfig` directly (not a config fragment)?** `linux-imx` runs
`do_copy_defconfig` **after** `do_kernel_configme`, which **overwrites `.config`**
with the stock `imx_v8_defconfig`, wiping any fragment applied by configme. Also,
`DELTA_KERNEL_DEFCONFIG`'s `merge_config.sh` **cannot turn `=y` off** — it only
appends duplicate `# … is not set` lines, which kconfig ignores. So the settings
must be written into `imx_v8_defconfig` itself (the `do_patch:append` in §D.2).

**Why the U-Boot fragment?** `CONFIG_DEFAULT_FDT_FILE` makes U-Boot load
`imx8mp-evk-lwb5plus.dtb` **by default**, so no manual `setenv fdtfile` is needed
at boot.

**Why `IMAGE_BOOT_FILES:append` in the image?** U-Boot only loads a dtb that's
present on the FAT boot partition. Without this entry the custom dtb is built but
never copied there, so U-Boot can't find it.
