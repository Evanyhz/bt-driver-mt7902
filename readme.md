# Ubuntu 22.04 / Kernel 6.8 下启用 MT7902 Wi-Fi 与蓝牙

本文记录在 Ubuntu 22.04、Kernel `6.8.0-117-generic` 下，修复 MediaTek MT7902 的 Wi-Fi 与蓝牙的过程。
MT7902 的 Wi-Fi 和蓝牙是两条独立链路：

```text
Wi-Fi      PCIe: 14c3:7902
Bluetooth USB : 13d3:3596
```

---

## 环境

```text
系统：Ubuntu 22.04
内核：6.8.0-117-generic
主板：ASUS B850 AYW GAMING WIFI D5
无线网卡：MediaTek MT7902 PCIe
Wi-Fi PCI ID：14c3:7902
Subsystem：AzureWave 1a3b:6040
蓝牙 USB ID：13d3:3596
```

---

## Wi-Fi 修复说明

Wi-Fi 使用 `hmtheboy154/mt7902` 仓库中的 `mt7902e` 驱动。

最终状态：

```text
驱动模块：mt7902e
无线接口：wlp8s0
状态：可正常连接和联网
```

关键问题：

```text
failed to insert STA entry for the AP (error -22)
```

关键修复点：

```text
src/mac80211.c
mt76_vif_phy()
```

当 `mlink->ctx == NULL` 时，不应直接返回 `NULL`，而应返回默认 PHY。
否则连接 AP 时会因为 PHY 获取失败导致 STA entry 插入失败。

蓝牙修复过程中不要重新折腾 Wi-Fi 驱动，不要卸载 `mt7902e`。

---

## 蓝牙初始问题

蓝牙服务本身正常运行：

```text
bluetooth.service active running
```

但没有可用控制器：

```bash
bluetoothctl list
```

无输出。

`btmgmt` 显示：

```text
Index list with 0 items
```

内核日志中出现：

```text
Bluetooth: hci0: Opcode 0x0c03 failed: -110
```

这说明问题发生在蓝牙控制器初始化阶段，不是配对问题。

---

## 确认蓝牙 USB 设备

查看蓝牙 USB 设备：

```bash
lsusb -d 13d3:3596
```

输出示例：

```text
Bus 007 Device 002: ID 13d3:3596 IMC Networks Wireless_Device
Manufacturer: MediaTek Inc.
```

说明系统已经枚举到 MT7902 的蓝牙 USB 设备。

查看 USB 拓扑：

```bash
lsusb -t
```

可见类似：

```text
Bus 07.Port 1: Dev 2, If 0, Class=Wireless, Driver=btusb
Bus 07.Port 1: Dev 2, If 1, Class=Wireless, Driver=btusb
```

说明 `btusb` 已经绑定设备，但原版 Ubuntu 6.8 的 `btusb/btmtk` 对 MT7902 支持不完整。

---

## 原版 btusb / btmtk 的问题

检查原版 `btusb`：

```bash
modinfo btusb | grep -Ei '13d3.*(3579|3580|3594|3596)|0e8d.*7902|7902'
```

没有 MT7902 明确信息。

检查原版 `btmtk`：

```bash
modinfo btmtk | grep -Ei '7902|BT_RAM_CODE_MT7902|firmware'
```

原版没有：

```text
mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin
```

因此仅靠 Ubuntu 22.04 HWE 6.8 自带模块无法正常初始化 MT7902 蓝牙。

---

## 蓝牙固件

MT7902 蓝牙需要以下固件：

```text
/lib/firmware/mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin
```

安装固件：

```bash
wget -O BT_RAM_CODE_MT7902_1_1_hdr.bin \
"https://gitlab.com/kernel-firmware/linux-firmware/-/raw/main/mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin"

sudo install -D -m 0644 BT_RAM_CODE_MT7902_1_1_hdr.bin \
/lib/firmware/mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin
```

检查：

```bash
ls -l /lib/firmware/mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin
```

注意：
该 `.bin` 是固件文件，不建议提交到本仓库。README 中说明下载方式即可。

---

## 蓝牙 backport

使用蓝牙专用 backport：

```bash
git clone https://github.com/bupd/bt-driver-mt7902.git
cd bt-driver-mt7902
```

只处理：

```text
btusb.ko
btmtk.ko
```

不处理：

```text
mt7902e
mt76
mac80211
cfg80211
Wi-Fi firmware
```

---

## Ubuntu 6.8 编译兼容修改

该 backport 源码来自较新的内核，直接在 Ubuntu 6.8 上编译会报错，需要做兼容修改。

### 1. 修改 unaligned 头文件

