# Sterling LWB5+ (SDIO/UART M.2) on NXP i.MX8M Plus EVK — Build Guide

This guide takes you from **nothing** to a bootable Yocto image for the NXP
**i.MX8M Plus EVK** that brings up:

- **WiFi (`wlan0`)** over **SDIO** — Ezurio/Summit Sterling **LWB5+** (`CYW4373A0`) in the M.2 slot
- **Bluetooth (`hci0`)** over **UART** (`uart1`)

Everything is a copy-paste block. Run each block top-to-bottom in the order shown.

> For the full technical design and background (device tree details, layer changes,
> architecture), see [`IMX8MP_LWB5_SDIO_UART_INTEGRATION.md`](IMX8MP_LWB5_SDIO_UART_INTEGRATION.md).

> **Bottom line:** we use **Ezurio/Summit's own wireless & Bluetooth stack** (the
> `meta-summit-radio` backports), not the mainline kernel drivers. That is why we
> disable the kernel's built-in wireless/BT and set `CONFIG_CFG80211=m` (a module,
> not built-in — see §8).

---

## Prerequisites

- A 64-bit Linux build machine (this was validated on Ubuntu 24.04), ~150 GB free disk.
- The **i.MX8MP EVK** with an **M.2 SDIO-capable slot** (the `M.2` connector, not PCIe-only).
- The **Sterling LWB5+** module installed in that M.2 slot.
- A serial console to the EVK and an SD card to boot the image.

---

## Part A — One-time machine setup

```bash
# Install host packages for Yocto (Ubuntu/Debian)
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential chrpath \
    socat cpio python3 python3-pip python3-venv python3-pexpect xz-utils debianutils \
    iputils-ping python3-git python3-jinja2 python3-subunit zstd liblz4-tool file \
    libssl-dev libgmp-dev libmpc-dev bison flex meson ninja-build pkg-config \
    bmap-tools mtools dosfstools

# 'repo' tool (Google)
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+rx ~/bin/repo
export PATH=~/bin:$PATH
```

> The `export PATH` only lasts for that shell. Add it to `~/.bashrc` if you want it permanent:
> `echo 'export PATH=~/bin:$PATH' >> ~/.bashrc`

---

## Part B — Fetch the BSP sources (repo manifest)

```bash
# Pick a workspace directory
mkdir -p ~/yocto && cd ~/yocto

# Initialize with the NXP i.MX BSP manifest we validated against.
# This creates a 'sources/' tree with poky + NXP/meta layers.
repo init -u https://github.com/nxp-imx/imx-manifest \
          -b imx-6.12.49-2.2.0 \
          -m imx-6.12.49-2.2.0.xml
repo sync -j$(nproc)
```

> If `repo` complains about missing platform tools, run `sudo apt install -y git-core` and retry.

---

## Part C — Add the Ezurio/Summit radio layer

`meta-summit-radio` supplies the **Summit backports** (kernel modules), **firmware**,
**supplicant/networkmanager/hostapd**, and our **custom image recipe**.

```bash
# Clone the Summit radio layer (branch lrd-14.8.0.x = stack version 14.8.0.6)
cd ~/yocto/sources
git clone -b lrd-14.8.0.x \
    https://github.com/Ezurio/meta-summit-radio.git
cd ~/yocto
```

> The case of the directory name matters when we refer to it in `bblayers.conf`
> (`meta-summit-radio/meta-summit-radio`).

---

## Part D — Create the build directory and add the files

```bash
# Create & configure the build directory for imx8mpevk / fsl-imx-wayland.
# This runs imx-setup-release.sh which generates build-imx8mpevk-wal-lwbp/.
DISTRO=fsl-imx-wayland MACHINE=imx8mpevk source ./imx-setup-release.sh \
    -b build-imx8mpevk-wal-lwbp
cd ~/yocto
```

Now create the four files below. **Use the exact contents shown.**

### D.1 — Custom device tree (`sources/meta-summit-radio/meta-summit-radio/recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts`)

```bash
mkdir -p sources/meta-summit-radio/meta-summit-radio/recipes-kernel/linux/files/dts
cat > sources/meta-summit-radio/meta-summit-radio/recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts <<'EOF'
/dts-v1/;
#include "imx8mp-evk-usdhc1-m2.dts"

/* LWB5+ Bluetooth on uart1: use Broadcom HCI, not NXP 88W8997 */
&uart1 {
    bluetooth {
        compatible = "cypress,cyw4373a0-bt";
    };
};
EOF
```

> **Why:** the stock `imx8mp-evk.dts` keeps the M.2 SDIO slot (`usdhc1`) **disabled** and
> declares BT on `uart1` with the **wrong** chip (`nxp,88w8997-bt`). Including NXP's
> `imx8mp-evk-usdhc1-m2.dts` enables SDIO (WLAN) and swaps the BT compatible string so the
> Broadcom `hci_uart`/`btbcm` driver binds.

### D.2 — Kernel bbappend (`sources/meta-summit-radio/meta-summit-radio/recipes-kernel/linux/linux-imx_%.bbappend`)

