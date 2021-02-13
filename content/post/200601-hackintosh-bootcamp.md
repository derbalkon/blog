---
title: 喜闻乐见的 Boot Camp (Hackintosh) 踩坑集锦
date: 2020-06-01
lastmod: 2020-06-21
image: /post/image/cover-image.png
description: 即使是在白苹果上，Boot Camp 的装机体验其实也没有想象中那么美好。
---

自从年初使用了广为流传的 300 系主板的原生 NVRAM 解法，并且全面转换到 OpenCore 引导 macOS 之后，几乎已经忘了这台主机是一台 PC 了。偶尔出于娱乐和兼容的需求要用到 Windows，又苦于 Parallels Desktop 的订阅份额被自己的 MacBook Pro 用掉了，于是萌生了利用 Boot Camp 进行启动转换的想法，让这台 PC 向 Mac 更进一步。~~既然要追求极致，那就贯彻到底咯～（什么烂梗）~~

**注意**：这里所说的 Boot Camp 仅仅是指 **启动转换**，而不是用 Boot Camp 去安装 Windows。理论上这样做是可行的，但是 PC 毕竟不是 Mac，在硬盘数量、分区或者其他一些东西上有差别，会导致 Boot Camp 系统盘制作错误、安装错误等等玄学问题。

## 用 OpenCore 引导有什么弊端？

正如 [OpenCore 官方文档](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.pdf) 写得那样：

- 不支持 MBR；
- ACPI、NVRAM 和 SMBIOS 在 OpenCore 上的所有改动都会被一视同仁地应用到 Windows 上；
- 如果 config.plist 中的 UUID 填写的是随机生成的，而不是主板本身的，Windows 系统可能需要重新激活。

但是反过来看，这些「弊端」其实是为了达到真正 Boot Camp 的使用体验而留下的，更加 **一体化且无缝**，毕竟真正的 Mac 设备并不需要两份不同的 ACPI、NVRAM 和 SMBIOS 不是吗？

## 为什么不用 Boot Menu 引导 Windows？

原因有很多：

- 比如用 Boot Menu 引导绕过了 OpenCore 启动器，也就意味着放弃了 Boot Camp 的体验——在操作系统中实现启动切换；
- 比如做好了 ACPI 的系统判断，UUID 也填好了主板本身的值，那么用 OpenCore 引导的「副作用」就不那么显著了；
- 还比如有些~~小倒霉蛋的~~主板在使用 OpenCore 引导之后，可能会出现按 <kbd>F11</kbd> 卡在黑屏而不能进入 Boot Menu 或 BIOS 的情况，那么就别无选择，只能用 OpenCore 来引导 Windows 了。

## 折腾 Boot Camp 的时候有那些坑？

由于我本人的机器仅仅是单硬盘双系统（分区为简单的 `EFI` + `Macintosh HD` + `Windows HD`），因此下面的所有描述都是在单硬盘双系统的情况下，可能遇到问题和相应解法。

### Boot Camp 死活装不上

当我定制好了 ACPI 和 SMBIOS，满怀欣喜地引导到 Windows 时，突然发现自己准备好的 Boot Camp 安装包无法完成安装，提示我不支持在这台机器上安装。

难道没有被识别为 Mac？

于是打开 AIDA64，看了一眼，果然概览还是显示为 MSI，而不是 iMac19,1。那么问题肯定就处在 SMBIOS 身上了。仔细地排查了一下机型信息，发现并没有哪里填的不对。后来看了一眼官方文档才知道，原来从 OpenCore 0.5.8 开始，把 `UpdateSMBIOSMode` 设置为 `Custom`，所填写的 SMBIOS 信息就不会被应用到 Windows 了。于是改回 `Create`，让 OpenCore 自己根据 `Generic` 里填写的必要信息生成一套齐全的 SMBIOS，果然成功安装。

在这里值得一提的是：当你把 UUID 填写为主板本身的 UUID 时，因为某些主板本身的 UUID 可能错的、不是随机生成的，Mac 下通过 Hackintool 查看会发现，`系统 ID`（也就是 `system-id`）可能会显示为 `???`，但目前就我本人的体验来看，并没有什么实质性的影响。

### Boot Camp 安装旷日持久，甚至卡死

看起来像是卡死了，实际上只是在原地待命，因为没有提前把不需要的驱动程序从大礼包中删掉，Boot Camp 不认识这台设备——「这不是我认识的那个 Mac！」。因此只需要删掉不需要的驱动即可。

- 如果你有 Apple 的拆机无线卡，`$WinPEDriver$` 文件夹中，除了 `AppleBluetoothBroadcom`, `BroadcomWireless`, `BroadcomBluetooth` 之外，其他都可以删掉。
- `BootCamp\Drivers` 里面的文件，`Apple` 不要删；如果有 Apple 的拆机卡， `Broadcom\BroadcomBluetooth` 也不要删。

### Mac 下的更改启动磁盘时报错

更改启动磁盘时，有时会出现 **Bless 工具无法设定当前的启动磁盘** 的错误提示，确保 `AdviseWindows` 和 `WriteFlash` 设置为 `True`，一般来说后者是引起这个问题的主要因素。

### Mac 下的启动磁盘不显示 Windows 分区

原因是你可能使用了第三方 NTFS 插件，比如 Tuxera 等，它会把 Windows NT 系统改为 **Windows NT 系统（Tuxera）**，macOS 自然也不会识别这种文件系统类型。

### Windows 下的启动转换助理不显示 Mac 分区

Boot Camp 的版本可能已经过时了，更新到最新版本应该就可以了。

### Boot Camp 中 Windows 分区显示为 Basic data partition

一般来讲，Windows 都是单独安装的，而 Boot Camp 是使用 GPT 分区表获取每个引导选项的名称的，因此也需要手动重新标记 Windows 分区。具体可参考[这个段落](https://oc.skk.moe/12-troubleshooting.html#3-为什么我会在-Boot-Camp-启动硬盘-控制面板-中看到-Basic-data-partition？)。

### OpenCore 启动菜单里没有 Windows 选项！

首先确保安装 Windows 时不是 MBR(legacy) 安装的，因为 OpenCore 只支持基于 UEFI 安装的 Windows。

然后查看一下 `ScanPolicy` 是否已经设置为 `0`，或其他能够识别到 Windows 文件系统和所在磁盘的类型的值。（一般来说设置为 `0` 比较省事，如果想要精简也可以自己调整）

OpenCore 从 0.5.9 开始使用了全新的启动管理，到此为止就可以自动识别到 Windows 分区了；0.5.8 及更老的版本还需要在 `BlessOverride` 中设置 `bootmgfw.efi` 的路径。

### Windows 蓝屏了！！

常见的错误代码可能是 `0xc000000d`，一般都是 config 没有配置好导致的。看看 `SyncRuntimePermissions` 有没有启用。新主板可能还要启用 `ProtectUefiServices`，13 年以前的主板可能要启用 `ProtectMemoryRegions`。

### D:\ 不见了！！！

原生的 Boot Camp 的逻辑是，创建一个 D 盘分区作为安装目录，把 Windows 安装到 C 盘，成功安装后 D 盘就功成身退。尽管你不是通过 Boot Camp 安装的 Windows，但当你有多个分区的时候，你的下一个分区依然会被 Boot Camp 误认为是安装目录了。解决这个的办法很简单，用 **傲梅分区助手** 把 D 盘取消隐藏即可。
