# ThinkPad Yoga 370 安装 macOS 15 Sequoia（OpenCore）

> **备注：这台的机型标识是 `MacBookPro14,1`，但那只是 OpenCore 的 SMBIOS 伪造。机器实际是 Lenovo ThinkPad Yoga 370**（Kaby Lake，i5-7300U），不是真的 MacBook Pro。第一次看 `system_profiler` / `sw_vers` 的人很容易被这个标识误导。
>
> 本文记录在这台机器上装 macOS 15 的完整路径、三个真正卡住过的坑，以及配套的 OpenCore EFI。
> 仓库里的 `EFI/` 中 config.plist 的三码（序列号 / MLB / SystemUUID / ROM）已替换成占位符。
>
> [English version](#installing-macos-15-sequoia-on-a-thinkpad-yoga-370-opencore) ｜ 中文

---

## 1. 机器与结果

| 项目 | 值 |
| --- | --- |
| 机型 | Lenovo ThinkPad Yoga 370（2017） |
| CPU | Intel Core i5-7300U（Kaby Lake，HD Graphics 620） |
| 内存 | 8GB |
| 硬盘 | 256GB NVMe（PCIe，Samsung OEM） |
| 引导 | OpenCore，SMBIOS 伪装 `MacBookPro14,1` |
| 系统 | macOS 15.8.3（24H241） |
| SIP | 自定义配置（`csr-active-config = 0x0A03`） |

这台机器从来不在 Apple 的支持列表里，所以也不存在「官方最后的支持版本」这个概念——它是黑苹果，靠 OpenCore 伪装 + 第三方驱动跑起来的。

## 2. 这套 EFI 覆盖了什么

`EFI/` 目录就是当时能正常开机的完整备份，结构：

```
EFI/
├── BOOT/BOOTx64.efi
└── OC/
    ├── OpenCore.efi
    ├── config.plist          # 三码已置空，见第 9 节
    ├── ACPI/                 # SSDT-EC-USBX-LAPTOP, SSDT-PNLF, SSDT-PLUG-DRTNIA, SSDT-XOSI
    ├── Kexts/                # 见下方表格
    ├── Drivers/              # OpenRuntime, HfsPlus, OpenCanopy, AudioDxe, ResetNvramEntry
    ├── Tools/                # OpenShell
    └── Resources/            # PickerMode=External，GUI 开机菜单要用到，别删
```

| 硬件 | 驱动 |
| --- | --- |
| 核显 HD 620 | WhateverGreen + `SSDT-PNLF`（背光）+ `SSDT-PLUG-DRTNIA`（CPU 电源管理） |
| 亮度键 / 亮度调节 | BrightnessKeys + `SSDT-PNLF` |
| 声卡 | AppleALC（`alcid=3`） |
| 无线网卡（Intel） | AirportItlwm |
| 蓝牙（Intel） | IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup |
| 有线网卡 | IntelMausi |
| 电池 / 传感器 | SMCBatteryManager + ECEnabler + SMCProcessor + SMCSuperIO + SMCLightSensor |
| 键盘 / 触摸板 | VoodooPS2Controller |
| USB 定制 | USBToolBox + UTBMap |
| NVMe | NVMeFix |
| 其他 | Lilu、VirtualSMC、CpuTscSync、RestrictEvents |

关键 quirk（在 config.plist 里）：

```
Kernel > Quirks: AppleXcpmCfgLock=true, DisableIoMapper=true, PanicNoKextDump=true, XhciPortLimit=true
Booter > Quirks: AvoidRuntimeDefrag=true, EnableWriteUnprotector=true, SetupVirtualMap=true
Misc  > Security: SecureBootModel=Disabled, Vault=Optional, ScanPolicy=0
Misc  > Boot: PickerMode=External, Timeout=5
```

含义：CFG Lock 没在 BIOS 里关，用 `AppleXcpmCfgLock` 绕；VT-d 没关，用 `DisableIoMapper` 绕；USB 端口数用 `XhciPortLimit` 放开（macOS 11+ 其实更推荐自己映射端口，UTBMap 就是这个）。

## 3. 安装流程

1. **BIOS**：关 Secure Boot、关快速启动；CFG Lock / VT-d 懒得关就用上面的 quirk；
2. **做 U 盘**：一个 ≥16GB 的盘，GUID 分区表，一个 FAT32 的 EFI 分区 + 写入 macOS 15 安装器；
3. **放 EFI**：把 `EFI/` 拷到 U 盘的 EFI 分区（先按第 9 节生成自己的三码）；
4. **装系统**：启动菜单选 U 盘的 EFI Boot，进安装器后先用「磁盘工具」把目标盘抹成 APFS / GUID，再装。中途会重启多次，每次都从 U 盘 EFI 进；
5. **装完先别拔 U 盘**：进系统后把 EFI 搬到内置盘（第 4 节），否则拔盘就开不了机。

装完确认版本：

```bash
sw_vers
# ProductVersion: 15.8.3
# BuildVersion:   24H241
```

## 4. 把 EFI 装到内置盘

```bash
# 1) 找分区：内置盘 EFI 通常是 disk0s1，U 盘那个看容量判断
diskutil list

# 2) EFI 分区默认不挂载，手动挂
sudo mkdir -p /tmp/usb_efi /tmp/int_efi
sudo diskutil mount -mountPoint /tmp/usb_efi disk2s1    # U 盘
sudo diskutil mount -mountPoint /tmp/int_efi disk0s1    # 内置盘

# 3) 先备份内置盘原有 EFI（大约 37MB，但这是唯一能原样退回去的东西）
sudo ditto /tmp/int_efi/EFI ~/efi-backup-$(date +%Y%m%d-%H%M%S)/EFI

# 4) 覆盖前先比一下，确认差异是预期的
for f in EFI/OC/OpenCore.efi EFI/OC/config.plist EFI/BOOT/BOOTx64.efi; do
  a=$(md5 -q "/tmp/usb_efi/$f"); b=$(md5 -q "/tmp/int_efi/$f")
  [ "$a" = "$b" ] && echo "一致  $f" || echo "不同  $f"
done

# 5) 覆盖同名文件（ditto 是合并，不会删掉对方独有的目录）
sudo ditto /tmp/usb_efi/EFI /tmp/int_efi/EFI
```

## 5. 坑一：bless 设默认启动项必失败

```bash
sudo bless --mount /tmp/int_efi --setBoot --file /tmp/int_efi/EFI/BOOT/BOOTx64.efi --verbose
```

在这台机器上稳定报错：

```
Setting EFI NVRAM:
    efi-boot-device='<array>...'
Could not set boot device property: 0xe00002e2
```

`--nextonly`、目录方式的 `bless --setBoot` 全一样。原因是 NVRAM 变量由 OpenCore 托管，固件层面写不进去——**不是 EFI 装错了**，也不是权限问题（`sudo` 已经加了）。

绕法：

1. 关机，按住 **Option**（或 F12 / 启动菜单键）开机；
2. 选**内置盘**上的 **EFI Boot**（不是 U 盘那个），进系统；
3. 系统设置 → 通用 → 启动磁盘 → 选 macOS 卷 → 重启。

之后核对：

```bash
nvram -p | grep -a efi-boot-device     # 应该指向内置盘
sudo bless --info /tmp/int_efi
```

能正常开机了再拔 U 盘。

## 6. 坑二：U 盘和内置盘的两份 EFI 不是同一次构建

第 4 节的三处 md5 如果全「不同」，说明两边不是同一次 OpenCore 构建的产物。常见原因：

- 换过 OpenCore 版本，只更新了一边；
- 改完 config.plist 只往一边装；
- 拿别人的 EFI 覆盖，又保留了旧的 BOOTx64.efi。

原则：**以当前能正常开机的那份为准，先备份再覆盖**。判断「是不是同一份」别比版本号字符串，直接比 `OpenCore.efi` 和 `config.plist` 的哈希。覆盖后如果起不来，走第 8 节的回滚，不要靠反复乱试。

## 7. 坑三：boot-args 和 csr-active-config 有两处来源

这是最容易浪费时间的坑。同一个值在两个地方：

- `config.plist` → `NVRAM > Add > 7C436110-... > boot-args`（这份是 `alcid=3 keepsyms=1 swd_panic=1`）；
- 系统里已经存在的 NVRAM 变量（`nvram boot-args` 当时显示 `-v debug=0x100 keepsyms=1 -igfxblr applbkl=1`）。

两者不一致时，你改 plist 可能「没生效」，或者改 NVRAM 重启又被打回去，于是误判成驱动坏了。同样的事也发生在 `csr-active-config` 上：plist 里是 `00000000`，实际系统是 `0x0A03`。

排查顺序固定成三步就不会乱：

```bash
nvram boot-args     # 1. 当前真正生效的是什么
# 2. 再看 config.plist 的 NVRAM > Add 里写的是什么
# 3. 决定改哪一边：要长期生效就改 plist，并确认它在 Delete 列表里不被残留值挡住
```

补充：

- `-v debug=0x100 keepsyms=1` 只在排障期需要，能看到卡在哪一行；
- `-igfxblr` 是背光寄存器相关参数，去掉容易黑屏或亮度不可调；
- `alcid=3` 决定声卡 layout-id，换值前先看 AppleALC 的机型列表。

## 8. 回滚与维护

回滚，从轻到重：

1. 换启动项：启动菜单选 U 盘 EFI，或系统设置里把启动磁盘改回去；
2. 还原 EFI：`sudo ditto ~/efi-backup-<时间戳>/EFI /tmp/int_efi/EFI`，再重复第 5 节的启动磁盘设置；
3. 重装：U 盘安装器重来。

维护：

- **系统小版本升级前先备份**。黑苹果升小版本最常见的后果是网卡/蓝牙失效（AirportItlwm 与系统版本强绑定），升级前先确认手里有新版本 kext；
- 每次改 config.plist 都用 `ocvalidate` 过一遍，改错一个 key 就进不了系统；
- EFI 备份按时间戳留最近两三份，方便对比；
- 换硬件（网卡、硬盘）后 USB 端口映射和 NVMe 相关 kext 要重新确认。

## 9. 用这份 EFI 之前必须做的事

**不要直接用仓库里的 config.plist 开机。** 三码已经被置空：

```
PlatformInfo > Generic:
  SystemSerialNumber = REPLACE-WITH-YOUR-SERIAL
  MLB                = REPLACE-WITH-YOUR-MLB
  SystemUUID         = 00000000-0000-0000-0000-000000000000
  ROM                = AAAAAAAA
```

用 GenSMBIOS 生成属于自己的一组（`MacBookPro14,1`），填进去再引导。三码相同意味着两台机器在 Apple 那边是同一台设备，会互相顶掉 iMessage / iCloud 的登录。

另外两点：

- 机型不同（不是 Yoga 370）时，ACPI 里的 `SSDT-EC-USBX-LAPTOP`、`SSDT-PNLF`、`SSDT-XOSI` 和 USB 映射（UTBMap）基本都得重做，别直接套；
- OpenCore 版本要和 config.plist 的 schema 对得上，跨大版本升级请先读 OpenCore 的 Configuration.pdf。

## 10. 脱敏说明

仓库里没有：设备序列号、主板序列号（MLB）、SystemUUID、ROM、主机名、局域网 IP、账户名、`ioreg` 全量输出、`nvram` 带 UUID 的完整输出。上面的命令都能直接照抄，不需要这些值。

---

# Installing macOS 15 Sequoia on a ThinkPad Yoga 370 (OpenCore)

> **Note: the model identifier shown is `MacBookPro14,1`, but that's just an OpenCore SMBIOS spoof. The machine is a Lenovo ThinkPad Yoga 370** (Kaby Lake, i5-7300U) — not a real MacBook Pro. This is the first thing that misleads people reading `sw_vers` or `system_profiler` output.
>
> This is a write-up of getting macOS 15 running on it: the path that worked, the three places it actually stopped me, and the OpenCore EFI to go with it.
> The SMBIOS values in `EFI/OC/config.plist` (serial / MLB / SystemUUID / ROM) are placeholders.
>
> English ｜ [中文版](#thinkpad-yoga-370-安装-macos-15-sequoiaopencore)

---

## 1. The machine and the result

| Item | Value |
| --- | --- |
| Model | Lenovo ThinkPad Yoga 370 (2017) |
| CPU | Intel Core i5-7300U (Kaby Lake, HD Graphics 620) |
| RAM | 8GB |
| Storage | 256GB NVMe (PCIe, Samsung OEM) |
| Bootloader | OpenCore, SMBIOS spoofed as `MacBookPro14,1` |
| OS | macOS 15.8.3 (24H241) |
| SIP | Custom configuration (`csr-active-config = 0x0A03`) |

This machine was never on Apple's support list, so there is no "last officially supported release" to speak of — it's a hackintosh, running on an OpenCore spoof plus third-party drivers.

## 2. What the EFI covers

`EFI/` is the complete backup of the EFI that booted at the time:

```
EFI/
├── BOOT/BOOTx64.efi
└── OC/
    ├── OpenCore.efi
    ├── config.plist          # SMBIOS values blanked, see section 9
    ├── ACPI/                 # SSDT-EC-USBX-LAPTOP, SSDT-PNLF, SSDT-PLUG-DRTNIA, SSDT-XOSI
    ├── Kexts/                # see table below
    ├── Drivers/              # OpenRuntime, HfsPlus, OpenCanopy, AudioDxe, ResetNvramEntry
    ├── Tools/                # OpenShell
    └── Resources/            # required: PickerMode=External (GUI boot picker)
```

| Hardware | Driver |
| --- | --- |
| HD 620 graphics | WhateverGreen + `SSDT-PNLF` (backlight) + `SSDT-PLUG-DRTNIA` (CPU power management) |
| Brightness keys | BrightnessKeys + `SSDT-PNLF` |
| Audio | AppleALC (`alcid=3`) |
| Wi-Fi (Intel) | AirportItlwm |
| Bluetooth (Intel) | IntelBluetoothFirmware + IntelBTPatcher + BlueToolFixup |
| Ethernet | IntelMausi |
| Battery / sensors | SMCBatteryManager + ECEnabler + SMCProcessor + SMCSuperIO + SMCLightSensor |
| Keyboard / trackpad | VoodooPS2Controller |
| USB | USBToolBox + UTBMap |
| NVMe | NVMeFix |
| Misc | Lilu, VirtualSMC, CpuTscSync, RestrictEvents |

Key quirks in config.plist:

```
Kernel > Quirks: AppleXcpmCfgLock=true, DisableIoMapper=true, PanicNoKextDump=true, XhciPortLimit=true
Booter > Quirks: AvoidRuntimeDefrag=true, EnableWriteUnprotector=true, SetupVirtualMap=true
Misc  > Security: SecureBootModel=Disabled, Vault=Optional, ScanPolicy=0
Misc  > Boot: PickerMode=External, Timeout=5
```

In plain terms: CFG Lock isn't disabled in BIOS, so `AppleXcpmCfgLock` works around it; VT-d isn't disabled, so `DisableIoMapper` does; `XhciPortLimit` lifts the USB port cap (on macOS 11+ mapping ports yourself is cleaner — that's what UTBMap is).

## 3. Install steps

1. **BIOS**: turn off Secure Boot and Fast Boot. Leave CFG Lock / VT-d alone and use the quirks above;
2. **USB drive**: ≥16GB, GUID partition map, one FAT32 EFI partition plus the macOS 15 installer;
3. **EFI**: copy `EFI/` to the USB drive's EFI partition (generate your own SMBIOS first — section 9);
4. **Install**: boot the USB's EFI Boot entry, erase the target disk as APFS / GUID in Disk Utility, install. It reboots several times; boot from the USB EFI each time;
5. **Don't unplug yet**: after the first boot, move the EFI to the internal disk (section 4) or the machine won't boot without the USB drive.

Confirm the version:

```bash
sw_vers
# ProductVersion: 15.8.3
# BuildVersion:   24H241
```

## 4. Moving the EFI to the internal disk

```bash
# 1) Locate the partitions: internal EFI is usually disk0s1, identify the USB one by size
diskutil list

# 2) EFI partitions aren't mounted by default
sudo mkdir -p /tmp/usb_efi /tmp/int_efi
sudo diskutil mount -mountPoint /tmp/usb_efi disk2s1    # USB
sudo diskutil mount -mountPoint /tmp/int_efi disk0s1    # internal

# 3) Back up the existing internal EFI (~37MB, but it's the only way back)
sudo ditto /tmp/int_efi/EFI ~/efi-backup-$(date +%Y%m%d-%H%M%S)/EFI

# 4) Compare before overwriting, confirm the difference is expected
for f in EFI/OC/OpenCore.efi EFI/OC/config.plist EFI/BOOT/BOOTx64.efi; do
  a=$(md5 -q "/tmp/usb_efi/$f"); b=$(md5 -q "/tmp/int_efi/$f")
  [ "$a" = "$b" ] && echo "same  $f" || echo "diff  $f"
done

# 5) Overwrite same-named files (ditto merges; it won't delete dirs only the other side has)
sudo ditto /tmp/usb_efi/EFI /tmp/int_efi/EFI
```

## 5. Pitfall 1: bless can't set the default boot entry

```bash
sudo bless --mount /tmp/int_efi --setBoot --file /tmp/int_efi/EFI/BOOT/BOOTx64.efi --verbose
```

Fails consistently on this machine:

```
Setting EFI NVRAM:
    efi-boot-device='<array>...'
Could not set boot device property: 0xe00002e2
```

`--nextonly` and the directory form of `bless --setBoot` fail identically. The cause is that NVRAM variables are managed by OpenCore and can't be written at the firmware level — **the EFI is not broken**, and it isn't a permissions problem (it's already under `sudo`).

Workaround:

1. Shut down, hold **Option** (or F12 / the boot menu key) while powering on;
2. Pick **EFI Boot** on the **internal** disk (not the USB one) and boot;
3. System Settings → General → Startup Disk → select the macOS volume → Restart.

Verify:

```bash
nvram -p | grep -a efi-boot-device     # should point at the internal disk
sudo bless --info /tmp/int_efi
```

Only unplug the USB drive once it boots on its own.

## 6. Pitfall 2: the two EFIs are from different builds

If all three md5 comparisons in section 4 say "diff", the two sides aren't from the same OpenCore build. Typical causes:

- A new OpenCore version installed to only one side;
- Editing config.plist and installing it to only one side;
- Copying someone else's EFI over yours but keeping the old BOOTx64.efi.

Rule: **take the copy that currently boots, back it up, then overwrite.** Don't compare version strings to decide whether two EFIs match — hash `OpenCore.efi` and `config.plist` instead. If it then won't boot, roll back as in section 8 rather than trying combinations.

## 7. Pitfall 3: boot-args and csr-active-config live in two places

This one burns the most time. The same value exists twice:

- `config.plist` → `NVRAM > Add > 7C436110-... > boot-args` (this backup has `alcid=3 keepsyms=1 swd_panic=1`);
- the NVRAM variable already present in the system (`nvram boot-args` showed `-v debug=0x100 keepsyms=1 -igfxblr applbkl=1` at the time).

When the two disagree, editing the plist may appear to "do nothing", or the NVRAM value may be reset on reboot — and you end up blaming a driver. `csr-active-config` has the same problem: `00000000` in the plist, `0x0A03` on the live system.

Check in this order:

```bash
nvram boot-args     # 1. what's actually in effect
# 2. then what the plist says under NVRAM > Add
# 3. decide which one to change — for a permanent change edit the plist and make
#    sure a stale NVRAM value isn't shadowing it
```

Notes:

- `-v debug=0x100 keepsyms=1` is for debugging only — verbose boot shows where it hangs;
- `-igfxblr` relates to backlight registers; without it you get a black screen or no brightness control;
- `alcid=3` is the audio layout-id; check AppleALC's list before changing it.

## 8. Rollback and maintenance

Rollback, least destructive first:

1. Boot the other entry: select the USB EFI in the boot menu, or change the startup disk back;
2. Restore the EFI: `sudo ditto ~/efi-backup-<timestamp>/EFI /tmp/int_efi/EFI`, then redo the startup-disk step in section 5;
3. Reinstall from the USB installer.

Maintenance:

- **Back up before any macOS point upgrade.** The most common fallout on a hackintosh is Wi-Fi/Bluetooth dying (AirportItlwm is tightly coupled to the OS version) — have the matching kext in hand first;
- Run `ocvalidate` after every config.plist edit; one wrong key means no boot;
- Keep the two or three most recent timestamped EFI backups for diffing;
- After swapping hardware (Wi-Fi card, SSD), re-check the USB map and the NVMe-related kexts.

## 9. Before you use this EFI

**Do not boot with the config.plist from this repo as-is.** The SMBIOS values are blanked:

```
PlatformInfo > Generic:
  SystemSerialNumber = REPLACE-WITH-YOUR-SERIAL
  MLB                = REPLACE-WITH-YOUR-MLB
  SystemUUID         = 00000000-0000-0000-0000-000000000000
  ROM                = AAAAAAAA
```

Generate your own set for `MacBookPro14,1` with GenSMBIOS and fill them in. Identical triples mean two machines look like the same device to Apple, and they will keep kicking each other off iMessage / iCloud.

Two more caveats:

- On a different model, the ACPI files (`SSDT-EC-USBX-LAPTOP`, `SSDT-PNLF`, `SSDT-XOSI`) and the USB map (UTBMap) almost certainly need redoing — don't drop this in unchanged;
- The OpenCore version must match the config.plist schema; read OpenCore's Configuration.pdf before a major upgrade.

## 10. Redaction

This repo contains no device serial, board serial (MLB), SystemUUID, ROM, hostname, LAN IP, account name, full `ioreg` dump, or `nvram` output with UUIDs. Every command above works as-is without those values.

---

*记录时间 / Written: 2026-11*
