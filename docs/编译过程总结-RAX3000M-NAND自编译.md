# RAX3000M NAND 自编译固件 — 完整过程总结（交接文档）

> 编译日期：2026-09-29
> 固件规格：OpenWrt v25.12.5 / 内核 6.12.108 / MTK 闭源无线驱动
> 产物位置：`D:\AI\RAX3000M\firmware\`（源）+ WSL `/root/openwrt/bin/targets/mediatek/filogic/`（原始输出）

---

## 1. 需求与选型结论

| 需求 | 结论 |
|---|---|
| CMCC RAX3000M **NAND** 版 | 目标 `mediatek/filogic`，设备 `cmcc_rax3000m-nand` |
| 高功率**闭源**无线驱动 | `kmod-mt_wifi 7.6.7.2`（mtk 官方驱动，非开源 mt76） |
| 固件输出 `.bin` 格式 | NAND 产出 `squashfs-sysupgrade.bin`，兼容旧 U-Boot |
| 兼容旧版 U-Boot（bl-mt798x） | `.bin` 非 FIT `.itb`，旧 Web 刷机直接用 |

**关键取舍**：闭源 mt_wifi 驱动最高只支持到 6.12 内核，且不在 ImmortalWrt/OpenWrt 主线里。想要闭源驱动就必须用移植了 MTK feed 的仓库，放弃 6.18 内核。

### 源码出处（均已验证）

| 层 | 仓库 | 用途 |
|---|---|---|
| **核心源码** | https://github.com/shiyu1314/openwrt-source | OpenWrt v25.12.5 + MTK 官方闭源驱动移植（`package/mtk/` 目录：mt_wifi / conninfra / warp / hnat）。编译时使用 **tag `v25.12.5`**（commit 注明 "MT798x mtk-wifi release: v25.12.5 kernel 6.12"） |
| **定制参考** | https://github.com/kubulama/openwrt-rax3000m （25.12 分支） | op.sh 定制脚本（nginx 替换 uhttpd、daed 等）+ patch + files + `config/config-common` 驱动配置。本次仅合并了其 **config-common**，未叠加 op.sh 全套定制 |
| **旧版 U-Boot** | https://github.com/Yuzhii0718/bl-mt798x-dhcpd | bl-mt798x + DHCP 自动获取 IP，Web 刷机 192.168.1.1 |
| **备选仓库** | hanwckf/immortalwrt-mt798x（5.4 内核，闭源驱动）、padavanonly/immortalwrt-mt798x（237，默认 25dBm） | 未采用，内核过旧 |

---

## 2. 编译环境（Windows 主机 + WSL2）

| 项 | 值 |
|---|---|
| 发行版 | WSL distro 名 **`openwrt`**（Ubuntu 22.04.5 rootfs 导入） |
| 虚拟磁盘 | `D:\WSL\openwrt\ext4.vhdx`（D 盘，约 955GB 可用，**不放 C 盘**） |
| rootfs 来源 | `https://cloud-images.ubuntu.com/wsl/jammy/current/ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz`（326MB，注意 jammy 目录下文件名带 `ubuntu22.04lts`，不带会 404） |
| 导入命令 | `wsl --import openwrt D:\WSL\openwrt D:\WSL\ubuntu-jammy-wsl.rootfs.tar.gz --version 2` |
| 网络模式 | **mirrored**（`C:\Users\Administrator\.wslconfig` 中 `networkingMode=mirrored`），GitHub 速度从 223KB/s 提升到 4.6MB/s |
| 硬件 | 20 核 / 15GB 内存 / 编译全程约 1.5 小时（含排错重启多次） |
| 源码路径 | WSL 内 `/root/openwrt`（**必须放 WSL 内部文件系统**，勿用 /mnt/d，IO 性能差一个数量级） |

### 依赖安装（WSL 内 root 执行）

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update && apt-get install -y tar build-essential flex bison cmake g++ gawk \
  gcc-multilib g++-multilib gettext git libfuse-dev libncurses5-dev libssl-dev \
  python3 python3-pip python3-ply python3-distutils python3-pyelftools rsync unzip \
  zlib1g-dev file wget subversion patch upx-ucl autoconf automake curl asciidoc \
  binutils bzip2 lib32gcc-s1 libc6-dev-i386 uglifyjs msmtp texinfo libreadline-dev \
  libglib2.0-dev xmlto libelf-dev libtool autopoint antlr3 gperf ccache swig \
  coreutils haveged scons libpython3-dev rename qemu-utils clang llvm xxd zstd
