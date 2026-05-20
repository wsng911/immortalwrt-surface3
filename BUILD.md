# ImmortalWrt Surface 3 编译部署方案（最终版）

## 硬件状态

| 硬件 | 状态 | 说明 |
|------|------|------|
| 屏幕背光 | ✅ | PWM + PMIC + LPSS |
| GPU | ✅ | i915 |
| USB 网卡 | ✅ | CDC Ether / AX88179 / RTL8152 |
| OpenClash | ✅ | 全部依赖已编译 |
| 电池 | ✅ | BQ27xxx |
| WiFi | ✅ STA | mwifiex_pcie，客户端模式可用 |
| 蓝牙 | ✅ | btusb（USB 接口） |
| 触摸屏 | ❌ | 待加驱动，暂搁置 |

---

# 一、编译（GCP Debian 12，全新部署）

## 环境要求

| 条件 | 要求 |
|------|------|
| 磁盘 | 50GB+ |
| 内存 | 8GB+ |
| 用户 | 非 root（用户 i） |
| CMake | 3.31+（luci feed 需要） |

## 1.1 安装依赖（root）

```bash
apt update
apt install -y \
  build-essential clang flex bison g++ gawk \
  gettext git libncurses-dev libssl-dev python3-setuptools \
  rsync unzip zlib1g-dev file wget python3 \
  libelf-dev libpam-dev libcap-dev qemu-utils genisoimage
```

```bash
cd /tmp
wget https://github.com/Kitware/CMake/releases/download/v3.31.6/cmake-3.31.6-linux-x86_64.tar.gz
tar xzf cmake-3.31.6-linux-x86_64.tar.gz
cp cmake-3.31.6-linux-x86_64/bin/cmake /usr/local/bin/
cp -r cmake-3.31.6-linux-x86_64/share/cmake-3.31 /usr/local/share/
```

## 1.2 创建用户（root）

```bash
useradd -m -s /bin/bash i
su - i
```

## 1.3 克隆源码（用户 i）

```bash
cd ~
git clone https://github.com/immortalwrt/immortalwrt.git
cd immortalwrt
git checkout v24.10.6
```

## 1.4 Feeds（用户 i）

```bash
sed -i 's|src-git luci.*|src-git luci https://github.com/immortalwrt/luci.git|' feeds.conf.default
echo "src-git openclash https://github.com/vernesong/OpenClash.git;master" >> feeds.conf.default
```

```bash
./scripts/feeds update -a -f
./scripts/feeds install -a 2>/dev/null || true
cp -r feeds/openclash/luci-app-openclash package/luci-app-openclash 2>/dev/null || true
```

## 1.5 包选择（用户 i）

