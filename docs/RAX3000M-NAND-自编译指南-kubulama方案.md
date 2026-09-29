# RAX3000M NAND 自编译指南 · kubulama 方案（闭源驱动 + .bin + 旧 U-Boot）

> 整理日期：2026-09-29
> 参考项目：https://github.com/kubulama/openwrt-rax3000m （分支 25.12）

## 一、方案概览

| 项目 | 内容 |
|---|---|
| 编译产物（已验证） | `openwrt-mediatek-filogic-cmcc_rax3000m-nand-squashfs-sysupgrade.bin`（56MB，.bin ✅） |
| 基线 | OpenWrt v25.12.5（checkout 最新 v25.* tag） |
| 内核 | 6.12（非 6.18，闭源驱动上限） |
| 无线驱动 | **MTK 闭源 mt_wifi**（mtk-openwrt-feeds 移植），支持 **WED + HNAT 硬件加速** |
| 核心源码 | `shiyu1314/openwrt-source`（基于 chasey-dev/immortalwrt-mt798x-rebase 思路，MTK feeds 移植到 25.12） |
| U-Boot | 旧版 bl-mt798x 系，推荐 [bl-mt798x-dhcpd](https://github.com/Yuzhii0718/bl-mt798x-dhcpd)（旧版+DHCP 自动获取 IP） |
| 支持设备 | RAX3000M NAND / EMMC / 256M NAND、XR30、JCG Q30 PRO |
| 默认口令 | root / password（刷机后立即改） |
| 同源官方发布 | kubulama Releases 有现成固件（v25.12.5-20260826），可直接下来自用，自编译仅为定制 |

kubulama 仓库本体是一套 **GitHub Actions 云编译 overlay**（op.sh 定制脚本 + patch + files 覆盖 + config），
叠加在 `shiyu1314/openwrt-source` 之上。本地编译既可以"最小化"（只编核心源码），也可以"完整复刻"（把 overlay 全搬下来）。

## 二、最小化本地编译（核心固件，不套 kubulama 定制）

```bash
# 0. Ubuntu 22.04 / Debian 11+，非 root，路径无中文无空格，磁盘 ≥ 40GB
sudo apt-get update
sudo apt-get install -y tar build-essential flex bison cmake g++ gawk gcc-multilib g++-multilib \
  gettext git libfuse-dev libncurses5-dev libssl-dev python3 python3-pip python3-ply python3-distutils \
  python3-pyelftools rsync unzip zlib1g-dev file wget subversion patch upx-ucl autoconf automake curl \
  asciidoc binutils bzip2 lib32gcc-s1 libc6-dev-i386 uglifyjs msmtp texinfo libreadline-dev \
  libglib2.0-dev xmlto libelf-dev libtool autopoint antlr3 gperf ccache swig coreutils haveged scons \
  libpython3-dev rename qemu-utils clang llvm

# 1. 克隆核心源码并切到最新 v25 tag
git clone https://github.com/shiyu1314/openwrt-source openwrt
cd openwrt
git checkout $(git tag --sort=taggerdate --list 'v25.*' | tail -1)

# 2. feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 3. 配置（目标：RAX3000M NAND）
cat >> .config <<'EOF'
CONFIG_TARGET_mediatek=y
CONFIG_TARGET_mediatek_filogic=y
CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_cmcc_rax3000m-nand=y
EOF
make defconfig
# 此步之后可 make menuconfig 微调插件（闭源无线驱动默认已含在源码内）

# 4. 下载依赖并编译
make download -j8
find dl -size -1024c -delete   # 清理下载不完整的文件
make -j$(nproc) V=s            # 首次建议保留 V=s 日志

# 5. 产物
# bin/targets/mediatek/filogic/
#   openwrt-mediatek-filogic-cmcc_rax3000m-nand-squashfs-sysupgrade.bin   <- 旧 U-Boot 用（.bin）
#   openwrt-mediatek-filogic-cmcc_rax3000m-squashfs-sysupgrade.itb        <- 官方 FIT U-Boot 用
#   openwrt-mediatek-filogic-cmcc_rax3000m-initramfs-recovery.itb         <- TFTP 救砖用
```

## 三、完整复刻 kubulama 定制（可选）

在最小化编译的步骤 1~2 之间，额外叠加 kubulama 的定制层：

```bash
# 克隆 overlay
git clone -b 25.12 https://github.com/kubulama/openwrt-rax3000m overlay

# 1) 打补丁（幂等，官方源码补丁 + luci 补丁）
cp -rf overlay/patch/diy/*.patch .
cp -rf overlay/patch/luci/*.patch feeds/luci/

# 2) 跑定制脚本（nginx 替换 uhttpd、daed 1.28、sbwml 新版包、
#    vermagic 定制、quic/http3、luci 调优等——改动量大，编译时间显著增加）
bash overlay/sh/op.sh

# 3) nginx 相关配置覆盖
cp -rf overlay/patch/nginx/luci.locations feeds/packages/net/nginx/files-luci-support/
cp -rf overlay/patch/nginx/uci.conf.template feeds/packages/net/nginx-util/files/

# 4) files 覆盖与公共配置
mv overlay/files files
cat overlay/config/config-common >> .config
```

注意事项：
- op.sh 会**移除 samba4/aria2/mosdns/sing-box/adguardhome 等 feeds 包**并改用作者自建源（`package/xd`、`package/porxy`）替代，完整复刻后插件体系与官方源不同，在线装包需用其自建源。
- op.sh 里的 daed/luci-app-daed 前端走 pnpm monorepo 源码构建，是整个流程中最容易失败的一环；不需要 daed 的话建议最小化编译，或跳过该脚本仅打补丁。
- 定制 LAN IP：`sed -i "s/192.168.1.1/192.168.2.1/" package/base-files/files/bin/config_generate`（kubulama 默认 192.168.2.1）。

## 四、U-Boot 与刷机

1. **U-Boot**：[bl-mt798x-dhcpd](https://github.com/Yuzhii0718/bl-mt798x-dhcpd)（旧版 bl-mt798x 分支，支持 DHCP 自动下发 IP）。
   已刷过旧版 U-Boot 的机器可不动；写 FIP 分区：`mtd write mt7981_cmcc_rax3000m-fip-fixed-parts.bin FIP`。
2. 进入 U-Boot Web（按住 RESET 上电，灯变绿）：上传 `...cmcc_rax3000m-nand-squashfs-sysupgrade.bin`。
3. 刷完重启，后台 `http://192.168.2.1`（kubulama 定制）或 `http://192.168.1.1`（最小化编译），root / password，**登录后立即改密码**。
4. 已在官方 OpenWrt 系统内升级：`sysupgrade /tmp/*.bin`（从 23.05/24.10/25.12 官方版升上来配置可不保留更稳）。

⚠️ 刷前备份全部分区（尤其 Factory），NAND 版救砖依赖 TFTP + `initramfs-recovery.itb`。

## 五、参考链接

- kubulama 项目（overlay + 云编译）：https://github.com/kubulama/openwrt-rax3000m
- 现成固件 Releases：https://github.com/kubulama/openwrt-rax3000m/releases
- 核心源码：https://github.com/shiyu1314/openwrt-source
- MTK feeds 移植核心：https://github.com/chasey-dev/immortalwrt-mt798x-rebase
- 旧版 U-Boot（DHCP）：https://github.com/Yuzhii0718/bl-mt798x-dhcpd
- 经典版 U-Boot：https://github.com/hanwckf/bl-mt798x/releases