```

### .wslconfig（`C:\Users\Administrator\.wslconfig`）

```ini
[wsl2]
networkingMode=mirrored
autoProxy=true
dnsTunneling=true
```

---

## 3. 配置要点

### 获取源码与 feeds

```bash
wsl -d openwrt -u root
cd /root && git clone https://github.com/shiyu1314/openwrt-source openwrt
cd openwrt && git checkout v25.12.5
./scripts/feeds update -a && ./scripts/feeds install -a
```

### 目标与驱动配置

```bash
# 基础目标
cat >> .config <<'EOF'
CONFIG_TARGET_mediatek=y
CONFIG_TARGET_mediatek_filogic=y
CONFIG_TARGET_DEVICE_mediatek_filogic_DEVICE_cmcc_rax3000m-nand=y
EOF

# 合并 kubulama 的驱动配置（mt_wifi 7.6.7.2 / warp / conninfra / hnat / mtwifi-cfg 等）
curl -sL https://raw.githubusercontent.com/kubulama/openwrt-rax3000m/25.12/config/config-common >> .config

# 追加应用插件（按需）
cat >> .config <<'EOF'
CONFIG_PACKAGE_luci-app-samba4=y
CONFIG_PACKAGE_luci-i18n-samba4-zh-cn=y
CONFIG_PACKAGE_luci-app-ttyd=y
CONFIG_PACKAGE_luci-i18n-ttyd-zh-cn=y
EOF