```bash
cat > .config << 'EOF'
CONFIG_TARGET_x86=y
CONFIG_TARGET_x86_64=y
CONFIG_TARGET_x86_64_DEVICE_generic=y
CONFIG_TARGET_ROOTFS_EXT4FS=y
CONFIG_TARGET_ROOTFS_PARTSIZE=1024
CONFIG_TARGET_IMAGES_GZIP=y
CONFIG_GRUB_EFI_IMAGES=y
# CONFIG_PACKAGE_dnsmasq is not set
CONFIG_PACKAGE_base-files=y
CONFIG_PACKAGE_block-mount=y
CONFIG_PACKAGE_ca-bundle=y
CONFIG_PACKAGE_default-settings-chn=y
CONFIG_PACKAGE_dnsmasq-full=y
CONFIG_PACKAGE_dropbear=y
CONFIG_PACKAGE_firewall4=y
CONFIG_PACKAGE_fstools=y
CONFIG_PACKAGE_nftables=y
CONFIG_PACKAGE_netifd=y
CONFIG_PACKAGE_opkg=y
CONFIG_PACKAGE_ppp=y
CONFIG_PACKAGE_ppp-mod-pppoe=y
CONFIG_PACKAGE_uci=y
CONFIG_PACKAGE_uclient-fetch=y
CONFIG_PACKAGE_urandom-seed=y
CONFIG_PACKAGE_fdisk=y
CONFIG_PACKAGE_partx-utils=y
CONFIG_PACKAGE_odhcp6c=y
CONFIG_PACKAGE_odhcpd-ipv6only=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-base=y
CONFIG_PACKAGE_luci-compat=y
CONFIG_PACKAGE_luci-lib-base=y
CONFIG_PACKAGE_luci-lib-ipkg=y
CONFIG_PACKAGE_luci-mod-admin-full=y
CONFIG_PACKAGE_luci-mod-network=y
CONFIG_PACKAGE_luci-mod-status=y
CONFIG_PACKAGE_luci-mod-system=y
CONFIG_PACKAGE_luci-app-package-manager=y
CONFIG_PACKAGE_luci-i18n-base-zh-cn=y
# CONFIG_PACKAGE_luci-theme-argon is not set
# CONFIG_PACKAGE_luci-app-argon-config is not set
CONFIG_PACKAGE_luci-app-openclash=y
CONFIG_PACKAGE_kmod-inet-diag=y
CONFIG_PACKAGE_kmod-nft-tproxy=y
CONFIG_PACKAGE_kmod-tun=y
CONFIG_PACKAGE_kmod-ipt-tproxy=y
CONFIG_PACKAGE_iptables-mod-tproxy=y
CONFIG_PACKAGE_kmod-nf-nathelper=y
CONFIG_PACKAGE_kmod-nf-nathelper-extra=y
CONFIG_PACKAGE_iptables=y
CONFIG_PACKAGE_iptables-mod-extra=y
CONFIG_PACKAGE_iptables-nft=y
CONFIG_PACKAGE_kmod-nft-core=y
CONFIG_PACKAGE_kmod-nft-nat=y
CONFIG_PACKAGE_kmod-nft-socket=y
CONFIG_PACKAGE_ruby=y
CONFIG_PACKAGE_ruby-yaml=y
CONFIG_PACKAGE_ipset=y
CONFIG_PACKAGE_coreutils-nohup=y
CONFIG_PACKAGE_kmod-drm-i915=y
CONFIG_PACKAGE_i915-firmware=y
CONFIG_PACKAGE_i915-firmware-dmc=y
CONFIG_PACKAGE_kmod-mwifiex-pcie=y
CONFIG_PACKAGE_mwifiex-pcie-firmware=y
CONFIG_PACKAGE_kmod-bluetooth=y
CONFIG_PACKAGE_kmod-btusb=y
CONFIG_PACKAGE_kmod-hid-generic=y
CONFIG_PACKAGE_kmod-usb-net=y
CONFIG_PACKAGE_kmod-usb-net-cdc-ether=y
CONFIG_PACKAGE_kmod-usb-net-asix-ax88179=y
CONFIG_PACKAGE_kmod-usb-net-rtl8152=y
# CONFIG_PACKAGE_kmod-usb-net-rtl8152-vendor is not set
CONFIG_PACKAGE_kmod-fs-f2fs=y
CONFIG_PACKAGE_kmod-fs-vfat=y
CONFIG_PACKAGE_wpad-basic-mbedtls=y
CONFIG_PACKAGE_bluez-daemon=y
CONFIG_PACKAGE_bluez-utils=y
CONFIG_PACKAGE_bash=y
CONFIG_PACKAGE_curl=y
CONFIG_PACKAGE_ip-full=y
CONFIG_PACKAGE_unzip=y
CONFIG_PACKAGE_wget-ssl=y
CONFIG_PACKAGE_nano=y
EOF
```

## 1.6 内核选项（用户 i）

```bash
cat >> target/linux/x86/config-6.6 << 'EOF'
CONFIG_PWM=y
CONFIG_PWM_SYSFS=y
CONFIG_PWM_CRC=y
CONFIG_PWM_LPSS=y
CONFIG_PWM_LPSS_PCI=y
CONFIG_PWM_LPSS_PLATFORM=y
CONFIG_I2C_DESIGNWARE_CORE=y
CONFIG_I2C_DESIGNWARE_PLATFORM=y
CONFIG_I2C_DESIGNWARE_PCI=y
CONFIG_MFD_INTEL_LPSS=y
CONFIG_MFD_INTEL_LPSS_ACPI=y
CONFIG_MFD_INTEL_LPSS_PCI=y
CONFIG_X86_INTEL_LPSS=y
CONFIG_INTEL_SOC_PMIC=y
CONFIG_INTEL_SOC_PMIC_CHTWC=y
CONFIG_MFD_INTEL_PMC_BXT=y
CONFIG_X86_PLATFORM_DEVICES=y
CONFIG_REGULATOR=y
CONFIG_POWER_SUPPLY=y
CONFIG_POWER_SUPPLY_HWMON=y
CONFIG_PM=y
CONFIG_PM_SLEEP=y
CONFIG_PM_GENERIC_DOMAINS=y
CONFIG_MMC_SDHCI_ACPI=y
CONFIG_PINCTRL_CHERRYVIEW=y
CONFIG_GPIO_CRYSTAL_COVE=y
CONFIG_BATTERY_BQ27XXX=y
CONFIG_BATTERY_BQ27XXX_I2C=y
CONFIG_I2C_CHT_WC=y
CONFIG_BATTERY_BQ27XXX_DT_UPDATES_NVM=n
EOF
```

