# OpenWrt for Ruijie RG-AP720-C (QCA9563) — GitHub Actions 构建

利用 GitHub Actions 构建 OpenWrt 固件，移植到锐捷 RG-AP720-C（QCA9563 + 16MB SPI-NAND + Breed bootloader）。

## 背景

设备硬件：
- SoC: QCA9563（2.4GHz ath9k 内置 wmac）
- 5GHz: QCA9888（ath10k PCIe）
- 交换机: QCA8337（SGMII）
- Flash: 16MB SPI-NAND
- Bootloader: Breed

**Breed 硬限制**：kernel 分区 ≤ 1472k (1,507,328 字节)。

## 本仓库的作用

本仓库**不含** OpenWrt 源码，只含：
1. 设备支持文件（DTS、nand.mk、base-files）
2. 内核配置模板（已裁剪 ZSTD/LZO/DEBUG_FS/SECCOMP/INITRAMFS）
3. 种子 .config（NAS 上已调好的版本）
4. GitHub Actions workflow

构建时 workflow 会：
1. `git clone` 官方 `openwrt/openwrt` 的 `openwrt-24.10` 分支
2. 覆盖上述设备文件与配置
3. `make defconfig` → 裁剪加固 → 下载源码 → 编译 → 打包
4. 上传 `sysupgrade.bin` / `factory.bin` / `kernel.bin` 作为 artifact

## 使用方法

### 1. 推送到 GitHub

```bash
cd gh-build
git init
git add .
git commit -m "init: OpenWrt build for Ruijie RG-AP720-C"
git branch -M main
git remote add origin https://github.com/<你的用户名>/openwrt-ruijie-ap720-c.git
git push -u origin main
```

### 2. 触发构建

进入仓库 **Actions** 标签页 → 选择 "Build OpenWrt for Ruijie RG-AP720-C" → **Run workflow**。

可选输入：
- `openwrt_ref`：OpenWrt 分支/tag（默认 `openwrt-24.10`）
- `build_target`：`defconfig`（从 seed.config 重建）或 `recompile`（只重编译）

### 3. 下载产物

构建完成后（约 1.5-2 小时），在 workflow run 页面下载 artifact：
```
openwrt-ath79-nand-ruijie-rg-ap720-c.zip
├── bin/targets/ath79/nand/  # sysupgrade.bin / factory.bin
├── .config                  # 实际使用的配置
└── build_dir/.../kernel.bin  # 验证大小
```

## 内核裁剪说明

为了把 vmlinux 压到 Breed 限制内，已关闭以下项：
- `TARGET_ROOTFS_INITRAMFS`（根因，曾导致 vmlinux 19.8MB）
- `USE_SECCOMP`（强制 select KERNEL_SECCOMP）
- `PACKAGE_MAC80211_DEBUGFS`（强制 select KERNEL_DEBUG_FS）
- `DEBUG_FS` / `CRYPTO_LZO` / `CRYPTO_ZSTD` 等

workflow 在 `make defconfig` 后还会做"裁剪加固"（sed 强制关闭），防止 defconfig 把这些项写回。

## 注意事项

- kernel.bin 必须满足 `stat -c %s kernel.bin <= 1507328`
- 如果仍超限，可在 workflow 的"Post-defconfig 裁剪加固"步骤追加更多 sed
- 5GHz ath10k 固件（QCA9888 board.bin）已包含在 `files/` 下，构建时会自动打包
- art 分区会被 Breed 擦除，首次刷写需通过 Breed 恢复 caldata

## 文件清单

```
gh-build/
├── .github/workflows/build.yml   # GitHub Actions workflow
├── files/                         # 设备支持文件
│   ├── qca9563_ruijie_rg-ap720-c.dts
│   ├── nand.mk
│   ├── 01_leds
│   ├── 02_network
│   ├── 10-ath9k-eeprom
│   ├── generic-config-6.6
│   └── ath79-config-6.6
└── seed.config                    # 种子 .config
```
