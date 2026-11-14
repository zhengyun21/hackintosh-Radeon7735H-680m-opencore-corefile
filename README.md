# Hackintosh — 机械革命 无界14 2023（R7-7735H / Radeon 680M）

**OpenCore 1.0.8 引导配置 · 完整折腾记录 · 当前状态：Recovery、USB 与内部 NVMe 均已可用，等待安装 macOS**

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

## 成果进展（截至 2026-11-14）

| 项目 | 状态 | 说明 |
|---|---|---|
| **AppleAPIC / CPU 枚举** | ✅ 已通过 | `SSDT-PLUG-ALT-16CPU.aml` 为 16 个 ACPI0007 CPU 补充 macOS 可用的 Processor 对象 |
| **PCI / 设备树** | ✅ 已通过 | `clean-lab / occlean1` 仅保留标准 AMD 补丁 0–24，不再需要全局 bridge/device 返回补丁 |
| **Kernel / 存储栈** | ✅ 已通过 | 已加载 USB、SCSI、IOStorage、APFSMedia，并进入根卷查找阶段 |
| **完整 Recovery** | ✅ 已完成 | `BaseSystem.dmg` 与 chunklist 下载完成，并通过 85 个 Apple chunk |
| **Recovery 启动** | ✅ 已通过 | 成功进入 macOS Sequoia 恢复主菜单 |
| **USB 输入** | ✅ 已修复 | USBToolBox 端口映射 + AppleUSBXHCI interrupter 1→0，鼠标移动已可用 |
| **内部 NVMe** | ✅ 已识别 | 移除 `nvme=-1` 后，Fanxiang S690 1TB 与 Intel 512GB 均显示在磁盘工具 |
| **macOS 安装** | ⏳ 待继续 | Recovery 已可操作，下一步创建 APFS 分区并安装 |
| **GPU / WindowServer** | ⚠️ WIP | `AMDRadeonX6000` / `RDNA3Driver` 均未作为稳定驱动启用 |
| **NVMe 历史解法** | ✅ 已归档 | 旧配置曾用 `nvme=-1` 解决 IONVMeFamily panic |
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
3. **放入完整 macOS 恢复镜像**：在 U 盘根目录建 `com.apple.recovery.boot/`，必须放入 `BaseSystem.dmg` + `BaseSystem.chunklist`。只有 `boot.efi` 和 `BootKernelExtensions.kc` 会进入 `Still waiting for root device.`。
4. **BIOS 设置**：关闭 Secure Boot / Fast Boot，开启 UEFI USB 引导，从 U 盘启动 OpenCore。
5. 选择 `F206 (external) (dmg)` 进入 Recovery。当前配置已通过 AppleAPIC、PCI、存储栈和 USB 输入验证。

## 关键配置

| 项 | 值 | 作用 |
|---|---|---|
| boot-args | `-v keepsyms=1 debug=0x144 -no_compat_check -no_restart kextlog=0xffff ocnvme2` | 当前 Recovery / NVMe 识别里程碑 |
| Kernel/Patch | 标准 AMD 补丁 + 6 个 XHCI interrupter 1→0 | 修复 AMD xHCI 连续输入事件延迟 |
| Kernel/Block | `IONVMeFamily` 不屏蔽 | 允许内部 NVMe 发布 block storage device |
| Kernel/Quirks | `DisableIoMapper=True` | 必需（AMD 平台） |
| 启用 kext | Lilu / AppleMCEReporterDisabler / VirtualSMC / NVMeFix / AmdTscSync / AirportItlwm / USBToolBox / UTBMap | 二进制 kext 未提交，README 记录用途和版本 |
| 启用 ACPI | SSDT-EC-USBX / SSDT-PLUG-ALT-16CPU | 其余 XHC/APSP/OCMARK 诊断项已关闭 |
| Drivers | OpenRuntime / OpenHfsPlus / ResetNvramEntry | |
| DeviceProperties | 无 GPU 伪装 | 当前保留原始 `1002:1681` 设备身份，避免污染后续原生 X6000 调试 |