## 1.7 预置文件（用户 i）

```bash
# WiFi 固件
mkdir -p files/lib/firmware/mrvl
wget -O files/lib/firmware/mrvl/sd8897_uapsta.bin \
  https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/mrvl/sd8897_uapsta.bin
```

```bash
# 默认网络配置
mkdir -p files/etc/config
cat > files/etc/config/network << 'EOF'
config interface 'loopback'
    option device 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config interface 'lan'
    option device 'br-lan'
    option proto 'static'
    option ipaddr '192.168.11.251'
    option netmask '255.255.255.0'
    option gateway '192.168.11.1'
    option dns '192.168.11.1'
    option defaultroute '1'

config device
    option name 'br-lan'
    option type 'bridge'
    list ports 'eth0'
EOF
```

```bash
# 开机 30 秒自动息屏
mkdir -p files/etc/init.d files/etc/rc.d
cat > files/etc/init.d/screen_off << 'EOF'
#!/bin/sh /etc/rc.common
START=99

start() {
    sleep 30
    echo 0 > /sys/class/backlight/intel_backlight/brightness 2>/dev/null
}
EOF
chmod +x files/etc/init.d/screen_off
ln -sf ../init.d/screen_off files/etc/rc.d/S99screen_off
```

```bash
# opkg 使用官方源（刷入后可手动切换清华源）
mkdir -p files/etc/opkg
cat > files/etc/opkg/distfeeds.conf << 'EOF'
src/gz immortalwrt_core https://downloads.immortalwrt.org/releases/24.10.6/targets/x86/64/packages
src/gz immortalwrt_base https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/base
src/gz immortalwrt_luci https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/luci
src/gz immortalwrt_packages https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/packages
src/gz immortalwrt_routing https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/routing
src/gz immortalwrt_telephony https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/telephony
EOF
```

## 1.8 生成配置 + CMake 修复（用户 i）

```bash
make defconfig
sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
```

```bash
cp /usr/local/bin/cmake ~/immortalwrt/staging_dir/host/bin/cmake
cp -r /usr/local/share/cmake-3.31 ~/immortalwrt/staging_dir/host/share/
```

## 1.9 编译（用户 i）

```bash
make download -j$(nproc)
make -j1 V=s 2>&1 | tee build.log
```

> **⚠️ MFD_INTEL_LPSS_ACPI 修复（每次必做）：**
> 编译后检查：
> ```bash
> grep "MFD_INTEL_LPSS_ACPI" build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
> ```
> 如果显示 `not set`，修复后重编内核：
> ```bash
> sed -i 's/.*MFD_INTEL_LPSS_ACPI.*/CONFIG_MFD_INTEL_LPSS_ACPI=y/' \
>   build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
> make target/linux/compile -j1 V=s
> sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
> make -j1 V=s 2>&1 | tee build.log
> ```

> **⚠️ 内核交互提示失败时：**
> ```bash
> echo "CONFIG_BATTERY_BQ27XXX_DT_UPDATES_NVM=n" >> \
>   build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
> echo "CONFIG_I2C_CHT_WC=y" >> \
>   build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
> sed -i 's/.*MFD_INTEL_LPSS_ACPI.*/CONFIG_MFD_INTEL_LPSS_ACPI=y/' \
>   build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
> make target/linux/compile -j1 V=s
> sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
> make -j1 V=s 2>&1 | tee build.log
> ```

## 1.10 取出固件

```bash
ls -lh bin/targets/x86/64/immortalwrt-*-ext4-combined-efi.img.gz
```

---

# 二、制作 U 盘（Mac）

```bash
cd ~/Downloads/Surface-Immorta/
scp i@GCP_IP:~/immortalwrt/bin/targets/x86/64/immortalwrt-*-ext4-combined-efi.img.gz .
```

```bash
gunzip -f immortalwrt-*-ext4-combined-efi.img.gz
```

```bash
diskutil eraseDisk FAT32 USB /dev/disk8
diskutil unmountDisk force /dev/disk8
sudo dd if=immortalwrt-*-ext4-combined-efi.img of=/dev/disk8 bs=1m
sudo dd if=immortalwrt-x86-64-generic-ext4-combined-efi.img of=/dev/disk8 bs=1m

```

---

# 三、写入硬盘（U 盘启动 Surface 后）

```bash
fdisk -l  # 确认 eMMC 设备名（58G 那个）
```

```bash
umount -l /dev/mmcblk1p* 2>/dev/null
dd if=/dev/zero of=/dev/mmcblk1 bs=1M count=1
dd if=/dev/sda of=/dev/mmcblk1 bs=1M count=1100
sync
poweroff
```