```bash
cat > sources/meta-summit-radio/meta-summit-radio/recipes-kernel/linux/linux-imx_%.bbappend <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# Custom LWB5+ device trees to install into the kernel source
LWB_DTS_FILES = "dts/imx8mp-evk-lwb5plus.dts"
SRC_URI += "file://dts/imx8mp-evk-lwb5plus.dts"
KERNEL_DEVICETREE:append:imx8mpevk = " freescale/imx8mp-evk-lwb5plus.dtb"

# Summit/Ezurio backports (kernel-module-lwb-if-backports) ship their own
# cfg80211, mac80211 and Bluetooth modules. checks.h only #errors when a symbol
# is BUILT-IN (=y). A module (=m) keeps net_device.ieee80211_ptr in the kernel
# (guarded by IS_ENABLED(CONFIG_CFG80211)) so the backports' own cfg80211 compiles,
# and sets CONFIG_*_MODULE (not CONFIG_*) so checks.h is satisfied.
#
# linux-imx bakes imx_v8_defconfig in via do_copy_defconfig (which overwrites
# configme fragments), and merge_config.sh can't turn =y symbols to 'not set'.
# So patch imx_v8_defconfig itself here.
do_patch:append() {
    if [ -d "${S}/arch/arm64/boot/dts/freescale" ]; then
        for i in ${LWB_DTS_FILES}; do
            if [ -f "${UNPACKDIR}/${i}" ]; then
                bbnote "Installing LWB5+ device tree ${i}"
                install -m 0644 "${UNPACKDIR}/${i}" "${S}/arch/arm64/boot/dts/freescale/"
            fi
        done
    fi
    DEF="${S}/arch/arm64/configs/imx_v8_defconfig"
    if [ -f "$DEF" ]; then
        bbnote "LWB5+: aligning imx_v8_defconfig with backports guidance"
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
        # Force symbols to our desired state (last match wins in kconfig)
        printf 'CONFIG_CFG80211=m\n' >> "$DEF"
        printf '# CONFIG_MAC80211 is not set\n' >> "$DEF"
        printf '# CONFIG_WLAN is not set\n' >> "$DEF"
        printf '# CONFIG_BT is not set\n' >> "$DEF"
        printf '# CONFIG_FW_LOADER_USER_HELPER_FALLBACK is not set\n' >> "$DEF"
    fi
}
EOF
```

### D.3 — U-Boot bbappend + config (boot the custom dtb by default)

```bash
mkdir -p sources/meta-summit-radio/meta-summit-radio/recipes-bsp/u-boot/files
cat > sources/meta-summit-radio/meta-summit-radio/recipes-bsp/u-boot/u-boot-imx_%.bbappend <<'EOF'
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://imx8mpevk-lwb5plus.cfg"
EOF

cat > sources/meta-summit-radio/meta-summit-radio/recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg <<'EOF'
CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"
EOF
```

> **Why:** makes U-Boot load the custom dtb (`imx8mp-evk-lwb5plus.dtb`) by default, so no
> manual `setenv fdtfile ...` is needed at boot.

### D.4 — Custom image recipe (`sources/meta-summit-radio/meta-summit-radio/recipes-packages/images/lwb5p-lrd14.bb`)

```bash
mkdir -p sources/meta-summit-radio/meta-summit-radio/recipes-packages/images
cat > sources/meta-summit-radio/meta-summit-radio/recipes-packages/images/lwb5p-lrd14.bb <<'EOF'
DESCRIPTION = "Sterling LWB5+ SDIO/UART M.2 sample image"
LICENSE = "MIT"
inherit core-image
export IMAGE_BASENAME = "${PN}"

IMAGE_FEATURES += "ssh-server-dropbear splash"
IMAGE_FEATURES:remove = "tools-profile tools-debug tools-testapps"

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
EOF
```

---

## Part E — Configure the build directory

Add the Summit layer and our distro/machine preferences.

```bash
cd ~/yocto
# Append the Summit layer to bblayers.conf
tee -a build-imx8mpevk-wal-lwbp/conf/bblayers.conf <<'EOF'
BBLAYERS += "${BSPDIR}/sources/meta-summit-radio/meta-summit-radio"
EOF
```

Ensure `build-imx8mpevk-wal-lwbp/conf/local.conf` contains at least:

```bash
cat >> build-imx8mpevk-wal-lwbp/conf/local.conf <<'EOF'

# Use Summit's supplicant/hostapd/networkmanager, not mainline
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

> `LWB_REGDOMAIN` sets the regulatory domain (change to your region, e.g. `EU`).

---

## Part F — Build the image

```bash
cd ~/yocto
# Re-enter the build environment
source sources/base/setup-environment build-imx8mpevk-wal-lwbp

# Build the LWB5+ image (this includes kernel + backports + firmware)
bitbake lwb5p-lrd14
```

Output image (on success):

```
build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
```

> First build takes a long time (hours). Rebuilds after that are incremental.

---

## Part G — Flash the SD card

Insert an SD card into the build machine and find its device (e.g. `/dev/sdX` — **verify
this carefully**, it will be overwritten).

```bash
# Replace /dev/sdX with YOUR card device. DANGER: data on it will be destroyed.
sudo zstdcat \
    build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz \
    | sudo dd of=/dev/sdX bs=1M conv=fsync status=progress