make defconfig   # 未知符号自动丢弃，不会报错
```

**⚠️ 两个配置坑**：
1. `CONFIG_TARGET_DEVICE_...` 设备行**必须单独追加后再跑 defconfig**——若在合并大段 config 之后再追加设备行（本例中设备行和 config-common 一起被 defconfig 处理时曾丢失），先确认 `grep cmcc_rax3000m-nand .config` 有结果再开编。
2. feeds 锁定在固定 commit，snapshot 的 LuCI feed 里**没有** zerotier / socat / diskman 的 LuCI 应用（只有 samba4、ttyd），`luci-app-zerotier` 等符号会被 defconfig 静默丢弃。缺的插件要么换 feed commit，要么编译完成后 opkg 在线装。

### 最终固件内容（已验证 manifest）

- **驱动**：kmod-mt_wifi 7.6.7.2 + kmod-warp + kmod-conninfra + kmod-mediatek_hnat（HNAT 硬件 NAT）
- **无线**：wpad-openssl（WPA3）+ wireless-regdb + **luci-app-mtwifi-cfg**（闭源驱动专用无线界面，功率/区域调节处）
- **加速**：luci-app-turboacc-mtk（HNAT+WED 联动开关）
- **应用**：luci-app-eqos-mtk（限速）、samba4（4.22.7）、ttyd、package-manager、firewall（nftables + **fullcone** + tproxy/offload）
- **Web**：nginx-ssl 1.26.3（brotli/rtmp/stream 模块），LuCI 跑在 nginx 上
- **USB3**：xhci-mtk + UAS；文件系统 ext4/ntfs3/exfat/vfat（插 U 盘即用）
- **工具**：htop、bash、dnsmasq-full、openssl-util；全应用带中文语言包
- **没带**：科学上网类、tailscale/zerotier、docker、SQM。kmod-tun 和 tproxy 模块已内置，后续可 opkg 在线加应用层插件。

---

## 4. 编译命令与排错记录

### 最终可用的完整编译命令（复现/续编用这一条）

```bash
wsl -d openwrt -u root -- bash -c 'cd /root/openwrt && export FORCE_UNSAFE_CONFIGURE=1 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin && make -j8'
```

> **三要素缺一不可**：`FORCE_UNSAFE_CONFIGURE=1`（root 编译）、干净 PATH（见坑 ③）、单引号包裹（见坑 ⑤）。

### 踩坑全记录（按时间序）

| # | 现象 | 根因 | 修复 |
|---|---|---|---|
| ① | 原 `Ubuntu` 发行版无法启动 | 旧注册项的 ext4.vhdx 文件丢失 | `wsl --unregister Ubuntu`，重新导入 rootfs 到 D 盘 |
| ② | WSL 内 GitHub 下载 223KB/s（宿主机 5.9MB/s） | WSL 默认 NAT 网络慢 | `.wslconfig` 设 `networkingMode=mirrored` + `wsl --shutdown` 重启，提速 20 倍 |
| ③ | `tools/tar` 编译失败："newly created file is older than distributed files" | **root 身份编译**触发 tar 安全检查 | `export FORCE_UNSAFE_CONFIGURE=1` |
| ④ | popt / perl(host) / python3(host) / htop 多包随机失败 | **Windows PATH（含空格的 Program Files）被 WSL 继承**，`find -execdir` 出于安全拒绝执行 | 编译时强制 `PATH=/usr/local/sbin:...:/bin`（最终命令已固化） |
| ⑤ | `bash -c "echo $TAG"` 拿到空值、变量被提前展开 | 外层 Git Bash 对**双引号**内的 `$VAR` 提前展开 | `wsl.exe ... bash -c` 一律用**单引号**包裹 |
| ⑥ | 一次失败误判为时钟漂移 | 实为坑④ | 无需修时钟，清理 PATH 后断点续编即可 |

**经验法则**：OpenWrt 编译中途失败后，修完原因直接重跑同一条 make 命令即可断点续编，不会从头来。单包排查用：
```bash
make package/feeds/packages/<包名>/{clean,compile} V=s
```

---

## 5. 产物

| 文件 | 大小 | SHA256 前 8 位 | 用途 |
|---|---|---|---|
| `openwrt-mediatek-filogic-cmcc_rax3000m-nand-squashfs-sysupgrade.bin` | 32MB | `223815f3` | **主固件**，旧 U-Boot Web / sysupgrade 直接刷 |
| `openwrt-mediatek-filogic-cmcc_rax3000m-nand-initramfs-kernel.bin` | 29MB | `89d4ffb2` | TFTP 救砖用 |
| `.itb`（FIT 格式） | — | — | 仅 FIT U-Boot 用，本次场景用不到 |

位置：`D:\AI\RAX3000M\firmware\`（已从 WSL 复制出）；WSL 内原始路径 `/root/openwrt/bin/targets/mediatek/filogic/`。完整包清单见同目录 `.manifest` 文件。

---

## 6. 刷机要点（交接必读）

1. **刷前必备份 Factory 分区**（丢了无线校准数据不可恢复）：
   ```bash
   dd if=$(grep -w Factory /proc/mtd | cut -d: -f1) of=/tmp/factory.bin
   ```
2. 确认 FIP 是**旧版 bl-mt798x** U-Boot：旧 Web 界面（192.168.1.1）直接上传 `.bin` 即可。若是官方 FIT U-Boot，此 `.bin` 不能直接用，需先换 U-Boot。
3. 已刷官方 OpenWrt 23.05+ 的话：`sysupgrade -n /tmp/*.bin`（`-n` 不保留配置）。
4. 刷完默认 **root 无密码，LAN 192.168.1.1**，登录后立即设密码。
5. 无线高功率设置在 **LuCI → 网络 → 无线（mtwifi-cfg）**，注意区域/功率合规风险自担。

---

## 7. 后续任务速查

| 想做的事 | 操作 |
|---|---|
| 改配置重编 | WSL 内改 `.config`（`make menuconfig`）→ 跑第 4 节的编译命令，断点续编 |
| 加插件（应用层） | 刷机后 LuCI → 系统 → 软件包（opkg）在线装，或改 .config 重编 |
| 加闭源驱动相关模块 | 必须改 .config 重编（kmod 不能在线装，内核版本锁定 6.12.108） |
| 升级到 shiyu1314 新 tag | `git fetch --tags && git checkout <新tag>` → feeds update → 重编 |
| 完整复刻 kubulama 定制 | 叠加其 op.sh（nginx 替换 uhttpd、daed 1.28 pnpm 构建、自建源），是链路中最易失败环节，本次未做 |
