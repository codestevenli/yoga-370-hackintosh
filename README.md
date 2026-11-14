# 在 2017 款 MacBook Pro 上安装 macOS 15 Sequoia

> 本文记录一台 MacBook Pro 13" 2017（\`MacBookPro14,1\`，i5-7300U / 8GB / 256GB NVMe）从官方支持的最后一个版本升级到 macOS 15 的完整过程，以及过程中真正卡住人的几处坑。
> 文中所有命令都在本机实跑过；不含任何序列号、硬件 UUID、磁盘 UUID、账户名等信息。
>
> [English version](#installing-macos-15-sequoia-on-a-2017-macbook-pro) ｜ 中文

---

## 1. 背景

\`MacBookPro14,1\` 官方支持到 macOS 13 Ventura 为止，14 Sonoma 起就不再在支持列表里。要装 15 Sequoia，只能走 **OpenCore Legacy Patcher（OCLP）**：

- 用 OpenCore 引导器把「这台机器不在支持列表」这件事在固件层面绕过去；
- 用 root patch（打补丁的内核扩展）补回官方驱动已经不再提供的硬件支持，例如 Kaby Lake 的核显与背光。

实测结果：系统信息显示 \`macOS 15.8.3 (24H241)\`，SIP 处于 OCLP 需要的「自定义配置」状态。

## 2. 准备

| 项目 | 说明 |
| --- | --- |
| 一个 ≥16GB 的 U 盘 | 会被整盘擦除，先备份里面的东西 |
| OCLP 最新版 | 用最新 release，不要用一两年前的旧版本拼装 EFI |
| 数据备份 | 升级前先 Time Machine / 手动拷走重要文件，这一步不能省 |
| 电源 | 全程插电，安装中断电基本等于重来 |
| 另一个能上网的设备 | 出问题时用来查资料、下载系统 |

## 3. 制作启动盘

在 OCLP 里选 **Create macOS Installer**，让它下载并写入安装器到 U 盘。这一步比较慢（下载几个 G + 写入），中途不要拔盘。

写完后 U 盘里会出现一个 \`Install macOS Sequoia\` 卷，这是安装源，**不是** EFI。

## 4. 生成并写入 OpenCore EFI

同一份 OCLP 里选 **Build and Install OpenCore**，目标选 U 盘的 EFI 分区。OCLP 会按当前机型的型号标识生成 \`EFI/OC/config.plist\` 和 \`EFI/BOOT/BOOTx64.efi\`。

判断 EFI 是否可用的最小标准：

\`\`\`bash
ls /Volumes/EFI/EFI/OC/OpenCore.efi     # 必须有
ls /Volumes/EFI/EFI/BOOT/BOOTx64.efi    # 必须有
\`\`\`

注意：**U 盘 EFI 和内置盘 EFI 的 OpenCore 版本必须一致**。我这次踩的坑就是两边不是同一次 OCLP 生成的，\`OpenCore.efi\`、\`config.plist\`、\`BOOTx64.efi\` 三个文件的 md5 全对不上（见第 7 节）。

## 5. 安装系统

1. 关机，按住 <kbd>Option</kbd>(⌥) 开机，选 **EFI Boot**（带 OpenCore 图标那个），不要直接选安装盘；
2. 进入 OpenCore 菜单后选 \`Install macOS Sequoia\`；
3. 进安装器后先用「磁盘工具」把内置盘抹掉（APFS / GUID 分区表），再装；
4. 安装过程中会自动重启多次，**每次都要按住 Option 选 EFI Boot 再选 macOS Installer 那个卷**，不要让它自己走默认启动项——此时内置盘的 EFI 还没装 OpenCore，直接默认启动会黑屏停在禁止符号或苹果标志。

装完进系统后，先确认版本：

\`\`\`bash
sw_vers
# ProductVersion: 15.8.3
# BuildVersion:   24H241
\`\`\`

## 6. 把 EFI 装到内置盘

系统装好后，U 盘还插着，此时要做的是把 U 盘上的 OpenCore 搬到内置盘的 EFI 分区，否则拔了 U 盘就开不了机。

\`\`\`bash
# 1) 找到内置盘和外接盘的 EFI 分区
diskutil list

# 2) 都要挂载（EFI 分区默认不挂载）
sudo mkdir -p /tmp/usb_efi /tmp/int_efi
sudo diskutil mount -mountPoint /tmp/usb_efi disk2s1   # U 盘 EFI
sudo diskutil mount -mountPoint /tmp/int_efi disk0s1   # 内置盘 EFI

# 3) 先备份内置盘现有 EFI，出问题能回滚
sudo ditto /tmp/int_efi/EFI ~/efi-backup-$(date +%Y%m%d-%H%M%S)/EFI

# 4) 对比两边内容，确认差异是预期的
for f in EFI/OC/OpenCore.efi EFI/OC/config.plist EFI/BOOT/BOOTx64.efi; do
  a=$(md5 -q "/tmp/usb_efi/$f"); b=$(md5 -q "/tmp/int_efi/$f")
  [ "$a" = "$b" ] && echo "一致  $f" || echo "不同  $f"
done

# 5) 用 U 盘版本覆盖（只覆盖同名文件，EFI/APPLE 之类原有目录不动）
sudo ditto /tmp/usb_efi/EFI /tmp/int_efi/EFI
\`\`\`

备份目录大概 37MB，很小，但它是唯一能让你「不改任何东西退回原状」的东西，务必保留。

## 7. 坑一：bless 设默认启动项会失败

把 EFI 拷过去之后还要告诉固件「默认从内置盘 EFI 启动」：

\`\`\`bash
sudo bless --mount /tmp/int_efi --setBoot --file /tmp/int_efi/EFI/BOOT/BOOTx64.efi --verbose
\`\`\`

在这台机器上它稳定失败：

\`\`\`
Setting EFI NVRAM:
    efi-boot-device='<array>...'
Could not set boot device property: 0xe00002e2
\`\`\`

\`--nextonly\`、目录方式 \`bless --setBoot\` 也一样。原因大概是 OCLP 已经把 NVRAM 保护/固件变量这块按非常规方式处理了，命令行层面写不进去。**这不是 EFI 装错了**，只是写默认启动项这一步得换路子。

绕过办法：

1. 关机，按住 <kbd>Option</kbd> 开机；
2. 选内置硬盘上的 **EFI Boot**（不是 U 盘那个），进系统；
3. 系统设置 → 通用 → 启动磁盘 → 选 macOS 卷，解锁后点「重新启动」。

之后确认一下，\`efi-boot-device\` 应该指向内置盘：

\`\`\`bash
nvram -p | grep -a efi-boot-device
\`\`\`

也可以顺便看看有没有真正的默认项：

\`\`\`bash
sudo bless --info /tmp/int_efi
\`\`\`

这一步做完再拔 U 盘，重启应该能直接进系统。

## 8. 坑二：两边 EFI 版本不一致

第 6 节的 md5 对比如果三个文件全「不同」，说明 U 盘上的 EFI 和内置盘上的 EFI 不是同一份 OCLP 构建产物。常见原因：

- 中途换过 OCLP 版本（比如先用旧版做了 U 盘，又用新版往内置盘装了一遍）；
- OCLP 升级后重新 Build 过一次 OpenCore，但只装到了其中一边。

处理原则：**以能正常开机的那一份为准，先备份，再覆盖**。覆盖后如果启动卡住，用第 10 节的方法回滚，而不是反复乱试。

判断「是不是同一次构建」不要看版本号字符串，直接比 \`OpenCore.efi\` 和 \`config.plist\` 的哈希最省事。

## 9. 坑三：root patch 与启动参数

系统进得去但亮度调不了 / 显卡没加速，是 OCLP 的 root patch 还没打。在 OCLP 里选 **Post-Install Root Patch**，打完必须重启。

这台机器上生效的关键设置：

\`\`\`bash
# 启动参数
nvram boot-args
# -v debug=0x100 keepsyms=1 -igfxblr applbkl=1

# SIP 状态（OCLP 要求的自定义配置，FileVault/系统保护部分被关掉是正常的）
csrutil status
# System Integrity Protection status: unknown (Custom Configuration).
\`\`\`

- \`-v debug=0x100 keepsyms=1\`：调试期保留，开机走文字模式，卡住时能看到卡在哪一行；稳定后可以去掉 \`-v\`；
- \`-igfxblr applbkl=1\`：Kaby Lake 的背光相关参数，去掉会黑屏或亮度不可调。

改启动参数用 OCLP 的图形界面最稳，手写 \`nvram boot-args\` 容易把 \`keepsyms\` 这类参数写丢。

## 10. 回滚

三条路，按破坏性从小到大：

1. **换回原启动项**：Option 开机选 U 盘 EFI，或者系统设置里把启动磁盘改回原卷；
2. **还原 EFI**：把第 6 节备份的 \`EFI\` 目录 \`ditto\` 回去（\`sudo ditto ~/efi-backup-<时间戳>/EFI /tmp/int_efi/EFI\`），然后重新走第 7 节的启动磁盘设置；
3. **整机重装**：U 盘重装，或 Time Machine 恢复。

## 11. 日常维护要点

- **系统小版本升级**（15.x → 15.y）之后，root patch 通常要重打一遍，否则显卡/背光会掉；
- **每次升级 OCLP 后重新 Build OpenCore，并且 U 盘 EFI 与内置盘 EFI 一起更新**，别只更一边；
- EFI 备份目录建议按时间戳留着最近两三份，出问题能对比；
- 别开 FileVault 再去折腾 EFI，会多一层恢复上的麻烦。

## 12. 不要放进仓库的东西

写这类记录时顺手把下面这些脱敏掉：设备序列号、硬件 UUID、磁盘/分区 UUID、\`nvram\` 里带 UUID 的那几行完整输出、主机名、局域网 IP、Apple ID / 用户名、以及任何 \`ioreg\` 全量输出。本文所有命令都能直接照抄，不需要这些值。

---

# Installing macOS 15 Sequoia on a 2017 MacBook Pro

> This is a write-up of upgrading a MacBook Pro 13" 2017 (\`MacBookPro14,1\`, i5-7300U / 8GB / 256GB NVMe) from its last officially supported release to macOS 15, including the three places where it actually stops you.
> Every command below was run on this machine. No serial numbers, hardware UUIDs, disk UUIDs, or account names are included.
>
> English ｜ [中文版](#在-2017-款-macbook-pro-上安装-macos-15-sequoia)

---

## 1. Background

\`MacBookPro14,1\` is officially supported up to macOS 13 Ventura; support was dropped with 14 Sonoma. To get 15 Sequoia you need **OpenCore Legacy Patcher (OCLP)**:

- An OpenCore bootloader makes the firmware believe the machine is supported;
- Root patches (patched kernel extensions) restore hardware support Apple no longer ships, e.g. Kaby Lake graphics and backlight.

Result on this machine: \`macOS 15.8.3 (24H241)\`, with SIP in the "custom configuration" state OCLP requires.

## 2. Prerequisites

| Item | Notes |
| --- | --- |
| USB drive ≥16GB | It will be wiped entirely — back up its contents first |
| Latest OCLP release | Use the newest release, don't stitch an EFI together from an old build |
| A data backup | Time Machine or a manual copy of important files. Non-negotiable |
| Power | Keep it plugged in; losing power mid-install basically means starting over |
| A second device with internet | For looking things up when it goes wrong |

## 3. Create the installer

In OCLP choose **Create macOS Installer** and let it download and write the installer to the USB drive. This is slow (several GB download plus writing) — don't unplug mid-way.

When it finishes there's an \`Install macOS Sequoia\` volume on the USB drive. That's the installer source, **not** the EFI.

## 4. Build and install the OpenCore EFI

In the same OCLP build, choose **Build and Install OpenCore** and target the USB drive's EFI partition. OCLP generates \`EFI/OC/config.plist\` and \`EFI/BOOT/BOOTx64.efi\` for the current model identifier.

Minimum sanity check for a usable EFI:

\`\`\`bash
ls /Volumes/EFI/EFI/OC/OpenCore.efi     # required
ls /Volumes/EFI/EFI/BOOT/BOOTx64.efi    # required
\`\`\`

Important: **the USB EFI and the internal disk EFI must come from the same OCLP build.** My mistake was having two different builds on the two sides — \`OpenCore.efi\`, \`config.plist\` and \`BOOTx64.efi\` all had different md5 hashes (section 7).

## 5. Install macOS

1. Shut down, hold <kbd>Option</kbd>(⌥) while booting, pick **EFI Boot** (the OpenCore entry) — not the installer volume directly;
2. In the OpenCore menu choose \`Install macOS Sequoia\`;
3. In the installer, erase the internal disk with Disk Utility (APFS / GUID partition map), then install;
4. The installer reboots several times. **Every time, hold Option and pick EFI Boot, then the macOS Installer volume.** Don't let it take the default boot entry — the internal disk has no OpenCore yet at that point, and the default path ends on a prohibitory sign or a stuck Apple logo.

After it boots, confirm the version:

\`\`\`bash
sw_vers
# ProductVersion: 15.8.3
# BuildVersion:   24H241
\`\`\`

## 6. Move the EFI onto the internal disk

With the system installed and the USB drive still plugged in, OpenCore has to be moved to the internal disk's EFI partition — otherwise the machine won't boot once the USB drive is gone.

\`\`\`bash
# 1) Identify the internal and external EFI partitions
diskutil list

# 2) Mount both (EFI partitions are not mounted by default)
sudo mkdir -p /tmp/usb_efi /tmp/int_efi
sudo diskutil mount -mountPoint /tmp/usb_efi disk2s1   # USB EFI
sudo diskutil mount -mountPoint /tmp/int_efi disk0s1   # internal EFI

# 3) Back up the existing internal EFI first, so you can roll back
sudo ditto /tmp/int_efi/EFI ~/efi-backup-$(date +%Y%m%d-%H%M%S)/EFI

# 4) Compare the two sides and confirm the difference is expected
for f in EFI/OC/OpenCore.efi EFI/OC/config.plist EFI/BOOT/BOOTx64.efi; do
  a=$(md5 -q "/tmp/usb_efi/$f"); b=$(md5 -q "/tmp/int_efi/$f")
  [ "$a" = "$b" ] && echo "same  $f" || echo "diff  $f"
done

# 5) Overwrite with the USB version (only same-named files; existing dirs like EFI/APPLE are untouched)
sudo ditto /tmp/usb_efi/EFI /tmp/int_efi/EFI
\`\`\`

The backup is about 37MB. Tiny — but it's the only thing that lets you return to the previous state without changing anything, so keep it.

## 7. Pitfall 1: bless cannot set the default boot entry

After copying, the firmware still has to be told to boot from the internal EFI:

\`\`\`bash
sudo bless --mount /tmp/int_efi --setBoot --file /tmp/int_efi/EFI/BOOT/BOOTx64.efi --verbose
\`\`\`

On this machine it fails every time:

\`\`\`
Setting EFI NVRAM:
    efi-boot-device='<array>...'
Could not set boot device property: 0xe00002e2
\`\`\`

\`--nextonly\` and the directory form of \`bless --setBoot\` fail the same way. Presumably OCLP handles NVRAM protection / firmware variables in a non-standard way, so the write doesn't go through at the command-line level. **The EFI is not broken** — only this default-boot-entry step needs a different route.

Workaround:

1. Shut down, hold <kbd>Option</kbd> while booting;
2. Pick **EFI Boot** on the internal disk (not the USB one) and boot into macOS;
3. System Settings → General → Startup Disk → select the macOS volume, unlock, click "Restart".

Then verify that \`efi-boot-device\` points at the internal disk:

\`\`\`bash
nvram -p | grep -a efi-boot-device
\`\`\`

And check the reported default:

\`\`\`bash
sudo bless --info /tmp/int_efi
\`\`\`

Only unplug the USB drive after this; a restart should now go straight into macOS.

## 8. Pitfall 2: the two EFIs are from different builds

If all three md5 comparisons in section 6 say "diff", the USB EFI and the internal EFI are not from the same OCLP build. Usual causes:

- Switching OCLP versions part-way (old build on the USB, new build installed to the internal disk);
- Rebuilding OpenCore after an OCLP upgrade but only installing it to one side.

Rule: **take the copy that currently boots, back it up, then overwrite.** If it then fails to boot, roll back as in section 10 instead of trying random combinations.

To decide whether two EFIs are "the same build", don't compare version strings — hash \`OpenCore.efi\` and \`config.plist\`.

## 9. Pitfall 3: root patches and boot arguments

If the system boots but brightness doesn't change and graphics feel unaccelerated, the OCLP root patch isn't applied yet. In OCLP run **Post-Install Root Patch**, then reboot.

Settings in effect on this machine:

\`\`\`bash
# boot arguments
nvram boot-args
# -v debug=0x100 keepsyms=1 -igfxblr applbkl=1

# SIP state (OCLP's custom configuration; system protections being off is expected)
csrutil status
# System Integrity Protection status: unknown (Custom Configuration).
\`\`\`

- \`-v debug=0x100 keepsyms=1\`: keep it while debugging — verbose boot shows where it hangs; drop \`-v\` once stable;
- \`-igfxblr applbkl=1\`: Kaby Lake backlight flags; without them you get a black screen or no brightness control.

Change boot arguments through OCLP's UI rather than hand-editing \`nvram boot-args\` — it's easy to lose flags like \`keepsyms\`.

## 10. Rollback

Three options, least destructive first:

1. **Boot the other entry**: hold Option and pick the USB EFI, or set the startup disk back in System Settings;
2. **Restore the EFI**: \`ditto\` the backup from section 6 back (\`sudo ditto ~/efi-backup-<timestamp>/EFI /tmp/int_efi/EFI\`), then redo the startup-disk step from section 7;
3. **Reinstall**: boot the USB installer again, or restore from Time Machine.

## 11. Day-to-day maintenance

- After each **macOS point release** (15.x → 15.y), the root patch usually has to be re-applied or graphics/backlight will degrade;
- After **each OCLP upgrade**, rebuild OpenCore and update **both** the USB EFI and the internal EFI — not just one;
- Keep the two or three most recent timestamped EFI backups so you can diff them when something breaks;
- Don't enable FileVault and then start fiddling with EFI; it adds a whole recovery problem.

## 12. What to redact before publishing

When writing up something like this, strip: device serial number, hardware UUID, disk/partition UUIDs, full \`nvram\` output containing UUIDs, hostname, LAN IP, Apple ID / user name, and any full \`ioreg\` dump. Every command above works as-is without any of those values.

---

*记录时间 / Written: 2026-11*
