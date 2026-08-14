---
title: OBS与虚拟摄像头
date: 2026-08-15 00:00:00
description: 记录使用OBS与虚拟摄像头替代腾讯会议的共享屏幕
categories: 
- GNU/Linux
tags:
- OBS
- 虚拟摄像头
---
参考：
https://wiki.archlinux.org.cn/title/V4l2loopback

(不得不说腾讯会议做的真不咋地，说是支持原生wayland结果只适配了KDE)

# 介绍

虚拟摄像头是一种软件模拟的视频输入设备。它本身并不采集物理图像，而是接收来自其他程序或文件、屏幕等的视频流，并将其伪装成一个标准的摄像头设备，供其他应用程序（如视频会议软件、浏览器、录屏软件等）识别和使用。

主要用途

- **视频会议/直播**：把预先录制的视频、屏幕共享内容或经过处理的画面作为摄像头画面发送给对方。
- **隐私保护**：在某些必须开启摄像头的场景下，用虚拟摄像头提供替代画面，避免暴露真实环境。
- **软件测试**：模拟摄像头输入，测试视频相关应用的兼容性和功能。
- **特效处理**：将 OBS、滤镜软件等处理后的画面输出为摄像头，实现美颜、虚拟背景等效果。

# v4l2loopback

在 Linux 中，可以使用 `v4l2loopback` 创建虚拟摄像头。`v4l2loopback` 是一个内核模块，它能够创建若干个虚拟的 V4L2 视频设备（如 `/dev/video0`、`/dev/video10` 等）。这些设备看起来就像真实的摄像头，但实际上它们接收的是用户空间程序（如 `ffmpeg`、OBS）写入的视频数据。

有了 `v4l2loopback`，你就可以把任意视频流“喂”给这些虚拟设备，然后任何支持摄像头的应用都能直接使用它。

因此，一个典型的视频处理流程可以表示为：

```
摄像头 / 桌面 / 视频文件
          ↓
    OBS / FFmpeg 等
          ↓
     视频处理/合成
          ↓
     v4l2loopback
          ↓
     /dev/video10
          ↓
浏览器 / Discord / Zoom 等
```

# 在 ArchLinux 创建虚拟摄像头

1. 安装 obs：

```bash
pacman -S obs-studio
```

2. 安装 v4l2loopback-dkms 软件包 以及 目标内核的头文件（如 linux-headers）

```bash
pacman -S linux-headers

pacman -S v4l2loopback v4l-utils
```

3. 内核热加载模块

```bash
sudo modprobe v4l2loopback
```

4. 检查

```bash
dkms status
```

确保包含类似以下输出，多内核用户确保正在使用的内核是否安装成功：

```bash
v4l2loopback/0.15.4, 6.18.44-1-lts, x86_64: installed
v4l2loopback/0.15.4, 7.1.8-arch1-3, x86_64: installed
```

5. OBS 设置

![](20260815064729.png)

最后点击启动虚拟摄像头。

6. 验证是否可用

打开经典设备测试网站：[gUM Test Page](https://mozilla.github.io/webrtc-landing/gum_test.html)
选择 Camera 测试
选择授予虚拟摄像头权限
![](20260815065525.png)

可以看见已经正常捕获桌面屏幕
![](20260815065553.png)
