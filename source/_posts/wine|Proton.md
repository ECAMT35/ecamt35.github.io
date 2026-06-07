---
title: Wine/Proton
date: 2026-06-07 00:00:00
updated: 2026-06-07 00:00:00 
description: 使用Wine/Proton运行.exe程序
categories: 
- GNU/Linux
tags:
- Wine/Proton
---

参考资料：
https://wiki.archlinuxcn.org/wiki/Wine#
https://www.protondb.com/

# 自用方案

使用 bottles/lutris 作为前端启动器，使用 [ProtonPlus](https://github.com/Vysp3r/ProtonPlus) 管理容器版本；
日常使用 Wine-staging/[proton-ge-custom-bin](https://aur.archlinux.org/packages/proton-ge-custom-bin)/[DW-Prototn](https://aur.archlinux.org/packages/dwproton-bin) 作为容器。

推荐安装的函数库： 

- d3d11 
- d3d9
- devenum
- gdiplus
- quartz
- wmp11
- directshow
- lavfilters

部分应用可能会需要其他的依赖（参考 [WineHQ 页面](https://gitlab.winehq.org/wine/wine/-/wikis/Building-Wine#satisfying-build-dependencies)），可以一劳永逸全部装完：

- TLS 的 32 位支持可以安装 [lib32-gnutls](https://archlinux.org/packages/?name=lib32-gnutls)
- 手柄和游戏杆的 32 位支持可以安装 [lib32-sdl2](https://aur.archlinux.org/packages/lib32-sdl2/)
- 媒体播放的 32 位支持可以根据需求安装 [lib32-gst-plugins-base](https://archlinux.org/packages/?name=lib32-gst-plugins-base)、 [lib32-gst-plugins-good](https://archlinux.org/packages/?name=lib32-gst-plugins-good)、[lib32-gst-plugins-bad](https://aur.archlinux.org/packages/lib32-gst-plugins-bad/)、[lib32-gst-plugins-ugly](https://aur.archlinux.org/packages/lib32-gst-plugins-ugly/) 和 [lib32-gst-libav](https://aur.archlinux.org/packages/lib32-gst-libav/)
- 对于 [NTLM](https://en.wikipedia.org/wiki/NTLM "wikipedia:NTLM") 验证可以安装 [samba](https://archlinux.org/packages/?name=samba)

运行依赖于 .NET 的应用程序需安装：wine-mono
运行依赖于 Internet Explorer 的应用程序 ：wine-gecko
这两个可以在 bottles/lutris 里面进行下载，一般可以不行进系统级的安装

# 常见问题
## 字体

建议安装 window 的字体到 wine 盘解决，只需要把 window 字体文件夹 `Font` 覆盖到 wine 盘的 c 驱动盘的 `window/Font` 即可。
~~其实就是把 `.ttf` 文件复制到 `window/Font` 目录下，用一个软链接就行~~

## 运行失败

运行不了的程序，如果找不出原因去修，那就尝试使用其他版本的 Wine（这就是我推荐使用 ProtonPlus 的原因，别只盯着原版 Wine 使用，这会让你轻松很多）

可以到 protonDB 看看有没有运行成功的例子，借鉴下方案。

如果还是不行只能启动 KVM 了

## 窗口缩放

Wine/Proton 默认使用 96 DPI，在高分辨率屏幕上可能导致程序窗口和字体过小。可以根据显示器 DPI 适当提高 Wine 的 DPI 设置，使程序以更大的 UI 尺寸进行渲染。
gamescope 也可以用于全屏显示和分辨率缩放，但如果将较低分辨率的画面放大到高分辨率输出，文字和 UI 可能会变得模糊，因此优先调整 DPI 或使用程序自身的高 DPI 支持。

# 提示与技巧
## Wineconsole

通常情况下，你可能需要运行.exe 来修补游戏文件，例如一个老游戏的宽屏 mod，通过 Wine 正常运行.exe 可能不会产生任何效果。在这种情况下，你可以打开一个终端，运行以下命令。

```bash
$ wineconsole cmd
```

然后转到该目录，从那里运行 _.exe_ 文件。

## 游戏内显示帧数

Wine 有一个内置的 FPS 检测器，在环境变量 `WINEDEBUG=fps` 设定的情况下，它会运行在所有图形程序上。这将会输出帧率到标准输出。你可以显示帧率到屏幕最上面通过 [xosd](https://archlinux.org/packages/?name=xosd) 包 包中的 `osd_cat`。[winefps.sh](https://gist.github.com/anonymous/844aefd70bb50bf72b35) 为它的帮助脚本。

## Vulkan

默认的 wine Vulkan ICD 加载器在大多数程序中都工作地很好，但是不支持高级特性，例如 Vulkan 层。为使用这些特性，你需要安装官方的 Vulkan SDK，见原始的 Vulkan 补丁作者的 [GitHub page](https://github.com/roderickc/wine-vulkan) 的第二到四步。


