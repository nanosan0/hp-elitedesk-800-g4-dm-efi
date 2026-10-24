# HP 400 G 系列 · Hackintosh OpenCore EFI

**简体中文** | [English](README.en.md)

适用于 HP 400 G 系列台式机（Coffee Lake / 300 系列平台）的 OpenCore 引导配置，SMBIOS 机型 `iMac20,1`，核显 UHD 630。

> ### ⚠️ 使用前必读
> 本仓库的 **SMBIOS 三码（序列号 / MLB / SystemUUID / ROM）已全部抹除**。
> 直接拿去用可以开机，但 **iCloud、iMessage、FaceTime、App Store 都无法登录**。
> 请先按「[三码](#三码必须自己重做)」一节生成一套属于你自己的，再写入 `config.plist`。

---

## 配置概览

| 项目 | 内容 |
|---|---|
| SMBIOS 机型 | `iMac20,1` |
| 核显 | Intel UHD 630（`AAPL,ig-platform-id = 0x3E9B0007`、`device-id = 0x3E9B`、`framebuffer-stolenmem = 0x01300000` + 自定义 EDID） |
| 有线网卡 | Intel I219-LM → `IntelMausi.kext` |
| 无线 / 蓝牙 | 博通 BCM94360 系列 → `AirportBrcmFixup` + `BrcmPatchRAM3` + `BrcmFirmwareData` +（`IOSkywalkFamily` / `IO80211FamilyLegacy` / `AirPortBrcmNIC`） |
| 声卡 | `AppleALC.kext`，boot-args `alcid=20` |
| 引导 | OpenCore 1.0.x（随包为原始 `OpenCore.efi`；`config.plist` 与 1.0.8 仅差 4 个 `HideVerbose` 键，见 [FAQ](#faq)） |
| SIP | `csr-active-config = 0x803` |
| SecureBootModel | `Disabled` |
| boot-args | `keepsyms=1 debug=0x100 igfxonln=1 amfi=0x80 ipc_control_port_options=0 alcid=20` |

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── OpenCore.efi
    ├── config.plist
    ├── ACPI/
    │   ├── SSDT-AWAC.aml                 # 强制 RTC，绕开 AWAC
    │   ├── SSDT-AWAC-HPET-RTC.aml        # SSDTTime 组合补丁（AWAC + HPET + RTC）
    │   ├── SSDT-EC-USBX-DESKTOP.aml      # 桌面 EC + USB 供电属性
    │   ├── SSDT-PLUG.aml                 # CPU 电源管理 plugin-type=1
    │   └── SSDT-PMC.aml                  # 300 系列原生 NVRAM
    ├── Drivers/
    │   ├── HfsPlus.efi
    │   ├── OpenCanopy.efi
    │   ├── OpenRuntime.efi
    │   └── ResetNvramEntry.efi
    ├── Kexts/                            # 18 个（CPUFriend 两项默认关闭）
    ├── Resources/                        # OpenCanopy 主题（Acidanthera GoldenGate）
    └── Tools/
        └── OpenShell.efi
```

## 快速开始

1. 准备一只 **FAT32** U 盘，把本仓库的 `EFI` 文件夹整个拷到 U 盘的 EFI 分区根目录（`EFI/BOOT`、`EFI/OC` 两级结构不能变）。
2. 进 BIOS：**关闭 Secure Boot / Fast Boot**，开启 **Above 4G Decoding**，显存（DVMT / VGA Share Memory）设为 **64MB 以上**，启动模式 **UEFI only（不用 CSM）**。
3. U 盘启动 → 进 OpenCore 选择器 → 选 macOS 安装器，先「磁盘工具」把目标盘抹成 **APFS + GUID**，再安装。
4. 装完把 `EFI` 拷到**硬盘的 EFI 分区**，并在 BIOS 启动项里把 OpenCore 置顶（或进选择器 `UEFI Shell` 执行 `bcfg boot add 0 FS0:\EFI\OC\OpenCore.efi "OpenCore"`）。

## 三码（必须自己重做）

`config.plist` → `PlatformInfo` → `Generic` 里这四项目前是占位值：

| 键 | 当前值 | 你该填什么 |
|---|---|---|
| `SystemSerialNumber` | `XXXXXXXXXXXX` | 用 GenSMBIOS / macserial 按 `iMac20,1` 生成 |
| `MLB` | `XXXXXXXXXXXXXXXXX` | 同上（生成时会一起给出） |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` | 随机生成一个 UUID，或留全 0 由 OpenCore 按硬件推导 |
| `ROM` | `00 00 00 00 00 00` | **改成你 `en0`（有线网卡）的真实 MAC**，6 字节，不带分隔符 |

生成后用 ProperTree / OCC 打开 `config.plist` 填进去，**建议重启一次再登录 iCloud**；登录失败先试 FaceTime 再试「信息」。

> 请勿把带自己三码的 `config.plist` 公开上传 —— 一台机器一套码，撞码或频繁换码都会触发 Apple 风控。

## 已知限制与注意

- **博通无线方案是按 macOS 15 Sequoia 配的**：`IOSkywalkFamily` / `IO80211FamilyLegacy` / `AirPortBrcmNIC` 三条的 `MaxKernel` 都是 `25.99.99`，并配合 `Kernel > Block` 屏蔽了 `com.apple.iokit.IOSkywalkFamily`。
  升级到 **macOS 26 Tahoe 之前**，需要换成 `AirPortBrcmNIC-Tahoe.kext`（`MinKernel 25.0.0`），否则升级后无线会直接消失。
- `amfi=0x80` 会关闭 AMFI：**副作用是 App 的隐私权限（麦克风/摄像头/屏幕录制）不再弹窗**，隐私列表里也不会出现条目。不需要时删掉这个 boot-arg 与 `AMFIPass.kext`。
- `iPhone`/`Continuity` 相关功能依赖无线网卡的 `en0` 归属，换过网卡或用过 root patch 后请确认 `en0` 还是无线。
- 本仓库**不包含**原 ESP 里的 `EFI/HP`（HP 固件、ME 固件）与 `EFI/Microsoft`（Windows 引导 + BCD）：前者是厂商固件，后者含你本机的 Windows 启动信息，都不适合公开分发。
- 也不包含 `oldConfig.plist`（旧版配置备份）。

## FAQ

**Q：升级 OpenCore 到 1.0.8 或更新版本后，`ocvalidate` 报 `Missing key HideVerbose`？**
A：1.0.8 给 `UEFI > Drivers` 每项加了 `HideVerbose` 键。在四项里各补 `HideVerbose = false` 即可，旧版本会忽略这个键，所以补上也安全。

**Q：卡在 `[EB|#LOG:EXITBS:START]`？**
A：检查 `Booter > Quirks` 里的 `EnableWriteUnprotector = true`、`SetupVirtualMap = true`、`ProvideCustomSlide = true` 是否还在（本配置已开）。换 OpenCore 版本时最容易漏这几项。

**Q：声卡没声音？**
A：先试 `alcid=20`，不行就换成常见值 `11 / 13 / 1 / 3` 逐个试；HP 的板载 codec 型号不同，layout-id 不一定通用。

**Q：能直接用这份 EFI 装 macOS 26 吗？**
A：能引导，但无线那条要按上面「已知限制」改掉，另外 26 对机型与 kext 版本的要求更严，建议先把系统装好、验证稳定再升级。

## 更新记录

### 1.0.0
- 首个发布版本：完整 OpenCore EFI（ACPI 5 张、kext 18 个、4 个驱动、GoldenGate 主题）
- 发布前已抹除全部 SMBIOS 三码与 ROM，并剔除 `EFI/HP`、`EFI/Microsoft`、`oldConfig.plist`

## 致谢与许可

- 引导与驱动：[Acidanthera](https://github.com/acidanthera)（OpenCorePkg、Lilu、VirtualSMC、WhateverGreen、AppleALC、AirportBrcmFixup、BrcmPatchRAM、IntelMausi、CPUFriend、AMFIPass）
- ACPI 补丁生成：[CorpNewt SSDTTime](https://github.com/corpnewt/SSDTTime)
- 主题图标：[OcBinaryData](https://github.com/acidanthera/OcBinaryData)
- 本仓库的配置文件与说明文档以 **MIT** 许可发布（见 [LICENSE](LICENSE)）；随包分发的 `OpenCore.efi`、kext 等二进制仍归各自作者与许可证所有。

## 免责声明

折腾黑苹果有风险：可能丢数据、可能刷坏固件。请先备份重要数据，并准备一只**能开机的备用 U 盘 EFI**。本仓库按「现状」提供，不对任何损坏负责。