将：

```c
#include <linux/unaligned.h>
```

改为：

```c
#include <asm/unaligned.h>
```

可用命令：

```bash
sed -i 's|#include <linux/unaligned.h>|#include <asm/unaligned.h>|' btusb.c btmtk.c
```

### 2. 删除 hci_drv.h

Ubuntu 6.8 没有：

```c
#include <net/bluetooth/hci_drv.h>
```

删除：

```bash
sed -i '/#include <net\/bluetooth\/hci_drv.h>/d' btusb.c btmtk.c
```

### 3. 删除新内核 hci_drv 代码块

Ubuntu 6.8 不支持较新内核中的 `hci_drv` driver-specific command 机制。
需要从 `btusb.c` 中删除相关代码，包括：

```text
BTUSB_HCI_DRV_OP_SUPPORTED_ALTSETTINGS
HCI_DRV_OP_READ_INFO
hci_drv_cmd_complete
hci_drv_cmd_status
struct hci_drv_handler
struct hci_drv
hdev->hci_drv = &btusb_hci_drv
```

这部分不是 MT7902 蓝牙初始化所必需。

### 4. 添加 hci quirk 兼容宏

在 `btusb.c` 的蓝牙头文件 include 后添加：

```c
/* Compatibility for kernels without hci_set_quirk()/hci_clear_quirk(). */
#ifndef hci_set_quirk
#define hci_set_quirk(hdev, quirk) set_bit((quirk), &(hdev)->quirks)
#endif

#ifndef hci_clear_quirk
#define hci_clear_quirk(hdev, quirk) clear_bit((quirk), &(hdev)->quirks)
#endif
```

注意必须使用：

```c
&(hdev)->quirks
```

不能写成：

```c
(hdev)->quirks
```

否则会出现 `set_bit/clear_bit` 指针类型警告，运行时存在风险。

---

## MT7902 蓝牙 USB ID

确认 `btusb.c` 中包含以下内容：

```c
/* MediaTek MT7902 Bluetooth devices */
{ USB_DEVICE(0x13d3, 0x3579), .driver_info = BTUSB_MEDIATEK |
                                             BTUSB_WIDEBAND_SPEECH },
{ USB_DEVICE(0x13d3, 0x3580), .driver_info = BTUSB_MEDIATEK |
                                             BTUSB_WIDEBAND_SPEECH },
{ USB_DEVICE(0x13d3, 0x3594), .driver_info = BTUSB_MEDIATEK |
                                             BTUSB_WIDEBAND_SPEECH },
{ USB_DEVICE(0x13d3, 0x3596), .driver_info = BTUSB_MEDIATEK |
                                             BTUSB_WIDEBAND_SPEECH },
```

本机实际使用的是：

```text
13d3:3596
```

---

## 编译

安装依赖：

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) git
```

编译：

```bash
make clean
make -j"$(nproc)"
```

成功后应生成：

```text
btusb.ko
btmtk.ko
```

检查 `btmtk` 是否包含 MT7902 固件声明：

```bash
modinfo -F firmware ./btmtk.ko | grep -Ei '7902|BT_RAM_CODE_MT7902'
```

期望输出：

```text
mediatek/BT_RAM_CODE_MT7902_1_1_hdr.bin
```

---

## 安装

将模块安装到 `updates` 目录，避免覆盖系统原模块：

```bash
KVER="$(uname -r)"
UPDDIR="/lib/modules/$KVER/updates/mt7902-bt"

sudo mkdir -p "$UPDDIR"
sudo install -m 0644 btusb.ko "$UPDDIR/btusb.ko"
sudo install -m 0644 btmtk.ko "$UPDDIR/btmtk.ko"

sudo depmod -a "$KVER"
```

确认模块路径：

```bash
modinfo -n btusb
modinfo -n btmtk
```

期望类似：

```text
/lib/modules/6.8.0-117-generic/updates/mt7902-bt/btusb.ko
/lib/modules/6.8.0-117-generic/updates/mt7902-bt/btmtk.ko
```

重启：

```bash
sudo reboot
```

---

## 验证蓝牙

```bash
bluetoothctl list
bluetoothctl show
btmgmt info
```

修复前常见错误：

```text
No default controller available
Index list with 0 items
Bluetooth: hci0: Opcode 0x0c03 failed: -110
```

修复后应能看到蓝牙 Controller。

---

## 验证 Wi-Fi

```bash
nmcli device status | grep -E 'DEVICE|wlp8s0'
lspci -nnk -s 08:00.0 | sed -n '1,12p'
```

期望：

```text
wlp8s0 connected
Kernel driver in use: mt7902e
```

