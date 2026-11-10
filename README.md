# Hackintosh — 机械革命 无界14 2023（R7-7735H / Radeon 680M）

**OpenCore 1.0.8 引导配置 · 完整折腾记录 · 当前状态：已显示白苹果启动界面，GPU 仍未完成**

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

## 成果进展（截至 2026-11-10）

| 项目 | 状态 | 说明 |
|---|---|---|
| **PCI / 设备树** | ✅ 已通过 | `ui-lab` 完成 PCI resource allocation、device tree 构建和 `iterate finalize: end(0)` |
| **白苹果界面** | ✅ 已出现 | 2026-11-10 18:54 已显示白苹果和启动进度条，绕过此前 `console setting @ 42000001` 卡点 |
| **GPU / WindowServer** | ⚠️ WIP | Recovery 图形界面仍未完成，`AMDRadeonX6000` / `RDNA3Driver` 均未作为稳定驱动启用 |
| **NVMe 历史解法** | ✅ 已归档 | 旧配置曾用 `nvme=-1` 解决 IONVMeFamily panic；当前 `ui-lab` 为 USB Recovery 诊断而暂时 Exclude IONVMeFamily |
| 镜像 | ✅ 已核实 | osrecovery default=696-28424 = **macOS 15.4**（Darwin 24.4.0，非 Ventura） |
| 三码 | ⚠️ 已清空 | 发布版不含序列号，**使用前必须自行生成**（见下） |

---

## 使用方法

> ⚠️ **重要：发布版 config.plist 已清空三码（SystemSerialNumber / MLB / SystemUUID / ROM），不生成三码无法启动，且不要使用他人的三码。**

1. **生成自己的三码**（iMacPro1,1）：
   - 下载 [macserial](https://github.com/acidanthera/OpenCorePkg/releases)（OpenCorePkg 内 Utilities/macserial），执行：
     ```
     macserial -s -n 20 -m iMacPro1,1
     ```
   - 任选一组填入 `EFI/OC/config.plist` → `PlatformInfo/Generic`：
     - `SystemSerialNumber`
     - `MLB`
     - `SystemUUID`（可留空让 OpenCore 自动生成）
2. **准备 EFI 分区**：将本仓库 `EFI` 目录复制到 FAT32 U 盘（或 ESP 分区）根目录，卷标随意。
3. **放入 macOS 恢复镜像**：在 U 盘根目录建 `com.apple.recovery.boot/`，放入 `BaseSystem.dmg` + `BaseSystem.chunklist`（用 macrecovery.py 从 osrecovery.apple.com 获取，default=696-28424 对应 macOS 15.4）。
4. **BIOS 设置**：关闭 Secure Boot / Fast Boot，开启 UEFI USB 引导，从 U 盘启动 OpenCore。
5. 选择 macOS 恢复安装器进入。当前配置已经能显示白苹果和启动进度条，但 Recovery 图形界面与 680M GPU 驱动仍在调试。

## 关键配置

| 项 | 值 | 作用 |
|---|---|---|
| boot-args | `keepsyms=1 -no_compat_check -no_restart` | 当前白苹果里程碑 `ui-lab / ocui1`，不再使用 verbose console relocation |
| Kernel/Block | AppleSMCRTC / IONVMeFamily / AppleRTC | 当前为 USB Recovery 和 RTC 故障隔离启用的诊断 Block |
| Kernel/Quirks | `DisableIoMapper=True` | 必需（AMD 平台） |
| 启用 kext | Lilu / AppleMCEReporterDisabler / VirtualSMC | 当前最小启动组合；GPU 注入未启用 |
| 启用 ACPI | SSDT-EC-USBX | 其余 XHC/APSP/OCMARK 诊断项已关闭 |
| Drivers | OpenRuntime / OpenHfsPlus / ResetNvramEntry | |
| DeviceProperties | 无 GPU 伪装 | 当前保留原始 `1002:1681` 设备身份，避免污染后续原生 X6000 调试 |

## 测试记录（完整版见 `docs/折腾记录.md`）

### 显卡（7 种配置全部卡 DSMOS 后显示初始化）
`-x`（恢复环境不支持）→ `agdpmod=pikera radpg=15` → 删伪装 → 最小化 kext（过 SMCRTC）→ 禁 AppleSMCRTC（过到 DSMOS）→ `-igfxvesa`（无效）→ 伪装+最小化（卡 DSMOS）

### NVMe（4 种配置全 panic → nvme=-1 解决）
双盘（panic）→ 单盘 660p 换槽（panic）→ NVMeFix 重开（panic）→ 换槽+只插 Fanxiang（panic）→ **`nvme=-1`（双盘全过 ✅）**

### 2026-11-10 白苹果里程碑
verbose console relocation 卡死 → 清理 XHC/APSP ACPI 诊断拓扑 → 精确定位 AMD bridge `0:8:1` PFM 写入 → 跳过 bridge configuration → 完成 PCI device tree → 切换 `ui-lab` 非 verbose 启动 → **白苹果和启动进度条出现 ✅**

## 结论

当前不再是“完全无法启动”的状态。OpenCore 已能完成 PCI 设备树并进入 macOS 图形启动界面；剩余问题是 Recovery 图形界面、存储/USB 初始化和 680M 原生 X6000 驱动。NootedRed 仍不支持 Rembrandt RDNA2，`RDNA3Driver` 也不能直接驱动 `1002:1681`，因此目前只能称为 **启动链 WIP**，不能表述为 GPU 加速成功。

本仓库为**全量方案存档**：将来换独显（AMD RX 6000 系）或换机器后，EFI + `nvme=-1` 解法 + 镜像下载工具可直接复用。`docs/config-backups/` 含 36 个分阶段配置备份，可回溯每一步改动。

## 参考

- [Dortania OpenCore 安装指南（AMD 配置）](https://deepwiki.com/dortania/OpenCore-Install-Guide/4.4-amd-configurations)
- [NVMeFix（acidanthera）](https://github.com/acidanthera/NVMeFix)
- [PC711Probe —— 特定 NVMe 的 Lilu 修复插件先例](https://www.insanelymac.com/forum/topic/363054-i-fixed-the-sk-hynix-pc711-kernel-panic-on-macos-12%E2%80%9315-%E2%80%94-pc711probe-is-now-public/)
- [AMD OS X：IONVMeFamily panic 讨论](https://forum.amd-osx.com/threads/cpu-panic-when-boot-with-ionvmefamily-iostoragefamily.4365/)
