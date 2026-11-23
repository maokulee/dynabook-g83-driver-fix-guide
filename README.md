# Dynabook G83 / G83-HU 黑苹果驱动检查与修复实战教程

> 适用机型：Dynabook G83 / G83-HU（Intel i5-1145G7 / Iris Xe，设备 ID `0x9A49`，Tiger Lake）
> 验证环境：macOS Tahoe 26.7.3 (25G342) + OpenCore
> 原始 EFI 配置与机型指引来源：[Kansamu216/dynabook-g83-macos-tahoe](https://github.com/Kansamu216/dynabook-g83-macos-tahoe)（本教程的实践基于该仓库的 EFI 方案，特此致谢）
> 整理日期：2026-11-23

本教程记录了一次完整的驱动体检：从「核显无加速」出发，最终解决核显加速与防睡死服务，同时厘清蓝牙、AirDrop、亮度键三个长期疑难问题的真实状态。**核心结论：EFI 配置本身从一开始就是对的，问题几乎都出在「EFI 之外的用户态组件没装」。**

> ⚠️ **隐私提醒**：本教程所有 `config.plist` 序列号、MLB、ROM、UUID 均为脱敏占位值（`XXXXXXXX`）。你自己的 EFI 里是真实值，**上传/分享前务必脱敏**，切勿把带真实序列号的 `config.plist` 提交到公开仓库。

## 📦 现成 EFI 包下载

完整 EFI 包（已升级 WhateverXe 0.8.2 + AMFIPass 1.4.1 Mod，**三码已脱敏**）可从 Releases 下载：

**[下载 EFI-G83HU-sanitized.zip（58MB）](https://github.com/maokulee/dynabook-g83-driver-fix-guide/releases/download/efi-v1/EFI-G83HU-sanitized.zip)** · [Release 页面](https://github.com/maokulee/dynabook-g83-driver-fix-guide/releases/tag/efi-v1)

> 🔑 **刷入前必做**：包内 `EFI/OC/config.plist` 的 `SystemSerialNumber`、`MLB`、`ROM`、`SystemUUID` 均为占位值。请用 ProperTree / OpenCore Configurator 填回**你自己的真实三码**（从原机 EFI 或 Windows 下获取），否则 iMessage / FaceTime / App Store 等依赖序列号的服务会异常。若你就是用这台 G83 的原始机器，直接从原 EFI 恢复这三码即可。

---

## 一、检查要点总结（先看结论）

| # | 问题 | 根因 | 处理结果 |
|---|------|------|----------|
| 1 | 核显无加速（VRAM 7MB、无 Metal） | **WhateverXe Graphics 用户态 pkg 从未安装**；配套的 kext 版本也偏旧 | ✅ 已修复：升级 kext + 安装 Graphics pkg，现在 VRAM 1536MB / Metal 2 |
| 2 | 假睡眠（合盖睡死风险） | 仓库要求的两个防睡死 launchd 服务完全没装 | ✅ 已修复：服务已装并常驻，`SleepDisabled=1` |
| 3 | 蓝牙「偶尔打不开」 | 底层驱动栈完全正常（kext 全加载、固件已上传、设备已配对） | ✅ 无需改驱动，属系统 UI 缓存的间歇假象 |
| 4 | AirDrop 手机搜不到 | `awdl0` 接口不存在——`AirportItlwm` 不支持 AWDL | ⛔ 驱动层硬限制，当前无解（详见第六节） |
| 5 | 亮度快捷键 Fn+F6/F7 无效 | EC 事件 `_Q01` 执行 `Notify` 时报 `AE_NOT_FOUND`；F12 走的是硬件级 EC 直控，两条路不同 | ⚠️ 保持现状：Fn+F12（调低）+ 系统滑块可用（详见第六节） |

**一句话经验**：这台机器的 EFI 配置是对的。排障时优先怀疑「EFI 之外的用户态驱动 / 服务是否装齐」，而不是反复改 `config.plist`。

---

## 二、EFI 配置要点（已验证正确，勿轻易改动）

### 2.1 关键启动参数

`config.plist` → `NVRAM → Add → 7C436110-AB2A-4BBB-A880-FE41995C9F82`：

```
boot-args = keepsyms=1 watchdog=0 -vi2c-force-polling alcid=11 -allow3d -disablegfxfirmware -wex-no-bcs-tlb amfi_get_out_of_my_way=1
csr-active-config = 03 08 00 00   (即 0x803)
```

核显相关的四个参数一个都不能少：

| 参数 | 作用 |
|------|------|
| `-allow3d` | 允许 3D 加速 |
| `-disablegfxfirmware` | 禁用固件帧缓冲，交给 WhateverXe 接管 |
| `-wex-no-bcs-tlb` | WEX 特定修正 |
| `amfi_get_out_of_my_way=1` | 放宽 AMFI，配合 AMFIPass |

`Misc → Debug → SysReport = False`（未开启 DSDT dump，这也是排查亮度键时拿不到 DSDT 的原因之一）。

### 2.2 Kext 清单（ON 31 / OFF 8）

**必须保持 ON 的核心驱动：**

| 类别 | Kext |
|------|------|
| 底层 | `Lilu` 1.7.2、`VirtualSMC` 1.3.8、`ECEnabler`、`RestrictEvents`、`NoTouchID` |
| 核显 | `WhateverXe` **0.8.2**、`AMFIPass` **1.4.1 Mod 版** |
| 音频 | `AppleALC` 1.9.8、`SimpleHDA` |
| 蓝牙 | `IntelBluetoothFirmware` 2.5.1、`IntelBTPatcher` 2.5.1、`BlueToolFixup` 2.7.3 |
| WiFi | `AirportItlwm` 2.4.0（`MinKernel=25.0.0 / MaxKernel=25.99.99`） |
| 存储/USB | `NVMeFix`、`USBToolBox` + `UTBMap`、`XHCI-unsupported`、`IntelMausi` |
| 输入 | `VoodooI2C`(+3 子 kext)、`VoodooI2CHID`、`VoodooPS2Controller`(+子 kext)、`BrightnessKeys` |
| 电池/传感器 | `SMCBatteryManager`、`SMCProcessor`、`SMCSuperIO` |

**必须保持 OFF 的 8 项（有意为之，勿启用）：**

`WhateverGreen`、`NootedGreen`、`itlwm`、`UTBDefault`、`PC711Probe`、`CpuTopologyRebuild`、`SMCLightSensor`、VoodooPS2 内重复的 `VoodooInput`。

> 🔑 **关键排错点**：`WhateverGreen` / `NootedGreen` 必须 OFF——它们会和 `WhateverXe` 抢核显，同时启用会直接导致加速失效。

### 2.3 ACPI / UEFI

- **SSDT（ON 9）**：`SSDT-ALS0`、`SSDT-SBUS`、`SSDT-PNLF`（亮度必需）、`SSDT-RTCAWAC`、`SSDT-USB-Reset`、`SSDT-USBX`、`SSDT-XOSI`（伪装 Windows 让 EC query 执行）、`SSDT-TPAD`、`SSDT-GPI0-HID`。OFF：`SSDT-GPI0`、`SSDT-PLUG-ALT`。
- **UEFI Drivers（ON 3）**：`OpenRuntime.efi`、`ResetNvramEntry.efi`、`OpenHfsPlus.efi`。

### 2.4 Kext 版本速查（升级后现状）

```
Lilu                    1.7.2
WhateverXe              0.8.2      ← 从 0.7.0 升级
AMFIPass                1.4.1      ← 换成 Mod 版（adhoc 签名）
VirtualSMC              1.3.8
AppleALC                1.9.8
AirportItlwm            2.4.0
IntelBluetoothFirmware  2.5.1
IntelBTPatcher          2.5.1
BlueToolFixup           2.7.3
```

---

## 三、需要安装的软件清单

EFI 里的 kext 只是「内核侧补丁」，很多功能还需要 **用户态组件** 配合。这是本次排障最核心的经验。

| 软件 | 类型 | 版本 | 作用 | 状态 |
|------|------|------|------|------|
| **WhateverXe Graphics** | pkg（用户态驱动） | 0.4.4 | 安装 AppleIntelTGL 图形栈到 `/Library` 并重建 AuxKC，**没有它核显就无加速** | ✅ 已安装 |
| **WhateverXe.kext** | EFI kext | 0.8.2 | 内核侧补丁，与 Graphics 版本必须配套 | ✅ 已升级 |
| **AMFIPass.kext** | EFI kext | 1.4.1 Mod | 绕过 AMFI；官方要求 Mod 版，不能与上游原版共存 | ✅ 已替换 |
| **防睡死服务** | launchd 服务 ×2 + 脚本 | — | 防止合盖睡死 | ✅ 已安装 |

### 3.1 下载链接（官方）

- WhateverXe kext：`https://github.com/b00t0x/WhateverXe/releases/tag/0.8.2`
- WhateverXe Graphics pkg：`https://github.com/b00t0x/WhateverXe-Graphics/releases/tag/v0.4.4`
- AMFIPass 1.4.1 Mod：`https://github.com/b00t0x/WhateverXe-Graphics/tree/main/extras`

> 版本配套很重要：WhateverXe kext 与 Graphics pkg 官方要求成对升级，不要新旧混搭。

---

## 四、安装步骤（完整流程）

### 第 1 步：升级 EFI 里的 kext（WhateverXe + AMFIPass）

挂载 EFI（会弹密码框）：

```bash
sudo diskutil mount disk0s1
# 挂载点：/Volumes/NO NAME
```

先备份原 kext（**关键，出问题可回滚**）：

```bash
cd "/Volumes/NO NAME/EFI/OC/Kexts"
cp -R WhateverXe.kext WhateverXe.kext.bak
cp -R AMFIPass.kext AMFIPass.kext.bak
```

解压下载包并替换（先确认解压出的目录名）：

```bash
cd ~/Downloads
unzip WhateverXe-0.8.2-RELEASE.zip
unzip AMFIPass-v1.4.1-Mod-RELEASE.zip
find . -maxdepth 3 -name "*.kext"

rm -rf "/Volumes/NO NAME/EFI/OC/Kexts/WhateverXe.kext"
cp -R WhateverXe.kext "/Volumes/NO NAME/EFI/OC/Kexts/"
rm -rf "/Volumes/NO NAME/EFI/OC/Kexts/AMFIPass.kext"
cp -R AMFIPass.kext "/Volumes/NO NAME/EFI/OC/Kexts/"
```

校验版本：

```bash
plutil -p "/Volumes/NO NAME/EFI/OC/Kexts/WhateverXe.kext/Contents/Info.plist" | grep CFBundleShortVersionString   # 期望 0.8.2
plutil -p "/Volumes/NO NAME/EFI/OC/Kexts/AMFIPass.kext/Contents/Info.plist" | grep CFBundleShortVersionString    # 期望 1.4.1
codesign -dv "/Volumes/NO NAME/EFI/OC/Kexts/AMFIPass.kext" 2>&1 | grep TeamIdentifier  # Mod 版应无输出（adhoc 签名）
```

> `config.plist` **不需要改**——kext 文件名没变，OpenCore 按 `BundlePath` 加载。

**回滚**：把 `.bak` 目录改回原名即可：

```bash
cd "/Volumes/NO NAME/EFI/OC/Kexts"
mv WhateverXe.kext WhateverXe.kext.new && mv WhateverXe.kext.bak WhateverXe.kext
mv AMFIPass.kext AMFIPass.kext.new && mv AMFIPass.kext.bak AMFIPass.kext
```

### 第 2 步：重启，让新 kext 生效

```bash
sudo diskutil unmount disk0s1
# 重启
```

### 第 3 步：安装 WhateverXe Graphics pkg（核显加速的关键）

进入系统后，双击 `~/Downloads/WhateverXe-Graphics-0.4.4.pkg`，按提示装完，**再重启一次**。

安装内容（可用 `pkgutil` 核对）：

```bash
pkgutil --pkg-info org.b00t0x.WhateverXe.Tahoe | grep -i version   # 期望 0.4.4
```

> 这一步是本次排障最容易被仓库文档遗漏的内容——README/INSTALL/GRAPHICS 只说「WEX 自动匹配」，没写明必须单独装 Graphics pkg。

### 第 4 步：安装防睡死服务

仓库 `services/` 目录提供 3 个文件：`nosleep.sh`、`com.local.keepawake.plist`、`com.local.nosleep.plist`。

用 `install`（不要用 `cp`，某些沙箱/权限环境会拒绝写入系统目录）：

```bash
# 假设三个文件在当前目录
sudo install -m 755 -o root -g wheel nosleep.sh /usr/local/bin/nosleep.sh
sudo install -m 644 -o root -g wheel com.local.keepawake.plist /Library/LaunchDaemons/
sudo install -m 644 -o root -g wheel com.local.nosleep.plist /Library/LaunchDaemons/

sudo launchctl bootstrap system /Library/LaunchDaemons/com.local.keepawake.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.local.nosleep.plist

sudo pmset -a disablesleep 1 sleep 0 hibernatemode 0 standby 0 powernap 0
```

`nosleep.sh` 是一个每 5 秒检查一次并确保 `disablesleep=1` 的守护脚本；`keepawake` 服务则常驻运行 `caffeinate -ims`。

---

## 五、验证清单（装完后逐条核对）

### 5.1 核显加速

```bash
system_profiler SPDisplaysDataType | grep -iE 'Chipset|VRAM|Metal'
# 期望：Intel Iris Xe Graphics / VRAM 1536 MB / Metal Support: Metal 2

ioreg -lw0 -r -c IntelAccelerator | grep -E 'IOGLBundleName|MetalPluginName'
# 期望：AppleIntelTGLGraphicsGLDriver / AppleIntelTGLGraphicsMTLDriver

kmutil showloaded | grep -iE 'WhateverXe|AppleIntelTGL|AMFIPass|Lilu'
# 应看到：Lilu(1.7.2)、WhateverXe(0.8.2)、AMFIPass(1.4.1)、
#         AppleIntelTGLCompat(0.8.1)、AppleIntelTGLGraphicsFramebuffer(16.0.0)、AppleIntelTGLGraphics(16.0.0)
```

> 修复前的异常对照：VRAM 只有 7MB、`MetalPluginName=AGXMetalA12`（软件渲染）、`No Kext Loaded`。出现这些就说明 Graphics pkg 没装或没生效。

### 5.2 防睡死服务

```bash
pmset -g | grep SleepDisabled              # 期望 SleepDisabled 1
sudo launchctl print system/com.local.keepawake | grep -E 'state|pid'
sudo launchctl print system/com.local.nosleep  | grep -E 'state|pid'   # 期望 state = running
pgrep -fl 'caffeinate -ims'                 # 应有进程
pgrep -fl 'nosleep.sh'                      # 应有进程
```

### 5.3 蓝牙

```bash
system_profiler SPBluetoothDataType | grep -iE 'State|Firmware'
kmutil showloaded | grep -iE 'IntelBluetooth|IntelBTPatcher|BlueToolFixup'
# 三个 kext 全加载、Firmware v8787 c8787、State: On 即为正常
```

### 5.4 已加载 kext 与配置一致性

```bash
kmutil showloaded | sed -n '2,$p' | awk '{print $6" "$7}' | grep -viE '^com\.apple\.' | sort -u
```

应列出约 29 个非 Apple kext，与 `config.plist` 里 ON 的项一一对应。`UTBMap.kext` 是端口映射数据，由 USBToolBox 读取，不会独立出现在加载列表，属正常。

---

## 六、已知限制与取舍（不必再折腾）

### 6.1 AirDrop —— 当前驱动做不到

- **现象**：手机搜不到这台电脑，无法互传。
- **根因**：`ifconfig awdl0` 接口不存在。AWDL 是 AirDrop 的底层通道，而 `AirportItlwm` / `itlwm` 系列驱动**都不支持 AWDL**。这是驱动层硬限制，**不是 EFI 配错，改配置解决不了**。
- **可行路径**：换支持 AWDL 的 Broadcom 网卡（原生支持），或等待实验性 AWDL build 成熟（目前仅在 BE200 上验证过，AX 系列未验证）。

### 6.2 亮度快捷键 —— F6/F7 无效，但有两个可用替代

- **现象**：`Fn+F6`/`Fn+F7` 无效；`Fn+F12` 能调低亮度；系统设置里的亮度滑块正常。
- **根因（实测）**：
  - `Fn+F12` 是 **EC 固件写死的硬件级快捷键**——直接改写背光寄存器（`backlight-level` 从 `<7bff>` 跳到 `<1183>`），完全绕过 macOS，所以抓键/日志/系统亮度值都看不到它。
  - `Fn+F6`/`Fn+F7` 走另一条路：EC 触发 ACPI 方法 `\_SB.PC00.LPCB.EC._Q01`，其中 `Notify(0x86)` 报 **`AE_NOT_FOUND`**（目标亮度对象在 macOS DSDT 命名空间里不存在），动作作废。
  - 仓库 README 声称「亮度调节 ✅」，但实际 F6/F7 不工作，属文档与实现有差距。
- **可用手段**：`Fn+F12`（调低）+ 系统亮度滑块。日常足够。
- **若想彻底修**：需 dump 并反编译 DSDT，确认 `_Q01` 的 Notify 目标，写一段 SSDT 热补丁把通知对象改成 macOS 认识的 `PS2K (0x0405/0x0406)` 或 PNLF，再改 `config.plist` 并重启验证。工作量和重启验证成本较高，本教程不做。

### 6.3 蓝牙「时好时坏」

底层驱动完全健康（见 5.3）。若 UI 偶发显示打不开，优先尝试：系统设置里 toggle 一次蓝牙开关、或重置蓝牙模块；不必动 kext。

---

## 七、排障过程中的关键命令速查

```bash
# 挂载/卸载 EFI（弹密码框）
osascript -e 'do shell script "diskutil mount disk0s1" with administrator privileges'
osascript -e 'do shell script "diskutil unmount disk0s1" with administrator privileges'

# 抓内核日志里的 EC / Notify 错误（定位亮度键）
osascript -e 'do shell script "dmesg > /tmp/dmesg.txt; chmod 644 /tmp/dmesg.txt" with administrator privileges'
grep -iE 'EC0|_Q[0-9A-F]|Notify|AE_NOT_FOUND' /tmp/dmesg.txt

# 追踪硬件背光值变化（区分 EC 直控 vs 系统 API）
ioreg -p IODeviceTree -l -w0 | grep -i 'backlight-level'
ioreg -r -c AppleBacklightDisplay -l -w0 | grep -i '"bklt"'

# 判断 AirDrop 可行性
ifconfig awdl0    # 不存在 = AirDrop 不可用
```

---

## 八、版本与状态总账

| 组件 | 修复前 | 修复后 |
|------|--------|--------|
| WhateverXe.kext | 0.7.0 | 0.8.2 |
| AMFIPass.kext | 上游 1.4.1 | 1.4.1 Mod（adhoc） |
| WhateverXe Graphics pkg | 未安装 | 0.4.4 |
| 核显 VRAM / Metal | 7MB / 无 | 1536MB / Metal 2 |
| 防睡死服务 | 未安装 | 2 服务 + 脚本，常驻 |
| SleepDisabled | 缺失 | 1 |
| AirDrop | 不可用 | 不可用（硬限制） |
| 亮度 Fn+F6/F7 | 无效 | 无效（保持现状） |

> 所有改动均**不涉及 `config.plist`**，核显/防睡死相关配置保持仓库原样。
