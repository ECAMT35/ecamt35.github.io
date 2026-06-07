---
title: 自用Linux桌面环境
date: 2026-06-07 00:00:00
updated: 2026-06-07 00:00:00 
description: 
categories: 
- GNU/Linux
tags:
- ArchLinux
---

My dotfiles : https://github.com/ECAMT35/dotfiles.git

# Arch

主文件系统：ext4
分区：/boot=1G, /=128G, SWAP=32G, /home=remaining, total=1T
boot：GRUB

命令解释器: bash
文本编辑器: neovim
网络管理: NetworkManager
蓝牙管理: blueman
硬盘检测: smartmontools
监控: btop
笔记本电源管理: TLP
固件更新：fwupd

# 桌面环境

窗口管理器：river + rivercarro

字体:
noto-fonts
noto-fonts-cjk
noto-fonts-emoji
ttf-hack
ttf-nerd-fonts-symbols
ttf-nerd-fonts-symbols-common

屏幕管理:
锁屏: swaylock + swayidle
关闭屏幕: wlopm + swayidle
屏幕分辨率: [wlr-randr](https://sr.ht/~emersion/wlr-randr/)
背光调节: brightnessctl
桌面壁纸: [swaybg](https://github.com/swaywm/swaybg)
bar: [dam（自定义修改）](https://github.com/ECAMT35/dam)

终端模拟器: foot
终端文件管理: yazi
输入法: fcitx5
启动器: [wofi](https://man.archlinux.org/man/wofi.1.en)
剪切板: wofi + wl-clipboard + [cliphist](https://github.com/sentriz/cliphist)
截图: [grim](https://sr.ht/~emersion/grim/) + slurp + swappy
浏览器: Firefox, Chromium
文件下载: aria2c, [aria2c config](https://github.com/P3TERX/aria2.conf)
安卓模拟器: Waydroid
移动硬盘、U 盘挂载：udiskie
快照：Timeshift + restic

音视频媒体:
声音设置: ALSA + PipeWire + WirePlumber
图片查看器: swayimg
视频播放器: mpv
视频录制: OBS Studio
桌面通知：fnott
串流：sunshine + moonlight(client)
XDG 桌面门户: xdg-desktop-portal-gtk, xdg-desktop-portal-wlr

文档编辑: LibreOffice, Obsidian
画板、图片编辑: Krita
小沙箱: Flatpak

游戏: Steam, Wine, KVM

其他 Wayland 应用查看：
https://arewewaylandyet.com/

# 优化部分

## ext4 挂载选项

推荐选项：
- noatime
- commit=180
- 开启 fast_commit

作用：
- noatime：减少磁盘写入
- commit=180：延长日志提交周期
- fast_commit：降低元数据写入开销

注：`commit` 参数调得太大这可能会导致突然断电时 ext4 文件系统损坏，不过正常都能使用 ArchISO 修复。

## 硬盘

开启 Periodic TRIM：

```bash
sudo systemctl enable fstrim.timer
```

## 禁用看门狗

https://wiki.archlinuxcn.org/wiki/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96#%E7%9C%8B%E9%97%A8%E7%8B%97

- nowatchdog
- nmi_watchdog=0

## 视频硬件加速

相关驱动（部分已经合并到 Mesa）：  
[https://wiki.archlinux.org/title/Hardware_video_acceleration](https://wiki.archlinux.org/title/Hardware_video_acceleration)

所需要的验证工具：
- vainfo
- vulkaninfo
- glxinfo

也可以利用 Firefox 的 `[更多排障信息](about:support)` 功能去检查

## Firefox

Firefox 开启硬件视频解码，进入 `about:config` 设置：

- media.ffmpeg.vaapi.enabled=true
- media.hardware-video-decoding.force-enabled=true

参考： [https://wiki.archlinux.org/title/Firefox#Hardware_video_acceleration](https://wiki.archlinux.org/title/Firefox#Hardware_video_acceleration)

关闭磁盘缓存减少 SSD 写入：

- browser.cache.disk.enable=false
- browser.cache.memory.enable=true

其他可选调整：  
[https://wiki.archlinux.org/title/Firefox/Tweaks](https://wiki.archlinux.org/title/Firefox/Tweaks)  
[https://github.com/arkenfox/user.js](https://github.com/arkenfox/user.js)

