# Hackintosh — 机械革命 无界14 2023（R7-7735H / Radeon 680M）

**OpenCore 1.0.8 引导配置 · 完整折腾记录 · 当前结论：NVMe 已攻克，显卡墙不可逆**

---

## 硬件

| 部件 | 规格 |
|---|---|
| CPU | AMD R7-7735H（Rembrandt，Zen3+） |
| 核显 | **Radeon 680M（RDNA2，macOS 无驱动）** |
| 独显 | 无 |
| 内存 | 16GB LPDDR5-6400 |
| 硬盘 | Intel 660p 512G + Fanxiang S690 1TB（双 M.2 NVMe） |
| 网卡 | RZ608（Wi-Fi 6E） |
| BIOS | 2.22.1282（无 NVMe 禁用项、无 PCIe 速度选项） |

## 成果进展（截至 2026-10-31）

| 项目 | 状态 | 说明 |
|---|---|---|
| **NVMe 启动** | ✅ **已解决** | IONVMeFamily panic（`ReleaseIDNode` → page fault @0x0）用 **`nvme=-1`** boot-arg 攻克，双盘均 `Successfully initialized NVMe drive` |
| **启动深度** | ✅ 最大推进 | 最小化 kext + Kernel/Block `com.apple.driver.AppleSMCRTC` 后，从 hidrm → lockdown_mode → SMCRTC → **DSMOS has arrived** |
| **显卡/显示** | ❌ 不可逆 | 680M 无 macOS 驱动，DSMOS 后显示初始化卡死（社区确认 + 7 种配置实测） |
| 镜像 | ✅ 已核实 | osrecovery default=696-28424 = **macOS 15.4**（Darwin 24.4.0，非 Ventura） |
| 三码 | ⚠️ 已清空 | 发布版不含序列号，**使用前必须自行生成**（见下） |

---

## 使用方法

> ⚠️ **重要：发布版 config.plist 已清空三码（SystemSerialNumber / MLB / SystemUUID / ROM），不生成三码无法启动，且不要使用他人的三码。**

1. **生成自己的三码**（MacPro7,1）：
   - 下载 [macserial](https://github.com/acidanthera/OpenCorePkg/releases)（OpenCorePkg 内 Utilities/macserial），执行：
     ```
     macserial -s -n 20 -m MacPro7,1
     ```
   - 任选一组填入 `EFI/OC/config.plist` → `PlatformInfo/Generic`：
     - `SystemSerialNumber`
     - `MLB`
     - `SystemUUID`（可留空让 OpenCore 自动生成）
2. **准备 EFI 分区**：将本仓库 `EFI` 目录复制到 FAT32 U 盘（或 ESP 分区）根目录，卷标随意。
3. **放入 macOS 恢复镜像**：在 U 盘根目录建 `com.apple.recovery.boot/`，放入 `BaseSystem.dmg` + `BaseSystem.chunklist`（用 macrecovery.py 从 osrecovery.apple.com 获取，default=696-28424 对应 macOS 15.4）。
4. **BIOS 设置**：关闭 Secure Boot / Fast Boot，开启 UEFI USB 引导，从 U 盘启动 OpenCore。
5. 选择 macOS 恢复安装器进入（**注意：本机 680M 无驱动，显示初始化会卡死，见结论**）。

## 关键配置

| 项 | 值 | 作用 |
|---|---|---|
| boot-args | `-v keepsyms=1 debug=0x100 alcid=11 -no_compat_check nvme=-1` | **`nvme=-1` 是 NVMe panic 解法**（禁用 APST/兼容开关） |
| Kernel/Block | `com.apple.driver.AppleSMCRTC` | 跳过 SMCRTC 卡点（`Cannot read PMU offset`），推进到 DSMOS |
| Kernel/Quirks | `DisableIoMapper=True` | 必需（AMD 平台） |
| 启用 kext | Lilu / VirtualSMC / WhateverGreen / AppleALC / NVMeFix | 最小化组合（其余 11 个已禁用） |
| 启用 ACPI | SSDT-EC-USBX / SSDT-SBUS-MCHC / SSDT-OCMARK | |
| Drivers | OpenRuntime / OpenHfsPlus / ResetNvramEntry | |
| DeviceProperties | `PciRoot(0x0)/Pci(0x8,0x1)` 伪装 6600 XT（0x73FF/0x1002） | 680M 无驱动，伪装仅供调试显示路径，不解决卡点 |

## 测试记录（完整版见 `docs/折腾记录.md`）

### 显卡（7 种配置全部卡 DSMOS 后显示初始化）
`-x`（恢复环境不支持）→ `agdpmod=pikera radpg=15` → 删伪装 → 最小化 kext（过 SMCRTC）→ 禁 AppleSMCRTC（过到 DSMOS）→ `-igfxvesa`（无效）→ 伪装+最小化（卡 DSMOS）

### NVMe（4 种配置全 panic → nvme=-1 解决）
双盘（panic）→ 单盘 660p 换槽（panic）→ NVMeFix 重开（panic）→ 换槽+只插 Fanxiang（panic）→ **`nvme=-1`（双盘全过 ✅）**

## 结论

**680M（Rembrandt/RDNA2）在 macOS 无任何驱动**：NootedRed 不支持（实测 0.8.9/0.8.10 必 panic + 社区确认）、无独显、无 eGPU 通道（USB-C 为 USB3.2 Gen1 无 PCIe），**本机装 macOS 不可行（显示墙不可逆）**。

本仓库为**全量方案存档**：将来换独显（AMD RX 6000 系）或换机器后，EFI + `nvme=-1` 解法 + 镜像下载工具可直接复用。`docs/config-backups/` 含 36 个分阶段配置备份，可回溯每一步改动。

## 参考

- [Dortania OpenCore 安装指南（AMD 配置）](https://deepwiki.com/dortania/OpenCore-Install-Guide/4.4-amd-configurations)
- [NVMeFix（acidanthera）](https://github.com/acidanthera/NVMeFix)
- [PC711Probe —— 特定 NVMe 的 Lilu 修复插件先例](https://www.insanelymac.com/forum/topic/363054-i-fixed-the-sk-hynix-pc711-kernel-panic-on-macos-12%E2%80%9315-%E2%80%94-pc711probe-is-now-public/)
- [AMD OS X：IONVMeFamily panic 讨论](https://forum.amd-osx.com/threads/cpu-panic-when-boot-with-ionvmefamily-iostoragefamily.4365/)
