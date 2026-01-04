---
title: Arch+River 使用
date: 2026-01-01 18:30:00
description: 自用系统配置
categories: 
- GNU/Linux
tags:
- Archlinux
- River
---

My dotfiles : https://github.com/ECAMT35/dotfiles.git

Q：为什么不用 Labwc 了？
A：想体验下平铺式的窗口管理器，并且听说 River 的资源消耗很友好。

# 实际体验

[River](https://codeberg.org/river)  虽然是平铺式窗口管理器，但是你也可以手动让他浮动起来做到堆叠的效果。River 和  Labwc 不同，它把合成器和布局管理器分离开，虽然它自带了一个 `rivertile` 作为一个布局管理器，但是我觉得并不太好用，我需要有窗口最大化的功能（最大化≠全屏），而社区开源的 `rivercarro` 很好地满足了我的需求，使我能在窗口管理上快速地从 Labwc 切换到 River。

除了平铺式的窗口管理，让我感知比较强的是资源占用。
以我的机器为基准，开机 Labwc 的初始占用大概是 `179M` 左右，久了可能会维持在 `350M` 上下。而 River 的初始占用是 `155M` 左右，虽然有几次我看到它的内存飙到 `400M`，但是很快就降到正常的状态，并且有时候会降到 `100M` 上下，感觉它的内存管理做得很好。

另外我已经不再使用 `Waybar`，它实在是过于重量而且 Bug 不少（之前的内存泄漏让我难受了一阵子，虽然说后面官方好像修了），但 Waybar 也是有优点的：开箱即用很便捷，并且挺好看。
作为替代，我使用了 River 社区开源的 [dam](https://codeberg.org/sewn/dam)。样子和个性化确实没 Waybar 好，但相比与 Waybar 这个实在太轻量了（Waybar 的初始内存占用大概是 `64M`，用久了可能成 `90M`，而 `dam + 脚本` 几乎一直是 `11M +6M` 上下），缺点就是需要自己完成脚本制作来获取数据以及不能被包管理器管理（每改一次就需要重新编译）。

虽然说这个时代可能都是 16G 内存起步的电脑了，而且窗口管理器（WM）的资源占用都挺小，纠结着几百 M 的内存和 CPU 意义不大。但是我还是更喜欢这种轻量、优秀的程序啊（也不是说  Labwc 和 Waybar 是垃圾的意思，它们都很优秀，只是 River 更符合我的爱好）。

唯一的缺点是，偶尔我的 `Super` 键会失灵。这个 River 的 Bug 有但很少触发，我至今不知道怎么才能再现这个 Bug，希望后续能彻底解决吧。

总的来说体验挺好，期待 River 0.4 的大更新。

# 桌面软件包

其实软件包和 [《Arch+Labwc 使用》](https://ecamt35.github.io/ce269dce74a9/) 里的差不多，只有一小部分做了更换：

更换内容：
bar: Waybar -> dam
背光调节: brightnessctl -> 自定义脚本，见 dotfile 仓库: `.local/bin`
监控: htop -> btop
浏览器: Firefox -> Firefox + Chromium
字体：增加 `ttf-hack` 为 monopace 首选

增加了 KVM，因为去工作了，需要用到一堆 Window 专用的软件

另外，附带上自己更改的 dam，直接编译就能用了，修改详情见 patch 目录，以及 dam 所需的脚本（[unixchad/damblocks](https://codeberg.org/unixchad/damblocks) 给了我很多灵感）：
- [dam](https://github.com/ECAMT35/dam)
- [dam-init](https://github.com/ECAMT35/dotfiles/blob/master/.config/dam/init)—这个还需要配合几个脚本使用，相关脚本仍在我的 dotfile 仓库: `.local/bin`

