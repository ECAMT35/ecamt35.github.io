---
title: Arch+Labwc 使用
date: 2025-01-23 17:29:09
description: 自用系统配置
categories: 
- GNU/Linux
tags:
- Archlinux
- Labwc
---
Archlinux + Labwc WM (Wayland)

My dotfiles : https://github.com/ECAMT35/dotfiles.git

# Arch

主文件系统：ext4
分区：/boot=1G, /=128G, SWAP=32G, /home=remaining, total=1T
boot：GRUB

命令解释器: bash
文本编辑器: neovim
网络: NetworkManager
硬盘检测: smartmontools
监控: htop
笔记本电源管理: TLP

# labwc

官方文档：[labwc](https://labwc.github.io/integration.html)

## 桌面环境软件包

字体:
noto-fonts 
noto-fonts-cjk
ttf-nerd-fonts-symbols
ttf-nerd-fonts-symbols-common

屏幕管理:
锁屏: swaylock + swayidle
关闭屏幕: wlopm + swayidle
屏幕分辨率: [wlr-randr](https://sr.ht/~emersion/wlr-randr/)
背光调节: brightnessctl
桌面壁纸: [swaybg](https://github.com/swaywm/swaybg)
bar: waybar

终端模拟器: foot
输入法: fcitx5
启动器: [wofi](https://man.archlinux.org/man/wofi.1.en)
剪切板: wofi + wl-clipboard + [cliphist](https://github.com/sentriz/cliphist)
截图: [grim](https://sr.ht/~emersion/grim/) + slurp + swappy
终端文件管理: yazi
浏览器: firefox
文件下载: aria2c, [aria2c config](https://github.com/P3TERX/aria2.conf)
~~消息提醒: dunst~~
安卓模拟器: waydroid

音视频媒体:
声音设置: alsa + pipewire + wireplumber
图片查看器: swayimg
视频播放器: mpv
视频录制: OBS
XDG 桌面门户: xdg-desktop-portal-gtk, xdg-desktop-portal-wlr

游戏: wine, steam, ~~KVM~~

文档编辑: LibreOffice, obsiadian
画板、图片编辑: krita
小沙箱: flatpak

---

其他wayland应用查看
https://arewewaylandyet.com/

# 优化部分

## ext4挂载选项

- noatime
- commit=180
- 开启 fast_commit

## 硬盘

- 开启 Periodic TRIM

## 禁用看门狗

- 禁用看门狗

## 视频硬件加速

相关驱动(部分已经合并到mesa)：
https://wiki.archlinux.org/title/Hardware_video_acceleration

firefox进`about:config`设置：
https://wiki.archlinux.org/title/Firefox#Hardware_video_acceleration
- media.ffmpeg.vaapi.enabled=true
- media.hardware-video-decoding.force-enabled=true

## firefox

关闭磁盘缓存：
- browser.cache.disk.enable=false
- browser.cache.memory.enable=true

其他可选调整：
https://wiki.archlinux.org/title/Firefox/Tweaks
https://github.com/arkenfox/user.js/
