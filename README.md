# Sterling LWB5+ (SDIO/UART M.2) on NXP i.MX8M Plus EVK — Quick Build Guide

Bring up **WiFi (`wlan0`) over SDIO** and **Bluetooth (`hci0`) over UART** on the
i.MX8MP EVK with an Ezurio/Summit **Sterling LWB5+** (`CYW4373A0`) module, using
**Summit's backports stack** (not mainline drivers).

This guide uses a **fork of `meta-summit-radio`** that already contains the custom
device tree, U-Boot default, kernel config (`CONFIG_CFG80211=m`), and the image
boot-partition fix. No manual file editing is needed.

> For a fork-free version (official Ezurio layer + manual integration files), see
> [`docs/customer_instruction.md`](docs/customer_instruction.md) or the deeper
> [`docs/customer_instruction_detailed.md`](docs/customer_instruction_detailed.md).

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
`nxp,88w8997-bt` (wrong chip). A custom DT (in the fork) fixes both.

## Prerequisites

- 64-bit Linux build host (Ubuntu 24.04 tested), ~150 GB free.
- i.MX8MP EVK with M.2 SDIO slot, Sterling LWB5+ installed, serial console, SD card.

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

## B — Create the Project + Fetch BSP Sources

```bash
mkdir -p ~/Projects/yocto_projects/lwb5p-imx8mp-v14
cd ~/Projects/yocto_projects/lwb5p-imx8mp-v14
repo init -u https://github.com/nxp-imx/imx-manifest -b imx-6.12.49-2.2.0 -m imx-6.12.49-2.2.0.xml
repo sync -j$(nproc)
```

This creates a `sources/` tree (poky, meta-imx, etc.).

## C — Add the Summit Fork

The forked layer already contains: custom DT, U-Boot default FDT,
`CONFIG_CFG80211=m`, and the `IMAGE_BOOT_FILES` boot-partition fix.

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp-v14
cd sources
git clone -b lrd-14.8.0.x https://github.com/vascode2/meta-summit-radio.git
cd ..
```

## D — Create the Build Directory

```bash
DISTRO=fsl-imx-wayland MACHINE=imx8mpevk source ./imx-setup-release.sh -b build-imx8mpevk-wal-lwbp
cd ~/Projects/yocto_projects/lwb5p-imx8mp-v14
```

## E — Configure the Build

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp-v14
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

## F — Build

```bash
cd ~/Projects/yocto_projects/lwb5p-imx8mp-v14
source sources/base/setup-environment build-imx8mpevk-wal-lwbp
bitbake lwb5p-lrd14
```

Output:
```
build-imx8mpevk-wal-lwbp/tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
```

## G — Flash the SD Card

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

## H — Verify on the EVK

Boot from the SD card, log in as root:

```bash
iw dev            # expect wlan0
hciconfig -a      # expect hci0 UP RUNNING
dmesg | grep -iE 'brcm|hci_uart|hci0|brcmfmac'
```

Expected log markers: `brcmfmac ... BCM4373/0`, `hci_uart_bcm`, `Bluetooth: hci0: BCM4373A0`.

## What the Fork Changes (Reference)

The `vascode2/meta-summit-radio` fork adds three commits on top of upstream `lrd-14.8.0.x`:

| File | Change |
|------|--------|
| `recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts` | custom DT: enable M.2 SDIO (WLAN) + Broadcom BT on uart1 |
| `recipes-kernel/linux/linux-imx_%.bbappend` | build the dtb; patch `imx_v8_defconfig` (`CONFIG_CFG80211=m`, BT/WLAN/MAC80211 off) |
| `recipes-bsp/u-boot/u-boot-imx_%.bbappend` + `files/imx8mpevk-lwb5plus.cfg` | `CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"` |
| `recipes-packages/images/lwb5p-lrd14.bb` | `IMAGE_BOOT_FILES:append = " imx8mp-evk-lwb5plus.dtb"` (put dtb on boot partition) |

> `CONFIG_CFG80211` **must be `=m`** (module). Built-in `=y` trips backports
> `checks.h` `#error`; `is not set` compiles out `net_device.ieee80211_ptr` and
> breaks the WLAN build (and aborts before BT modules are packaged → no `hci0`).