sync
```

Boot the EVK from the SD card and connect a serial console.

---

## Part H — Verify WiFi + Bluetooth on the EVK

```bash
# WiFi (SDIO)
iw dev                 # expect: phy0 / wlan0
ip link set wlan0 up
iw dev wlan0 scan | head

# Bluetooth (UART)
hciconfig -a           # expect: hci0 UP RUNNING
bluetoothctl           # interactive pairing / scan

# Kernel log — expect Broadcom HCI binding + LWB5+ HCD firmware load
dmesg | grep -iE 'brcm|hci_uart|hci0'
```

Working kernel log shows something like `BCM4373A0-sdio*…hcd` being loaded and an
`hci0` device.

---

## Reference

### §8 — Why `CONFIG_CFG80211=m` (not built-in, not "not set")

The Summit backports build **their own** cfg80211 / mac80211 / Bluetooth modules.
`checks.h` refuses only when a symbol is **built into** the kernel (`=y`):

```c
#if defined(CONFIG_CFG80211) && defined(CPTCFG_CFG80211_MODULE)
#error "You must not have cfg80211 built into your kernel if you want to enable it"
#endif
```

- `=y` → `CONFIG_CFG80211` is defined → **`#error` fires.** Not allowed.
- `=m` → only `CONFIG_CFG80211_MODULE` is defined → **no `#error`,** and
  `net_device.ieee80211_ptr` stays in the kernel (guarded by `IS_ENABLED(CONFIG_CFG80211)`).
- `is not set` → the `ieee80211_ptr` member **compiles out** → the backports WLAN fails to
  build (`nl80211.c: no member 'ieee80211_ptr'`), aborting the whole recipe before the
  Bluetooth modules are even packaged (so **no `hci0`**).

Resulting kernel config:

| Option | Setting | Reason |
|--------|---------|--------|
| `CONFIG_CFG80211` | `=m` | backports WLAN + keeps `ieee80211_ptr`; avoids `checks.h` |
| `CONFIG_MAC80211` | off | Summit backports provides it |
| `CONFIG_WLAN` | off | no in-tree wireless LAN drivers |
| `CONFIG_BT` | off | Summit backports provides its own BT stack (`lwb` defconfig) |
| `CONFIG_FW_LOADER_USER_HELPER_FALLBACK` | off | no sysfs firmware fallback |
| `CONFIG_IMX_SDMA` | `=m` | DMA for the BT UART path |

> Because `linux-imx` copies the defconfig in via `do_copy_defconfig` (overwriting
> `configme` fragments), these settings must be written **into `imx_v8_defconfig` itself**
> in `do_patch:append` (Part D.2) — a normal config fragment would be discarded.

### Files created (summary)

| File | Purpose |
|------|---------|
| `recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts` | custom DT: SDIO wifi + BT on uart1 |
| `recipes-kernel/linux/linux-imx_%.bbappend` | build dtb, patch defconfig |
| `recipes-bsp/u-boot/u-boot-imx_%.bbappend` | boot custom dtb by default |
| `recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg` | `CONFIG_DEFAULT_FDT_FILE` |
| `recipes-packages/images/lwb5p-lrd14.bb` | custom image recipe |

### Architecture

```
                    i.MX8M Plus EVK
  ┌───────────────────────────────────────────────┐
  │  M.2 slot ── SDIO ──► usdhc1 (mmc@30b40000)  │  WLAN (wlan0)
  │      └────── UART ──► uart1                   │  BT (hci0)
  │  SD card   ── SDIO ──► usdhc2 (mmc@30b50000) │  boot device (mmcblk1)
  │  eMMC      ── SDIO ──► usdhc3 (mmc@30b60000) │  non-removable
  └───────────────────────────────────────────────┘
```

### Reference documents

- Ezurio USB-dongle app note (original): <https://www.ezurio.com/documentation/application-note-sterling-lwb5-usb-dongle-nxp-i-m8x-evk-integration-guide>
- Ezurio `meta-summit-radio` (`lrd-14.8.0.x`): <https://github.com/Ezurio/meta-summit-radio>
- NXP i.MX8MP EVK device trees in `linux-imx`: `imx8mp-evk.dts`, `imx8mp-evk-usdhc1-m2.dts`

---

## Troubleshooting

- **Build fails: `struct net_device has no member named 'ieee80211_ptr'`** → the kernel was
  built with `CONFIG_CFG80211 is not set`. Re-check Part D.2: it must be `CONFIG_CFG80211=m`.
- **`hciconfig` shows nothing** → confirm the backports built its BT modules
  (`hci_uart.ko`, `btbcm.ko` in the image), and that the custom dtb is active
  (`dmesg | grep -i bluetooth`). If `usdhc1` is still disabled in the boot dtb, WLAN won't
  come up either.
- **Wrong device tree at boot** → confirm `CONFIG_DEFAULT_FDT_FILE` (Part D.3) took effect;
  temporarily, at the U-Boot prompt: `setenv fdtfile imx8mp-evk-lwb5plus.dtb; saveenv; boot`.
