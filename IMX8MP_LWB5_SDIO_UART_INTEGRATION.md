# Sterling LWB5+ (SDIO/UART M.2) Integration on NXP i.MX8M Plus EVK — Yocto Bring-up Guide

Application note for bringing up **WiFi (WLAN) over SDIO** and **Bluetooth over UART**
on the NXP **i.MX8M Plus EVK** using an **Ezurio/Summit Sterling LWB5+ (BCM4373/CYW4373A0)**
module fitted in the M.2 slot — built with **Yocto/Poky**.

This is an update to the Ezurio application note
[“Sterling LWB5+ USB Dongle – NXP i.MX8M EVK Integration Guide”][appnote-usb],
but for the **M.2 SDIO/UART** module variant instead of the USB dongle.

---

## 1. Scope & Hardware

| Item | Value |
|------|-------|
| SoC / Board | NXP **i.MX8M Plus** EVK (`imx8mpevk`) |
| Radio | Ezurio/Summit **Sterling LWB5+**, **CYW4373A0 (BCM4373)** chipset |
| Form factor | **M.2** module in the EVK's M.2 SDIO-capable slot |
| WLAN interface | **SDIO** (`usdhc1`, mmc@30b40000) |
| Bluetooth interface | **UART** (`uart1`) |

> The M.2 slot on the i.MX8MP EVK can route **SDIO** (not just PCIe). NXP ships a
> dedicated device tree for this: `imx8mp-evk-usdhc1-m2.dts`.

## 2. Software Stack

| Component | Version |
|-----------|---------|
| Yocto / Poky | 5.2 ("walnascar", DISTRO_VERSION 5.2.2) |
| DISTRO | `fsl-imx-wayland` |
| Machine | `imx8mpevk` |
| Kernel | `linux-imx` 6.12.34 (NXP BSP) |
| U-Boot | `u-boot-imx` 2025.04 |
| Summit Connectivity Stack | **LRD-REL-14.8.0.6** (`RADIO_VERSION = "14.8.0.6"`) |

Summit layer used: `meta-summit-radio`.

---

## 3. Reference Documents

- Ezurio USB-dongle app note (original): [Sterling LWB5+ USB Dongle – i.MX8M EVK][appnote-usb]
- Ezurio `meta-summit-radio` (branch `lrd-14.8.0.x`): <https://github.com/Ezurio/meta-summit-radio>
- NXP i.MX8M Plus EVK device tree sources (in `linux-imx`):
  - `arch/arm64/boot/dts/freescale/imx8mp-evk.dts`
  - `arch/arm64/boot/dts/freescale/imx8mp-evk-usdhc1-m2.dts` (SDIO/PCIe M.2 slot)

---

## 4. Architecture Overview

```
                    i.MX8M Plus EVK
  ┌───────────────────────────────────────────────┐
  │                                               │
  │  M.2 slot  ── SDIO ──► usdhc1 (mmc@30b40000) │  WLAN (wlan0)
  │      └────── UART ──► uart1                   │  BT (hci0)
  │                                               │
  │  SD card   ── SDIO ──► usdhc2 (mmc@30b50000) │  boot device (mmcblk1)
  │  eMMC      ── SDIO ──► usdhc3 (mmc@30b60000) │  non-removable
  │                                               │
  └───────────────────────────────────────────────┘
```

Stock `imx8mp-evk.dts` prints:
- `usdhc1` (M.2 SDIO slot): **disabled**, no wireless child node
- `uart1`: enabled for BT, but with `compatible = "nxp,88w8997-bt"` (the *wrong* chip —
  NXP/Marvell 88W8997, not Broadcom CYW4373A0)

Both are fixed by a **custom device tree** (see §6).

---

## 5. Why the Radio Did Not Appear Out of the Box

When first booting the stock image with `fdtfile=imx8mp-evk.dtb`:

- `iw dev` → empty (**no wlan0**)
- `hciconfig -a` → empty (**no hci0**)
- `lsusb` → only Linux root hubs (USB path not used for M.2)

**Root causes (both device-tree, not firmware):**

