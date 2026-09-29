# RAX3000M NAND 自编译指南：闭源高功率驱动 + .bin 固件 + 旧版 U-Boot

> 整理日期：2026-09-29

## 一、方案选型

| 仓库 | 基线 | 内核 | 无线驱动 | 输出格式 | 旧 U-Boot 兼容 |
|---|---|---|---|---|---|
| hanwckf/immortalwrt-mt798x（master） | ImmortalWrt 21.02 | 5.4.x | mt_wifi 7.6.6.1 闭源 | `squashfs-factory.bin` | ✅ 直接刷 |
| padavanonly/immortalwrt-mt798x（237 大佬，23.05 系） | ImmortalWrt 23.05 | 5.4.x | mt_wifi 闭源 + 默认高功率 | `.bin` | ✅ 直接刷 |
| padavanonly/immortalwrt-mt798x-24.10（分支 openwrt-24.10-6.6） | ImmortalWrt 24.10 | **6.6.133** | mt_wifi 闭源（较新版） | NAND 新源码为 `.itb` ⚠️ | ⚠️ 需 FIT U-Boot |
| ImmortalWrt master / OpenWrt 主线 | 主线 | 6.18 | 仅开源 mt76 | `.itb` | ❌ |

**结论：**
- 三条硬性要求（闭源高功率 + .bin + 旧 U-Boot）→ 选 **hanwckf 仓库**（最稳、社区验证最久）或 **237 仓库 23.05 系**（默认功率更高，2.4G/5G 均 25dBm）。
- 闭源驱动不支持 6.18 内核，上限为 6.6。若坚持 6.18 只能用开源 mt76（主线、功率受 CN 区域限制约 20dBm）。
- 237 的 24.10/6.6 分支最新源码 NAND 版输出 `.itb`，如需 `.bin` + 旧 U-Boot，需检出较早提交或修改 image 的 Makefile，不推荐新手。

## 二、推荐：hanwckf/immortalwrt-mt798x 编译步骤

```bash
# 0. Linux x86_64 主机（Ubuntu 22.04 / Debian 11+），非 root 用户，路径无中文无空格
sudo apt update -y && sudo apt full-upgrade -y
sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
  bzip2 ccache clang cmake cpio curl device-tree-compiler ecj fastjar flex gawk gettext gcc-multilib \
  g++-multilib git gnutls-dev gperf haveged help2man intltool lib32gcc-s1 libc6-dev-i386 libelf-dev \
  libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses-dev libpython3-dev \
  libreadline-dev libssl-dev libtool libyaml-dev libz-dev lld llvm lrzsz mkisofs msmtp nano \
  ninja-build p7zip p7zip-full patch pkgconf python3 python3-pip python3-ply python3-docutils \
  python3-pyelftools qemu-utils re2c rsync squashfs-tools subversion swig texinfo uglifyjs \
  upx-ucl unzip vim wget xmlto xxd zlib1g-dev zstd

# 1. 克隆源码
git clone https://github.com/hanwckf/immortalwrt-mt798x
cd immortalwrt-mt798x

# 2. feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 3. 配置目标
make menuconfig
#   Target System  -> MediaTek MT7981 (ARM Cortex-A53)
#   Target Profile -> CMCC RAX3000M
#   无线保持默认 mtwifi 闭源驱动（mtwifi-cfg 配置工具）

# 4. 编译（首次建议 -j1 便于排错，之后可 -j$(nproc)）
make -j1 V=s

# 5. 产物
# bin/targets/mediatek/mt7981/
#   immortalwrt-mediatek-mt7981-cmcc_rax3000m-squashfs-factory.bin   <- 旧 U-Boot 网页刷这个
#   immortalwrt-mediatek-mt7981-cmcc_rax3000m-squashfs-sysupgrade.bin
```

## 三、高功率选项（可选增强）

1. **237 仓库自带高功率**：`padavanonly/immortalwrt-mt798x`（23.05 系）默认 2.4G/5G 开启 25dBm 高功率模式，最省事。
2. **eeprom 替换法**（hanwckf 仓库适用）：用 H3C NX30 Pro 提取的 eeprom 替换原厂 eeprom（237 大佬提取），
   原厂功率 2.4G 23dBm / 5G 22dBm → 替换后 2.4G 25dBm / 5G 24dBm。
3. 编译期替换：将 eeprom 文件放入 `files/lib/firmware/` 覆盖，或参考 GD-Slime/Actions-rax3000m-nand 的做法。

## 四、刷机（旧版 U-Boot：hanwckf bl-mt798x）

1. 前提：FIP 分区已是 `mt7981_cmcc_rax3000m-fip-fixed-parts.bin`（bl-mt798x release 下载）。
2. 牙签按住 RESET 上电，指示灯变绿松手 → 进入 U-Boot Web（192.168.1.1，本机静态 IP 192.168.1.100/网关 192.168.1.1）。
3. 上传 `immortalwrt-mediatek-mt7981-cmcc_rax3000m-squashfs-factory.bin`，确认刷写。
4. 刷完重启，后台 http://192.168.1.1（hanwckf 版）或 http://192.168.6.1（237 版），root 无密码。

⚠️ 刷任何固件前务必备份 mtd 全部分区，尤其 **Factory**（无线校准数据，丢了无线功率/ MAC 无法恢复）。

## 五、参考

- hanwckf 仓库：https://github.com/hanwckf/immortalwrt-mt798x
- 项目介绍（hanwckf 博客）：https://cmi.hanwckf.top/p/immortalwrt-mt798x/
- 237 仓库（24.10/6.6）：https://github.com/padavanonly/immortalwrt-mt798x-24.10
- U-Boot 下载：https://github.com/hanwckf/bl-mt798x/releases
- 恩山参考帖：闭源高功率固件 https://www.right.com.cn/forum/thread-8420900-1-24.html