> 设备名以实际为准（mmcblk0 或 mmcblk1）。

拔 U 盘，开机。

---

# 四、刷入后配置

管理地址：`http://192.168.11.251`

## WiFi 客户端模式

```bash
opkg update
opkg install --force-reinstall wpad-basic-mbedtls
```

```bash
uci set wireless.radio0.channel='auto'
uci set wireless.default_radio0.mode='sta'
uci set wireless.default_radio0.ssid='你的WiFi名'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='你的WiFi密码'
uci set wireless.default_radio0.network='wan'
uci commit wireless
reboot
```

## 蓝牙配对

```bash
opkg install bluez-daemon bluez-utils
bluetoothctl
```

```
power on
agent on
default-agent
scan on
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
connect XX:XX:XX:XX:XX:XX
```

## Argon 主题（如果编译时因 wget-any 冲突未预装）

```bash
opkg update
opkg install luci-theme-argon luci-app-argon-config
```

## 切换 opkg 源（清华不支持 24.10.6，使用官方源）

```bash
echo 'src/gz immortalwrt_core https://downloads.immortalwrt.org/releases/24.10.6/targets/x86/64/packages
src/gz immortalwrt_base https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/base
src/gz immortalwrt_luci https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/luci
src/gz immortalwrt_packages https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/packages
src/gz immortalwrt_routing https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/routing
src/gz immortalwrt_telephony https://downloads.immortalwrt.org/releases/24.10.6/packages/x86_64/telephony' > /etc/opkg/distfeeds.conf
opkg update
```

> 注意：自编译内核的固件用官方源装 kmod 包会版本不匹配，只有用户态包能装。

---

# 五、已知问题

| 问题 | 解决 |
|------|------|
| `MFD_INTEL_LPSS_ACPI` 被 defconfig 覆盖 | 编译后 sed 修复再重编内核 |
| `make defconfig` 拉回 dnsmasq | 每次 defconfig 后 sed 删除 |
| Argon 依赖 `wget-any` 编译失败 | 刷完后 opkg 安装 |
| 内核配置交互提示失败 | 手动写入默认值 |
| WiFi AP 模式不可用 | TXPWR_CFG 错误，仅 STA 模式可用 |
| wpad 路径不匹配 | `opkg install --force-reinstall wpad-basic-mbedtls` |
| fdisk 扩容后无法启动 | 起始扇区用默认值，不要手动输入 |

---

# 编译修复记录

## I2C_CHT_WC 交互提示导致编译失败

内核配置出现 `I2C_CHT_WC [N/m/y/?] (NEW)` 交互提示，非交互模式下失败。

**用户 i 执行：**

```bash
cd ~/immortalwrt

echo "CONFIG_I2C_CHT_WC=y" >> target/linux/x86/config-6.6
```

```bash
echo "CONFIG_I2C_CHT_WC=y" >> build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
sed -i 's/.*MFD_INTEL_LPSS_ACPI.*/CONFIG_MFD_INTEL_LPSS_ACPI=y/' \
  build_dir/target-x86_64_musl/linux-x86_64/linux-6.6.133/.config
```

```bash
make target/linux/compile -j1 V=s
```

```bash
sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
make -j1 V=s 2>&1 | tee build.log
```

---

# 下一版：开箱即用版构建

在当前版本基础上，将所有需要刷入后手动安装的内容提前封装进固件。
目标：dd 写入后直接可用，无需任何 opkg 操作。

## 预置内容清单

| 内容 | 路径 | 说明 |
|------|------|------|
| mihomo 内核 | `files/etc/openclash/core/clash_meta` | OpenClash 代理内核 |
| Yacd-meta 面板 | `files/etc/openclash/ui/` | OpenClash Web 面板 |
| GeoIP 数据 | `files/etc/openclash/GeoIP.dat` | IP 分流 |
| GeoSite 数据 | `files/etc/openclash/GeoSite.dat` | 域名分流 |
| Country.mmdb | `files/etc/openclash/Country.mmdb` | IP 数据库 |
| WiFi 固件 | `files/lib/firmware/mrvl/sd8897_uapsta.bin` | mwifiex PCIe |
| i915 GPU 固件 | 编译时打包 | .config 包含 |
| mwifiex-pcie 固件 | 编译时打包 | .config 包含 |
| USB 网卡驱动 | 编译时打包 | cdc-ether/ax88179/rtl8152 |
| 蓝牙驱动+工具 | 编译时打包 | btusb/bluez-daemon/bluez-utils |
| wpad | 编译时打包 | WiFi STA 模式依赖 |