1. **WLAN** – The M.2 SDIO slot (`usdhc1`) is `status = "disabled"` in the stock dtb, so
   the mmc/SDIO interface for the radio is never brought up.
2. **Bluetooth** – `uart1` is declared for BT but with the `nxp,88w8997-bt` compatible
   string. The Broadcom `hci_uart`/`btbcm` driver cannot bind to it, so no `hci0`.

The image already contained the correct **SDIO firmware** (`lwb5plus-sdio-div-firmware`)
and the Summit **backports** wifi/bt drivers — they just had no matching DT node.

---

## 6. Custom Device Tree

Create a device tree that **includes** NXP's SDIO M.2 config and **fixes** the BT node.

`recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts`:

```dts
/dts-v1/;
#include "imx8mp-evk-usdhc1-m2.dts"

/* LWB5+ Bluetooth on uart1: use Broadcom HCI, not NXP 88W8997 */
&uart1 {
    bluetooth {
        compatible = "cypress,cyw4373a0-bt";
    };
};
```

- `imx8mp-evk-usdhc1-m2.dts` (from NXP) does the heavy lifting for WLAN:
  - enables `usdhc1` for SDIO
  - adds `reg_usdhc1_vmmc` (WLAN_EN, GPIO2_IO06) and `usdhc1_pwrseq` (reset GPIO2_IO10)
  - adds `wifi_wake_host`, `mmc-pwrseq-simple`
  - disables `reg_pcie0`, `pcie`, `pcie_phy` (SDIO vs PCIe mux)
- `cypress,cyw4373a0-bt` is the exact compatible string the Summit backported
  `hci_bcm.c` binds to for the LWB5+ (with `.no_uart_clock_set = true`).

This single dtb enables **both WiFi (SDIO) and Bluetooth (UART)**.

---

## 7. Yocto Layer Changes (meta-summit-radio)

### 7.1 Kernel bbappend
`recipes-kernel/linux/linux-imx_%.bbappend`

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

LWB_DTS_FILES = "dts/imx8mp-evk-lwb5plus.dts"
SRC_URI += "file://dts/imx8mp-evk-lwb5plus.dts"

KERNEL_DEVICETREE:append:imx8mpevk = " freescale/imx8mp-evk-lwb5plus.dtb"

do_patch:append() {
    # 1) install custom dts
    if [ -d "${S}/arch/arm64/boot/dts/freescale" ]; then
        install -m 0644 "${UNPACKDIR}/dts/imx8mp-evk-lwb5plus.dts" \
            "${S}/arch/arm64/boot/dts/freescale/"
    fi
    # 2) disable in-kernel cfg80211/mac80211/BT in the defconfig (see §8)
    DEF="${S}/arch/arm64/configs/imx_v8_defconfig"
    if [ -f "$DEF" ]; then
        sed -i \
            -e '/^CONFIG_CFG80211=y$/c\# CONFIG_CFG80211 is not set' \
            -e '/^CONFIG_MAC80211=y$/c\# CONFIG_MAC80211 is not set' \
            "$DEF"
        grep -q '^# CONFIG_BT is not set$' "$DEF" \
            || printf '# CONFIG_BT is not set\n' >> "$DEF"
    fi
}
```

### 7.2 U-Boot bbappend
`recipes-bsp/u-boot/u-boot-imx_%.bbappend`
```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://imx8mpevk-lwb5plus.cfg"
```

`recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg`:
```
CONFIG_DEFAULT_FDT_FILE="imx8mp-evk-lwb5plus.dtb"
```

This makes U-Boot load the custom dtb **by default** so no manual `setenv` is needed
at boot.

---

## 8. Kernel Config: cfg80211 / mac80211 / Bluetooth

The Summit **backports** provide their own cfg80211 / mac80211 / Bluetooth modules.
Its `checks.h` refuses to build when these are present **in the kernel**:

```c
#if defined(CONFIG_CFG80211) && defined(CPTCFG_CFG80211_MODULE)
#error "You must not have cfg80211 built into your kernel if you want to enable it"
#endif
```

> `CONFIG_CFG80211=m` does **not** help — `defined(CONFIG_CFG80211)` is true even for a
> module, so the `#error` still fires. The kernel must have these **completely absent**.