## 测试记录（完整版见 `docs/折腾记录.md`）

### 显卡（7 种配置全部卡 DSMOS 后显示初始化）
`-x`（恢复环境不支持）→ `agdpmod=pikera radpg=15` → 删伪装 → 最小化 kext（过 SMCRTC）→ 禁 AppleSMCRTC（过到 DSMOS）→ `-igfxvesa`（无效）→ 伪装+最小化（卡 DSMOS）

### NVMe（4 种配置全 panic → nvme=-1 解决）
双盘（panic）→ 单盘 660p 换槽（panic）→ NVMeFix 重开（panic）→ 换槽+只插 Fanxiang（panic）→ **`nvme=-1`（双盘全过 ✅）**

### 2026-11-10 白苹果里程碑
verbose console relocation 卡死 → 清理 XHC/APSP ACPI 诊断拓扑 → 精确定位 AMD bridge `0:8:1` PFM 写入 → 跳过 bridge configuration → 完成 PCI device tree → 切换 `ui-lab` 非 verbose 启动 → **白苹果和启动进度条出现 ✅**

### 2026-11-12 引导链里程碑
发现 DSDT 只有 16 个 `ACPI0007` CPU Device → 增加 `SSDT-PLUG-ALT-16CPU.aml` → 通过 `AppleAPIC` → 关闭 25 号以后的实验性内核补丁 → 通过 PCI、USB、SCSI、IOStorage 和 APFSMedia → 最终停在 `Still waiting for root device.`。这证明引导链已经工作，但当前 USB 缺少完整 `BaseSystem.dmg`，且没有可挂载的 APFS 系统卷。

### 2026-11-13 Recovery 与 USB 输入里程碑
补齐 `BaseSystem.dmg` → 进入 Sequoia Recovery → USBToolBox 映射全部 16 个端口 → 发现 AMD XHCI 仅有一个实际 MSI，而 macOS 普通 TRB 指向 interrupter 1 → 将 6 类 TRB 的 interrupter target 改成 0 → 鼠标连续移动从严重积压恢复到可用。

### 2026-11-14 内部 NVMe 里程碑
磁盘工具仅显示内存镜像 → `kextstat`/`ioreg` 确认 IONVMeFamily 已加载且两个控制器 active → 未发布 block storage device → 移除 `nvme=-1` → 两块内部 NVMe 均被磁盘工具识别。

## 结论

当前 OpenCore、内核、AppleAPIC、PCI、Recovery、USB 输入和内部 NVMe 均已验证。`nvme=-1` 虽解决旧 IONVMe panic，但会阻止当前 Recovery 发布 NVMe 存储设备；`nvme-test / ocnvme2` 是当前工作基线。下一步是在 Fanxiang 约 273 GB 分区上创建 APFS 并安装 macOS，再继续 680M 原生 X6000 驱动。NootedRed 仍不支持 Rembrandt RDNA2，`RDNA3Driver` 也不能直接驱动 `1002:1681`。

本仓库为**全量方案存档**：将来换独显（AMD RX 6000 系）或换机器后，EFI + `nvme=-1` 解法 + 镜像下载工具可直接复用。`docs/config-backups/` 含 36 个分阶段配置备份，可回溯每一步改动。

## 参考

- [Dortania OpenCore 安装指南（AMD 配置）](https://deepwiki.com/dortania/OpenCore-Install-Guide/4.4-amd-configurations)
- [NVMeFix（acidanthera）](https://github.com/acidanthera/NVMeFix)
- [PC711Probe —— 特定 NVMe 的 Lilu 修复插件先例](https://www.insanelymac.com/forum/topic/363054-i-fixed-the-sk-hynix-pc711-kernel-panic-on-macos-12%E2%80%9315-%E2%80%94-pc711probe-is-now-public/)
- [AMD OS X：IONVMeFamily panic 讨论](https://forum.amd-osx.com/threads/cpu-panic-when-boot-with-ionvmefamily-iostoragefamily.4365/)
