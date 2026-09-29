# myop_rax3000m

CMCC RAX3000M（NAND 版）自定义 OpenWrt 固件项目：源码、配置、云编译流水线与文档一站式管理。

## 固件特性

| 项目 | 说明 |
|---|---|
| 基础源码 | [shiyu1314/openwrt-source](https://github.com/shiyu1314/openwrt-source) tag `v25.12.5`（OpenWrt 25.12.5） |
| 内核 | **6.12.108** |
| 无线驱动 | **MTK 闭源 mt_wifi 7.6.7.2**（20250408 固件）+ `kmod-warp`（WED 无线加速）+ `kmod-conninfra`，高功率 |
| 有线加速 | `kmod-mediatek_hnat` 硬件 NAT + luci-app-turboacc-mtk |
| 固件格式 | `.bin`（squashfs sysupgrade），**兼容旧版 bl-mt798x U-Boot**（Web 刷机 192.168.1.1） |
| 体积 | 42MB |

## 固件自带插件

- **OpenClash**（vernesong master 最新版，含 ruby 运行时）
- **iStore** 应用商店（luci-app-store + taskd 全家桶，已修补 +tar 依赖问题）
- **UPnP**（miniupnpd，nftables 后端）
- **网络共享** Samba4、**网页终端** ttyd（菜单已移至「系统」分组）、软件包管理、EQoS 限速（MTK 版）
- 无线管理 luci-app-mtwifi-cfg（功率/区域调节入口）、wpad-openssl（WPA3）
- USB3（xhci-mtk + UAS）、ext4/ntfs3/exfat 文件系统、fullcone NAT、全中文界面



## 仓库结构

```
myop_rax3000m/
├── README.md                          ← 本文件
├── seed.config                        ← 编译配置种子（make defconfig 用，479 行精简）
├── .github/workflows/
│   └── rax3000m-nand.yml              ← GitHub Actions 云编译流水线（手动触发）
├── docs/
│   ├── 编译过程总结-RAX3000M-NAND自编译.md   ← 完整交接文档（环境/配置/踩坑/复现）
│   ├── RAX3000M-NAND-自编译指南-kubulama方案.md
│   └── RAX3000M-NAND-自编译指南-闭源高功率.md
└── firmware/                          ← 本地编译产物（v2）
    ├── openwrt-mediatek-filogic-cmcc_rax3000m-nand-squashfs-sysupgrade.bin  (42MB)
    ├── openwrt-mediatek-filogic-cmcc_rax3000m-nand-initramfs-kernel.bin     (37MB, TFTP 救砖)
    └── openwrt-mediatek-filogic.manifest
```

## 云编译（GitHub Actions）

1. 进入仓库 **Actions** 页 → 选择 **Build RAX3000M NAND** → **Run workflow**
2. 全自动流程：克隆 v25.12.5 → 安装依赖 → feeds（含 istore）→ 打补丁（ttyd 菜单 / iStore +tar）→ 按 seed.config 编译 → 产物上传 Artifacts（保留 30 天）
3. 改配置：编辑 `seed.config` 后提交推送，或手动触发

流水线要点（踩坑固化）：

- `feeds.conf` 会整体遮蔽 `feeds.conf.default`，必须先复制默认列表再追加自定义 feed
- `luci-app-store` 依赖 `+tar` 会生成未定义符号 `PACKAGE_TAR_XZ` 导致 kconfig 判死，已 sed 移除
- root 编译需 `export FORCE_UNSAFE_CONFIGURE=1`

## 本地编译（WSL2 / Linux）

依赖安装与完整过程详见 [docs/编译过程总结-RAX3000M-NAND自编译.md](docs/编译过程总结-RAX3000M-NAND自编译.md)。速查：

```bash
git clone https://github.com/shiyu1314/openwrt-source openwrt
cd openwrt && git checkout v25.12.5
# feeds 配置（注意先复制 default 再追加，见上）
./scripts/feeds update -a && ./scripts/feeds install -a
cp /path/to/seed.config .config && make defconfig
export FORCE_UNSAFE_CONFIGURE=1
make download -j8 && make -j$(nproc)
# 产物: bin/targets/mediatek/filogic/*rax3000m-nand-squashfs-sysupgrade.bin
```

已验证环境：Windows 11 + WSL2 Ubuntu 22.04（mirrored 网络），20 核 / 15GB 内存，全量编译约 3.5 小时。


## 刷机要点（NAND 版）

1. **刷前务必备份 factory 分区**（无线校准数据，丢失不可恢复）：
   ```bash
   dd if=$(grep -w Factory /proc/mtd | cut -d: -f1) of=/tmp/factory.bin
   ```
2. 确认 FIP 为旧版 bl-mt798x U-Boot：Web（192.168.1.1）直接上传 sysupgrade.bin
3. 已刷官方 OpenWrt 23.05+ 可直接 `sysupgrade /tmp/*.bin`
4. 原厂固件需先按 ToH 流程 TFTP 刷 initramfs，或先刷 H 大 U-Boot
5. 默认 root 无密码，**登录后立即改密**；LAN 默认 192.168.1.1

## 相关仓库

- 源码基础：[shiyu1314/openwrt-source](https://github.com/shiyu1314/openwrt-source)（MTK feeds 闭源驱动移植）
- 参考项目：[kubulama/openwrt-rax3000m](https://github.com/kubulama/openwrt-rax3000m)
- U-Boot：[Yuzhii0718/bl-mt798x-dhcpd](https://github.com/Yuzhii0718/bl-mt798x-dhcpd)（旧版 + DHCP）

## License

源码遵循各上游仓库许可；本仓库文档与配置仅供个人学习使用。