### Pitfall — why a normal config fragment doesn't work
`linux-imx` runs `do_copy_defconfig` **after** `do_kernel_configme`, which **overwrites
`.config`** with the stock `imx_v8_defconfig`, wiping any fragment applied by configme.
Also, `DELTA_KERNEL_DEFCONFIG`'s `merge_config.sh` **cannot turn `=y` off** — it only
appends duplicate `# … is not set` lines, which are ignored by kconfig.

**Therefore the disables must be written into `imx_v8_defconfig` itself**
(via the `do_patch:append` in §7.1), so `do_copy_defconfig` copies them in.

### Consequence of disabling cfg80211
`struct net_device.ieee80211_ptr` in the kernel is guarded by
`#if IS_ENABLED(CONFIG_CFG80211)`. Disabling cfg80211 **compiles that member out** —
which the backports' own cfg80211 code requires. This is the core tension (see
§11 “Open Issues”).

---

## 9. Build Commands

```bash
cd <project-root>
# machine/distro are baked in via conf/local.conf / bblayers.conf
bitbake lwb5p-lrd14          # custom image recipe
```

Image outputs (deploy dir):
```
tmp/deploy/images/imx8mpevk/lwb5p-lrd14-imx8mpevk.rootfs.wic.gz
```

### 9.1 On-target (manual fallback) — verify the DT fix
At the U-Boot prompt you can load the SDIO M.2 dtb manually to prove the fix:

```
setenv fdtfile imx8mp-evk-usdhc1-m2.dtb
saveenv
boot
```

(Add the same file to the FAT boot partition if you only want to test and not bake.)

---

## 10. Verification on Target

```bash
# WiFi (SDIO)
iw dev                  # expect: phy0 / wlan0
ip link set wlan0 up
iw dev wlan0 scan | head

# Bluetooth (UART)
hciconfig -a            # expect: hci0 UP RUNNING
bluetoothctl            # interactive pairing/scan
dmesg | grep -iE 'brcm|hci_uart|hci0'
```

Kernel log should show the Broadcom HCI binding and loading of the LWB5+ HCD firmware,
e.g. `BCM4373A0-sdio*…hcd`.

---

## 11. Open Issues / Notes

- **Compiler API mismatch (kernel 6.12 vs backports 14.8.0.6).**
  Summit backports 14.8.0.6 cfg80211 references `net_device.ieee80211_ptr`, which
  kernel 6.12 compiles out when cfg80211 is disabled. The backports must have cfg80211
  *absent* (for `checks.h`) but *present* (for the struct member) — a fundamental
  conflict on this kernel. The `lrd-14.8.0.x` branch still pins 14.8.0.6 and does not
  fix it. **Needed:** a backports build validated on 6.12 (check with Ezurio), an older
  kernel BSP, or a backports compat patch.

- **EEPROM/reg files** are installed via the firmware packages
  (`regLWB5plus-aarch64` etc.) and picked up at runtime.

- **M.2 vs USB preference.** This guide uses the M.2 **SDIO/UART** module, not the USB
  dongle. (The USB path can additionally suffer from power/enumeration issues on
  `usb@38100000`.)

---

## 12. Related Files (in this BSP)

| File | Purpose |
|------|---------|
| `recipes-kernel/linux/files/dts/imx8mp-evk-lwb5plus.dts` | custom DT (SDIO wifi + BT) |
| `recipes-kernel/linux/linux-imx_%.bbappend` | build dtb, patch defconfig |
| `recipes-bsp/u-boot/u-boot-imx_%.bbappend` | boot custom dtb by default |
| `recipes-bsp/u-boot/files/imx8mpevk-lwb5plus.cfg` | `CONFIG_DEFAULT_FDT_FILE` |
| `recipes-packages/images/lwb5p-lrd14.bb` | custom image recipe (SDIO variant) |

---

[appnote-usb]: https://www.ezurio.com/documentation/application-note-sterling-lwb5-usb-dongle-nxp-i-m8x-evk-integration-guide
