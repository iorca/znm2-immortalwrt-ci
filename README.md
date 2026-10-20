# ZN-M2 自动编译 (VIKINGYFY/immortalwrt · owrt 分支，每两周一次)

给 **兆能 ZN-M2 (IPQ6018, qualcommax/ipq60xx)** 做的固件编译骨架（每两周一次）。
**不需要 fork 源码仓库**——CI 运行时浅克隆 `VIKINGYFY/immortalwrt` 的 `owrt` 分支，套上本仓库的 `Config/znm2.seed.config` 直接编。

配置已对齐 xdm2211/ZN-M2-VIKINGYFY 的真实可编 ZN-M2 配置，并叠加了 OpenClash + USB 3.0。

## 目录结构
```
.
├── .github/workflows/build-znm2.yml   # 编译 workflow（含双周 gate）
├── Config/znm2.seed.config            # zn_m2 target + 包选择种子
└── README.md
```

## ⚠️ 硬警告：Actions 分钟数
- 单次冷编译（无缓存）约 **60–120 分钟**；每两周一次 ≈ 每月 2 次、**120–240 分钟**。
- GitHub 免费额度：私有仓库每月仅 **2000 分钟**——每两周一次够用；但仍建议 Public（分钟无上限、固件/Release 可直接访问）。
- **对策：把这个 CI 仓库设为 Public。** Public 仓库的 Actions 分钟数免费且无上限。
- 源码仓库 `VIKINGYFY/immortalwrt` 本身是 Public，浅克隆无授权问题。

## 部署步骤
1. 在 GitHub 新建一个仓库（建议 Public），把这 3 个文件按目录结构推上去。
2. `Settings → Actions → General → Workflow permissions` 选 **Read and write permissions**（发 Release 要写权限）。
3. `Actions` 标签页确认 `Build ZN-M2` workflow 已启用。
4. 自动跑：每周一触发一次，**按 ISO 周奇偶只在偶数周真正编译（≈每 14 天一次）**；也能在 `Actions → Run workflow` 随时手动触发，手动不受频率限制。

## 产物与自动清理
- **Release（推荐，自动发布）**：每次编译成功自动发到固定 tag `zn-m2-owrt-latest`，**overwrite 覆盖更新**——永远只有一份、始终是最新，不会堆积。Release body 带源码 commit 和编译时间。固件下载走这里。
- **Artifacts**：每次构建另存一份，**保留 30 天后 GitHub 自动删除**。按每两周一次的频率，稳态只有当前 + 上一份（偶发三份），到点自动清，无需手动处理。

## 已包含的包（相对参考配置）
- **target**：`qualcommax / ipq60xx / zn_m2`（设备符号 `CONFIG_TARGET_DEVICE_qualcommax_ipq60xx_DEVICE_zn_m2`，与 xdm2211 实际能编过的配置一致）。
- **Luci**：luci + luci-ssl + 中文语言包 + firewall/opkg/upnp/ttyd/commands。
- **OpenClash**：`luci-app-openclash`，由 workflow 自动追加 `vernesong/OpenClash` 的 `dev` 分支 feed 后安装。
- **USB 3.0 + 存储**：`kmod-usb3` / `kmod-usb-dwc3` / `kmod-usb-dwc3-qcom`（DWC3 主控）+ `kmod-usb-storage(-uas)` / `block-mount` / `e2fsprogs` / `kmod-fs-{ext4,ntfs,exfat,vfat}`；另含 `kmod-usb-serial-qualcomm`（USB 上网卡/模组 tty）。
- **基础工具**：curl、htop。

## 关于 NSS / 硬件加速（重要，先读）
- 你最初点名的 `kmod-qca-nss-drv` / `kmod-qca-nss-ecm` / `nss-packages` **在本 fork 不适用，不要加**。已核验：
  - `feeds.conf.default`（owrt）里**没有** nss-packages feed；`immortalwrt/packages/kernel` 也没有任何 qca/nss 目录。
  - xdm2211 真实可编的 ZN-M2 配置里**完全没有**这三个名字，且其 `Settings.sh` 明确 `CONFIG_FEED_nss_packages=n`（把经典 NSS feed 关掉）。
  - 本 fork 的 qualcommax 加速走 **in-tree 的 qca_ppe / qca_edma 内核补丁**（`target/linux/qualcommax/patches-6.18` 的 `add-ppe.patch` / `add-EDMA.patch`），已编进内核，**无需、也选不了**那三个包。
- 也就是说「满血 PPE 驱动」指的是这套 in-tree PPE/EDMA，不是经典 NSS 模块。

## 分支说明（owrt vs main）
- 你指定用 **`owrt`** 分支，workflow 写死 `--branch owrt`。
- 注意：参考仓库 xdm2211 用的是 **`main`** 分支。两个分支的 qualcommax 目标结构一致、设备符号相同，但源码提交点不同。
- 若想完全对齐 xdm2211，把 `build-znm2.yml` 里 `--branch owrt` 改成 `--branch main` 即可。

## 自定义
- **改刷的包**：编辑 `Config/znm2.seed.config`，加/删 `CONFIG_PACKAGE_xxx=y`，推送后会自动重新编译。
- **改编译频率/时间**：`cron` 当前 `'0 20 * * 1'`（每周一 20:00 UTC = 北京周二 04:00 触发）；实际每两周编一次由 `Biweekly gate` 步按 ISO 周奇偶决定（偶数周才编）。想每周都编，删掉 gate 步、各步 `if` 恒 true 即可；想改锚定周，改 gate 里的奇偶判断。
- **加别的 luci 插件**（如 passwall / homeproxy）：仿照 OpenClash，先在 workflow 的 `Prepare config` 步骤里 `echo "src-git xxx ..." >> feeds.conf.default` 加对应 feed，再在 seed 里加 `CONFIG_PACKAGE_luci-app-xxx=y`。

## 排查
- **顺序陷阱（重要）**：`easimon/maximize-build-space` 会把新 LVM 卷挂到 workspace 路径、盖住已检出文件，所以它必须放在 `actions/checkout` **之前**（现顺序：gate → maximize → checkout）。若报 `Config/` 或 `.config` 找不到，先查这两步顺序。
- 若构建报 “no target device” / 选不到 zn_m2：设备符号应为 `CONFIG_TARGET_DEVICE_qualcommax_ipq60xx_DEVICE_zn_m2=y`（已修正，原错写少一段 `DEVICE_`）。
- 首次冷编译很慢且吃磁盘；`easimon/maximize-build-space` 已清掉 dotnet/android/haskell 等腾空间，`dl` 与 `ccache` 已缓存，二次编译明显更快。
- owrt 分支用的是 APK 包管理器（非 opkg），镜像仍落在 `bin/targets/qualcommax/ipq60xx/`。
