# ImmortalWrt Surface 3 专用固件

基于 ImmortalWrt 24.10.6 自编译的 Surface 3 专用固件，预装全部硬件驱动和 OpenClash。

## 硬件支持

| 硬件 | 状态 | 说明 |
|------|------|------|
| 屏幕背光 | ✅ | 开机 30 秒自动息屏 |
| GPU | ✅ | Intel i915 |
| USB 网卡 | ✅ | CDC Ether / AX88179 / RTL8152 |
| WiFi | ✅ STA | Marvell 88W8897 PCIe，客户端模式 |
| 蓝牙 | ✅ | BT 4.2 via USB |
| 电池 | ✅ | BQ27xxx |
| OpenClash | ✅ | mihomo compatible 内核 + Yacd-meta 面板 |
| 触摸屏 | ❌ | 待加 Silead 驱动 |
| WiFi AP | ❌ | TXPWR_CFG 错误，仅 STA 可用 |

## 固件下载

| 文件 | 用途 |
|------|------|
| `immortalwrt-x86-64-generic-ext4-combined-efi.img.gz` | **推荐**，EFI 启动，dd 写入 |
| `immortalwrt-x86-64-generic-squashfs-combined-efi.img.gz` | SquashFS 版本 |

## 快速部署

### 1. 制作 U 盘（Mac）

```bash
gunzip -f immortalwrt-x86-64-generic-ext4-combined-efi.img.gz
diskutil eraseDisk FAT32 USB /dev/diskN
diskutil unmountDisk force /dev/diskN
sudo dd if=immortalwrt-x86-64-generic-ext4-combined-efi.img of=/dev/diskN bs=1m
```

### 2. Surface 3 UEFI 设置

- 关机 → 按住音量+ → 按电源键 → 进入 UEFI
- 关闭 Secure Boot
- U 盘启动优先

### 3. 写入 eMMC（U 盘启动后）

```bash
fdisk -l  # 确认 eMMC 设备名（58G）
umount -l /dev/mmcblk1p* 2>/dev/null
dd if=/dev/zero of=/dev/mmcblk1 bs=1M count=1
dd if=/dev/sda of=/dev/mmcblk1 bs=1M count=1100
sync && poweroff
```

拔 U 盘，开机。

### 4. 首次配置

管理地址：`http://192.168.11.251`

**修复网关（如果无法上网）：**

```bash
uci set network.lan.gateway='192.168.11.1'
uci set network.lan.dns='192.168.11.1'
uci set network.lan.defaultroute='1'
uci commit network
/etc/init.d/network restart
```

**opkg 源：**

```bash
echo 'src/gz immortalwrt_core https://downloads.immortalwrt.org/releases/24.10.6/targets/x86/64/packages
src/gz immortalwrt_base https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/base
src/gz immortalwrt_luci https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/luci
src/gz immortalwrt_packages https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/packages
src/gz immortalwrt_routing https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/routing
src/gz immortalwrt_telephony https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/telephony' > /etc/opkg/distfeeds.conf
opkg update
```

**Argon 主题：**

```bash
opkg install luci-theme-argon luci-app-argon-config
```

### 5. WiFi 客户端模式

```bash
opkg install --force-reinstall wpad-basic-mbedtls
uci set wireless.radio0.channel='auto'
uci set wireless.default_radio0.mode='sta'
uci set wireless.default_radio0.ssid='你的WiFi名'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='你的WiFi密码'
uci set wireless.default_radio0.network='wan'
uci commit wireless
reboot
```

### 6. 蓝牙

```bash
opkg install bluez-daemon bluez-utils
bluetoothctl
# power on → agent on → scan on → pair/trust/connect
```

## 已知问题

| 问题 | 解决 |
|------|------|
| 刷入后无法上网 | 执行网关修复命令（见上方） |
| WiFi AP 模式不可用 | TXPWR_CFG 错误，仅 STA 模式 |
| Argon 主题未预装 | `opkg install luci-theme-argon` |
| opkg 装 kmod 包失败 | 自编译内核与官方源不兼容，kmod 已预装 |
| wpad 路径不匹配 | `opkg install --force-reinstall wpad-basic-mbedtls` |

## 编译指南

完整编译步骤见 [BUILD.md](BUILD.md)。

## 默认配置

- IP: `192.168.11.251`
- 网关/DNS: `192.168.11.1`
- OpenClash 内核: mihomo compatible (支持 Atom CPU)
- 面板: Yacd-meta
- 开机 30 秒自动息屏
