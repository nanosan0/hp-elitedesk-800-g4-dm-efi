# HP EliteDesk 800 G4 DM · Hackintosh OpenCore EFI

**简体中文** | [English](README.en.md)

HP EliteDesk 800 G4 DM 35W（Q370 芯片组 / i3-8100 / UHD 630）的 OpenCore 引导配置，SMBIOS 机型 `iMac20,1`。这份 EFI 是从这台机器的**实际运行配置**导出的。

> ### ⚠️ 使用前必读
> 1. **SMBIOS 三码（Serial / MLB / SystemUUID / ROM）已全部抹除**，直接开机可以，但 iCloud / iMessage / App Store 登不上，必须先按「[三码](#三码必须自己重做)」生成自己的。
> 2. **核显的 EDID 是硬编码的**（针对下面那台 HKC IG27 显示器）。**换显示器后必须删掉这一项**，否则可能黑屏或分辨率异常 —— 见「[三处必须知道的定制点](#三处必须知道的定制点)」。

---

## 硬件规格（本 EFI 的适配对象）

| 部件 | 实际型号 / 规格 | 相关配置 |
|---|---|---|
| 整机 | **HP EliteDesk 800 G4 DM 35W**（Desktop Mini，Product `83E2`，芯片组 Q370） | SMBIOS 伪装 `iMac20,1` |
| BIOS | Q21 Ver. `02.33.00`（2026-02-03），UEFI，**CSM 关 / Secure Boot 关 / Above 4G 开 / ReBAR 关** | — |
| CPU | **Intel Core i3-8100**（Coffee Lake-S，4 核 4 线程，3.6 GHz，**无睿频**） | `SSDT-PLUG`；`CPUFriend` 两项默认关闭（无睿频，无需重做频率向量） |
| 核显 | **Intel UHD Graphics 630**（`8086:3E91`，动态共享显存 1 GB） | `WhateverGreen`；`AAPL,ig-platform-id = 0x3E9B0007`、`device-id` 伪装为 `0x3E9B` |
| 内存 | **16 GB = 8 GB × 2** DDR4-2667（Crucial `CT8G4SFS8266.M8FD` + `WAR4S2666D19081C`），DIMM1 / DIMM3 双通道，实际跑 2400 | 原生识别，无 `PlatformInfo > Memory` 注入 |
| 硬盘 | **ADATA SX6000LNP 256 GB NVMe**（主控 SM2263，`126F:2263`，挂在 `RP21`） | 原生 NVMe |
| 有线网卡 | **Intel I219-LM**（`8086:15BB`，`_SB.PCI0.GLAN`） | `IntelMausi.kext` |
| 无线网卡 | **Dell Wireless 1820A 802.11ac = Broadcom BCM4350**（`14E4:43A3`，`RP08`） | `AirportBrcmFixup` + `IOSkywalkFamily` + `IO80211FamilyLegacy`（含 `AirPortBrcmNIC`） |
| 蓝牙 | DW1820A 蓝牙 4.1 LE（`0A5C:6412`，USB 挂在 `XHC_.RHUB.HS14`） | `BrcmPatchRAM3` + `BrcmFirmwareData` + `BlueToolFixup` |
| 声卡 | **Synaptics / Conexant HD Audio**（`14F1:5098`），控制器 `8086:A348`（`HDAS`） | `AppleALC.kext`，boot-arg `alcid=20` |
| 显示器 | **HKC IG27**，27 英寸 1920×1080 **@120 Hz**，DisplayPort，EDID 已硬编码注入 | 见「定制点」第 1 条 |
| USB | 300 系列 XHCI（`8086:A36D`），键鼠接在 HS12 的 Hub 上 | `USBInjectAll` + `XHCI-unsupported` |
| 其它 | 嵌入式控制器 `LPCB.EC0`、HPET / RTC / SLPB / PWRB 齐全，AWAC 存在 | `SSDT-AWAC*` + `SSDT-EC-USBX-DESKTOP` + `SSDT-PMC` |

## 关键设置一览

| 项目 | 值 |
|---|---|
| boot-args | `keepsyms=1 debug=0x100 igfxonln=1 amfi=0x80 ipc_control_port_options=0 alcid=20` |
| SIP | `csr-active-config = 0x803`（部分关闭） |
| SecureBootModel | `Disabled` |
| 机型 / 序列号体系 | `iMac20,1`，`UpdateSMBIOSMode = Custom` |
| 生效的 Kernel Quirks | `AppleXcpmCfgLock`（CFG Lock 未解锁）、`CustomSMBIOSGuid`、`DisableIoMapper`（VT-d 保持开启）、`DisableLinkeditJettison`、`XhciPortLimit`、`PanicNoKextDump`、`PowerTimeoutKernelPanic` |
| ACPI 补丁 | `HPET _CRS to XCRS`（改名为 `XCRS`，配合 `SSDT-AWAC-HPET-RTC`） |
| 选择器 | OpenCanopy 图形界面，主题 `Acidanthera\GoldenGate`，超时 5 秒，显示辅助项 |

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── OpenCore.efi                   # OpenCore 1.0.x
    ├── config.plist                   # 本说明对应的配置
    ├── ACPI/
    │   ├── SSDT-AWAC.aml              # 关掉 AWAC，强制用 RTC
    │   ├── SSDT-AWAC-HPET-RTC.aml     # SSDTTime 组合补丁：AWAC + HPET + RTC
    │   ├── SSDT-EC-USBX-DESKTOP.aml   # 桌面假 EC + USB 供电属性
    │   ├── SSDT-PLUG.aml              # CPU 电源管理（plugin-type = 1）
    │   └── SSDT-PMC.aml               # 300 系列原生 NVRAM
    ├── Drivers/
    │   ├── HfsPlus.efi
    │   ├── OpenCanopy.efi
    │   ├── OpenRuntime.efi
    │   └── ResetNvramEntry.efi
    ├── Kexts/                         # 18 个（CPUFriend 两项默认关闭）
    ├── Resources/                     # 选择器主题（GoldenGate）
    └── Tools/
        └── OpenShell.efi
```

## 快速开始

1. U 盘格式化成 **FAT32**，把整个 `EFI` 文件夹拷到 U 盘 EFI 分区的根目录（`EFI/BOOT`、`EFI/OC` 两级结构不能变）。
2. 进 BIOS（开机按 `F10`）确认：**Secure Boot 关、CSM 关、Above 4G Decoding 开、Resizable BAR 关**（本机现状即如此，一般不用改）。
3. U 盘启动 → OpenCore 选择器 → 进 macOS 安装器 → 磁盘工具把目标盘抹成 **APFS / GUID** → 安装。
4. 装完把 `EFI` 拷到**硬盘的 EFI 分区**（双系统就把原 `EFI/Microsoft` 原样留着，只替换 `BOOT` 和 `OC`），BIOS 里把 OpenCore 置顶；或用选择器里的 `UEFI Shell` 执行：
   ```
   bcfg boot add 0 FS0:\EFI\OC\OpenCore.efi "OpenCore"
   ```

## 三码（必须自己重做）

`config.plist` → `PlatformInfo` → `Generic`：

| 键 | 当前占位值 | 该填什么 |
|---|---|---|
| `SystemSerialNumber` | `XXXXXXXXXXXX` | 用 GenSMBIOS / macserial 按 **`iMac20,1`** 生成 |
| `MLB` | `XXXXXXXXXXXXXXXXX` | 同一工具一起生成 |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` | 随机 UUID，或保留全 0 由 OpenCore 按硬件推导 |
| `ROM` | `00 00 00 00 00 00` | **本机 `en0`（I219-LM）的真实 MAC**，6 字节 |

改完重启一次再登录；iMessage 登不上时先试 FaceTime。**不要把自己填好的 config 公开上传**。

## 三处必须知道的定制点

1. **显示器 EDID 是硬编码的** —— `DeviceProperties > PciRoot(0x0)/Pci(0x2,0x0) > AAPL00,override-no-connect` 里塞的是上面那台 HKC IG27 的完整 EDID（128 字节）。
   - 换显示器：**删掉这个键**（保留 `AAPL,ig-platform-id`、`device-id`、`framebuffer-*`），否则可能黑屏、花屏或分辨率/刷新率不对。
   - 同一台显示器但换了接口（DP ↔ HDMI）：一般也要删掉重来。
2. **`alcid=20` 是为这台机器的 Synaptics（`14F1:5098`）试出来的**。Conexant / Synaptics 的 codec 在 AppleALC 里支持面很窄，若无声或只有杂音，按 `11 → 13 → 1 → 3 → 99` 逐个试；实在不行用 USB 声卡。
3. **SMBIOS 用的是 `iMac20,1`（10 代 6 核机型），而真实 CPU 是 4 核 i3-8100**。这是让 `UHD 630 + Q370` 能过机型校验的常规做法，代价是「关于本机」的机型名与 CPU 描述不符、跑分参考值失真。**能不动就别动**：换机型必须连三码一起重做，还可能牵动电源管理与无线行为。

## 已知限制 / 待优化

- **无线网卡是 DW1820A（BCM4350）**，黑苹果圈公认的"坑卡"：要靠 `AirportBrcmFixup` + 这套 legacy 栈才能出 Wi-Fi，稳定性一般（断流、速率低、睡眠后掉线都可能遇到）。想稳，换 **BCM94360CS2 / BCM94360NG** 这类免驱卡最省事。
- **这套无线方案是按 macOS 15 Sequoia 配的**：`IOSkywalkFamily` / `IO80211FamilyLegacy` / `AirPortBrcmNIC` 三条 `MaxKernel = 25.99.99`，并 `Kernel > Block` 屏蔽了 `com.apple.iokit.IOSkywalkFamily`。
  升级 **macOS 26 Tahoe 前**必须换成 `AirPortBrcmNIC-Tahoe.kext`（`MinKernel 25.0.0`），否则升级后 Wi-Fi 直接消失。
- **`SSDT-AWAC.aml` 与 `SSDT-AWAC-HPET-RTC.aml` 功能重叠**（后者已含 AWAC 修复）。现在能用，但若遇到开机偶发「时间倒退 / CMOS 重置 / 唤醒异常」，先把 `SSDT-AWAC` 关掉只留后者。
- **`XhciPortLimit` 在 macOS 11+ 已失效**，`USBInjectAll` 也只是"全注入"老办法。想彻底稳妥，建议用 USBToolBox 生成真正的端口映射，然后把这两个都去掉。
- **`amfi=0x80` 关闭了 AMFI**：副作用是 App 的麦克风 / 摄像头 / 屏幕录制权限**不再弹窗**，隐私列表里也不会出现条目。不需要时连同 `AMFIPass.kext` 一起删掉。
- 本机原本是**双系统（Windows 11 在内置 NVMe 上）**。发布包不含 `EFI/Microsoft` 与 `EFI/HP`（后者是 HP 的 BIOS / ME 固件）；自己配双系统时把原来的 `EFI/Microsoft` 留着即可，OpenCore 会自动识别 Windows 启动项。

## FAQ

**Q：升级 OpenCore 到 1.0.8+ 后 `ocvalidate` 报 `Missing key HideVerbose`？**
A：1.0.8 给 `UEFI > Drivers` 每项加了 `HideVerbose`，四项各补 `false` 即可。旧版本会忽略这个键，补上也安全。

**Q：卡在 `[EB|#LOG:EXITBS:START]`？**
A：检查 `Booter > Quirks`：`EnableWriteUnprotector`、`SetupVirtualMap`、`ProvideCustomSlide` 必须为 `true`（本配置已开）。换 OpenCore 版本时最容易丢这几项。

**Q：亮机就黑屏 / 只出苹果 logo 不进系统？**
A：九成是 EDID 注入和当前显示器不匹配 —— 先删 `AAPL00,override-no-connect` 再试；仍不行就连 `device-id` 伪装（`9b3e0000`）一起删掉，让 macOS 按真实 `0x3E91` 识别。

**Q：睡眠 / 唤醒正常吗？**
A：本配置只做了 `SSDT-PMC`、`SSDT-PLUG`、`HPET/RTC` 这层基础修复，**长时挂机 + 睡眠唤醒未做深度调优**。建议先在「节能」里关掉「断电后自动重启 / Power Nap」，再用 `pmset -g log` 看有没有异常唤醒源。

**Q：能直接装 macOS 26 吗？**
A：能引导，但无线要按上面换 Tahoe 版驱动；另外 26 对机型与 kext 版本更严，建议先在 15 上跑稳再升。

## 更新记录

### 1.0.0
- 首个发布版本：完整 OpenCore EFI（ACPI 5 张、kext 18 个、驱动 4 个、GoldenGate 主题）
- 发布前抹除全部 SMBIOS 三码与 ROM；剔除 `EFI/HP`（HP BIOS / ME 固件）、`EFI/Microsoft`（Windows 引导与 BCD）、`oldConfig.plist`

## 致谢与许可

- 引导与驱动：[Acidanthera](https://github.com/acidanthera) —— OpenCorePkg、Lilu、VirtualSMC、WhateverGreen、AppleALC、AirportBrcmFixup、BrcmPatchRAM、IntelMausi、CPUFriend、AMFIPass
- ACPI 补丁：[CorpNewt SSDTTime](https://github.com/corpnewt/SSDTTime)
- 主题图标：[OcBinaryData](https://github.com/acidanthera/OcBinaryData)
- 硬件信息采集：`sysInfo`（JeoJay127）
- 本仓库的配置与文档以 **MIT** 发布（见 [LICENSE](LICENSE)）；随包二进制（`OpenCore.efi`、kext、主题资源）仍归各自作者与许可证所有。

## 免责声明

折腾黑苹果有风险：可能丢数据、可能刷坏固件。请先备份，并始终保留一只**能开机的备用 U 盘 EFI**。本仓库按「现状」提供，不对任何损坏负责。