## 预置命令（用户 i，在 make 之前执行）

### OpenClash 内核（mihomo compatible 版，Surface 3 不支持 v3 指令集）

```bash
mkdir -p files/etc/openclash/core
VERSION=$(curl -s https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep tag_name | cut -d'"' -f4)
wget -O /tmp/mihomo.gz \
  "https://github.com/MetaCubeX/mihomo/releases/download/${VERSION}/mihomo-linux-amd64-compatible-${VERSION}.gz"
gunzip -f /tmp/mihomo.gz
mv /tmp/mihomo-linux-amd64-compatible-${VERSION} files/etc/openclash/core/clash_meta 2>/dev/null \
  || mv /tmp/mihomo files/etc/openclash/core/clash_meta
chmod +x files/etc/openclash/core/clash_meta
```

### GeoIP / GeoSite / Country 数据

```bash
mkdir -p files/etc/openclash
wget -O files/etc/openclash/GeoIP.dat \
  https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download/geoip.dat
wget -O files/etc/openclash/GeoSite.dat \
  https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download/geosite.dat
wget -O files/etc/openclash/Country.mmdb \
  https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download/country.mmdb
```

### OpenClash Web 面板（Yacd-meta）

```bash
mkdir -p files/etc/openclash/ui
wget -O /tmp/yacd.zip \
  https://github.com/MetaCubeX/Yacd-meta/archive/refs/heads/gh-pages.zip
unzip -o /tmp/yacd.zip -d /tmp/ui_tmp
mv /tmp/ui_tmp/Yacd-meta-gh-pages/* files/etc/openclash/ui/
rm -rf /tmp/ui_tmp /tmp/yacd.zip
```

### wpad（WiFi STA 模式依赖，避免刷入后 force-reinstall）

> 已在 .config 中包含 `CONFIG_PACKAGE_wpad-basic-mbedtls=y`，编译时会自动打包。
> 无需额外操作。

### 蓝牙工具

> 已在 .config 中包含 `CONFIG_PACKAGE_bluez-daemon=y` 和 `CONFIG_PACKAGE_bluez-utils=y`。
> 无需额外操作。

---

## 完整构建流程（在 1.7 预置文件步骤中追加以上命令，然后正常编译）

执行顺序：
1. 步骤 1.1 ~ 1.6（不变）
2. 步骤 1.7 预置文件（加入上面的 mihomo/geo/ui 下载命令）
3. 步骤 1.8 ~ 1.10（不变）

构建完成后固件即为开箱即用版：
- OpenClash 内核 + 面板 + 分流数据已就绪
- WiFi STA 模式配置后即可连接
- 蓝牙工具已预装
- 背光自动息屏已配置

---

# 编译修复：网关/DNS 不生效

LAN 接口缺少 `defaultroute` 选项导致无默认路由。修复后重新打包：

**用户 i 执行：**

```bash
cd ~/immortalwrt
cat > files/etc/config/network << 'EOF'
config interface 'loopback'
    option device 'lo'
    option proto 'static'
    option ipaddr '127.0.0.1'
    option netmask '255.0.0.0'

config interface 'lan'
    option device 'br-lan'
    option proto 'static'
    option ipaddr '192.168.11.251'
    option netmask '255.255.255.0'
    option gateway '192.168.11.1'
    option dns '192.168.11.1'
    option defaultroute '1'

config device
    option name 'br-lan'
    option type 'bridge'
    list ports 'eth0'
EOF
```

```bash
sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
make -j1 V=s 2>&1 | tee build.log
```

---

# 编译修复：mihomo 内核不兼容 Surface 3 CPU

Surface 3 (Atom x7) 不支持 AMD64 v3 指令集，需要用 `compatible` 版本。

**用户 i 执行：**

```bash
cd ~/immortalwrt
rm -f files/etc/openclash/core/clash_meta
VERSION=$(curl -s https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep tag_name | cut -d'"' -f4)
wget -O /tmp/mihomo.gz \
  "https://github.com/MetaCubeX/mihomo/releases/download/${VERSION}/mihomo-linux-amd64-compatible-${VERSION}.gz"
gunzip -f /tmp/mihomo.gz
mv /tmp/mihomo-linux-amd64-compatible-${VERSION} files/etc/openclash/core/clash_meta 2>/dev/null \
  || mv /tmp/mihomo files/etc/openclash/core/clash_meta
chmod +x files/etc/openclash/core/clash_meta
```

```bash
sed -i '/^CONFIG_PACKAGE_dnsmasq=y/d' .config
make -j1 V=s 2>&1 | tee build.log
```
