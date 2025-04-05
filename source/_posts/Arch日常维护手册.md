---
title: Arch日常维护手册
date: 2024-03-12 00:00:00
description: Archlinux日常维护命令速查
categories: 
- GNU/Linux
tags:
- Archlinux
---

# 包管理

## pacman

### 升级
升级系统及所有已经安装的软件
```bash
pacman -Syu
```

### 安装
直接安装某一个包
```bash
pacman -S <packageName>
```

### 安装原因
pacman 数据库将软件包按照安装的原因分为两类：
- **显式安装**：那些真正地被传递给通用 pacman `-S`和`-U`命令的包；
- **依赖**：那些虽然（一般）从未被传递给 pacman 安装命令，但由于被其它显式安装的包[需要](https://wiki.archlinuxcn.org/wzh/index.php?title=Dependency&action=edit&redlink=1 "Dependency（页面不存在）")从而被隐式安装的包

当安装软件包时，可以把安装原因强制设为**依赖**:
```bash
pacman -S --asdeps <package_name>
```

该命令使用的原因通常是显式安装的包可能会提供[可选安装包](https://wiki.archlinuxcn.org/wiki/PKGBUILD#optdepends "PKGBUILD")，这些包提供了非必须功能，可供用户自由选择。

>提示：用`--asdeps`安装可选依赖将确保如果你[移除孤立包](https://wiki.archlinuxcn.org/wiki/Pacman/%E6%8F%90%E7%A4%BA%E5%92%8C%E6%8A%80%E5%B7%A7#删除未使用的软件包（孤立软件包） "Pacman/提示和技巧")，pacman 将会一同移除按照上述方法指定安装的可选依赖。

但是重新安装该软件包，当前安装原因默认会被保留。

显式安装的软件包列表可用`pacman -Qe`获取, 设置为依赖的软件包可用`pacman -Qd`获取。

改变某个已安装软件包的安装原因，可以执行：
```bash
pacman -D --asdeps <package_name>
```

反过来的对应参数`--asexplicit`

**注意：** 在升级时使用`--asdeps`和`--asexplicit`选项，例如`pacman -Syu <package_name> --asdeps`，是不被推荐的。这会导致不仅改变要被安装的软件包的安装原因，也会改变被升级的软件包的安装原因。

>根据2021.05-1之后的行为变化，即使指定了 `--asdeps` 参数，`<package_name>` 本身仍然会被视为显式安装的包。只能显示安装后在通过 `pacman -D --asdeps <package_name>` 变为以依赖安装

### 搜索

```bash
pacman -Ss <packageName> # 在仓库中搜索含关键字的软件包（本地已安装的会标记）

pacman -Qs <packageName> # 搜索已安装的软件包

pacman -Qu # 列出所有可升级的软件包

pacman -Qt # 列出不被任何软件要求的软件包

pacman -Q <packageName> # 查看软件包是否已安装，已安装则显示软件包名称和版本

pacman -Qi <packageName> # 查看某个软件包信息，显示较为详细的信息，包括描述、构架、依赖、大小等等

pacman -Ql <packageName> # 列出软件包内所有文件，包括软件安装的每个文件、文件夹的名称和路径

pacman -Qo /path/to/a/file # 可以通过查询数据库获知目前你的文件系统中某个文件是属于哪个软件包

pacman -Qdt # 罗列所有不再作为依赖的软件包（孤立 orphans)
```

### 清理

删除孤立软件包
```bash
pacman -Rns $(pacman -Qtdq)
```

删除当前未安装的所有缓存包和未使用的同步数据库
```bash
pacman -Sc
```

从缓存中删除所有文件，这是最激进的方法，不会在缓存文件夹中留下任何内容
```bash
pacman -Scc
```

### 卸载

删除指定软件包，及其所有没有被其他已安装软件包使用的依赖关系：
```bash
pacman -Rns <packageName>

# -R 删除单个软件包，会保留其全部已经安装的依赖关系
# -n pacman 删除某些程序时会备份重要配置文件，在其后面加上*.pacsave扩展名。-n 选项可以避免备份这些文件
# -s 删除其所有没有被其他已安装软件包使用的依赖关系
```

### 其他用法

在不同的 `-大写字母`，可跟着不同的小写字母实现更多不同的功能，具体可用指南查看
```bash
pacman -h # 查看指南

pacman {-R --remove} # 查看 -R 的可选参数
```

## yay

### 升级

升级系统，相当于yay -Syu
```bash
yay
```

### 安装
```bash
yay -S <packageName>
```

### 搜索

查看仅通过yay安装的包
```bash
yay -Qm
```

### 卸载

要删除包及其依赖项
```bash
yay -Rns <packageName>
```

### 清理

清理 yay 缓存
```bash
rm -rf ~/.cache/yay
```

清理孤立的包
```bash
yay -Yc

# -Y：表示同时清理未使用的依赖项。
# -c：表示清理缓存，即删除已下载的软件包文件。
```

### 其他用法

打印系统统计信息，显示已安装软件包和系统健康状况的统计数据
```bash
yay -Ps
```

## pacnew处理

处理过程基本一致，请看`mkinitcpio.conf变动`章节

## mkinitcpio.conf变动

在Arch Linux系统中，`mkinitcpio.conf` 是一个非常重要的配置文件，它用于定义在初始化ramdisk（initrd或initramfs）创建过程中包含哪些模块和文件。
这个配置对系统启动至关重要。当系统更新并且这个更新影响到 `mkinitcpio` 时，可能会生成一个新的配置文件示例，名为 `mkinitcpio.conf.pacnew` 。

处理 `mkinitcpio.conf` 和 `mkinitcpio.conf.pacnew` 文件步骤如下：

1. 首先，你需要检查 `.pacnew` 文件与现有的 `mkinitcpio.conf` 文件之间的差异。可以使用 `nvim -d` （Diff mode）来完成这一任务。如：
```bash
nvim -d /etc/mkinitcpio.conf /etc/mkinitcpio.conf.pacnew
# 这将显示两个文件之间的差异。vimdiff 的使用看 vim 篇
```

2. 在做任何更改之前，建议备份原始的 `mkinitcpio.conf` 文件。这样，如果新的配置导致问题，你可以轻松地回滚到之前的配置。如：
```bash
cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.bak
```

3. 把 `mkinitcpio.conf.pavnew` 的新增参数内容复制到 `mkinitcpio.conf` ，对于不同的参数的值，请谨慎更改或保留。（建议使用 vim/neovim 的 [diff 模式](https://neovim.io/doc/user/diff.html#_1.-starting-diff-mode) 完成）

4. 一旦你合并了并保存了 `mkinitcpio.conf` 文件，你需要重新生成 initramfs 使更改生效：
```bash
mkinitcpio -P
# 为所有已安装的内核生成新的initramfs。
```

5. 更改生效后，重启你的系统以确保一切正常工作。没啥问题就可以删除`.pacnew`文件和备份文件了

### mkinitcpio -P
命令`mkinitcpio -P`在Arch Linux及其衍生系统中用于重新生成所有已安装内核的初始RAM磁盘（initramfs）。initramfs是一个临时的根文件系统，加载到内存中，在Linux系统启动过程中被用来准备真正的根文件系统。

具体来说，`mkinitcpio -P`命令的作用包括：
- **自动检测**：该命令会自动检测`/boot`目录下所有已安装的Linux内核，并为每个内核生成对应的initramfs。这意味着如果你有多个内核版本（例如，标准Linux内核和LTS版本内核），该命令将为它们每一个重新生成initramfs。
    
- **应用配置更改**：如果你更改了`/etc/mkinitcpio.conf`配置文件或者相关的钩子脚本（hooks），使用`mkinitcpio -P`可以确保这些更改被应用到所有内核的initramfs中。这对于添加驱动、修改启动行为等操作至关重要。
    
- **修复启动问题**：在某些情况下，如果系统无法启动，可能是因为initramfs损坏或配置不正确。在这种情况下，重新生成initramfs可能有助于解决问题。
    
- **更新内核后的必要步骤**：当通过包管理器更新Linux内核后，通常会自动重新生成initramfs。然而，在手动安装或更新内核、更改重要的启动配置时，运行`mkinitcpio -P`确保initramfs是最新的并且包含所有必要的模块和配置。

# 磁盘
## df命令（Disk Free）

显示 **文件系统级别（挂载的磁盘/分区）** 的整体磁盘空间使用情况，包括总容量、已用空间、剩余空间等。-h 后的默认参数为 `/` ，即根目录，查看的是根目录中所有分区的空间使用。
```bash
df -h # 以人类可读格式显示
```

若想查看指定一目录的分区的空间使用情况，可以在 -h 后自定义
```bash
df -h /mnt # 查看/mnt目录下空间使用情况
```

## du命令（Disk Usage）

计算 **文件或目录** 的实际磁盘使用量（递归统计文件和子目录占用的空间）。

```bash
du -sh /path/to/dir   # 统计目录总大小（-s 汇总，-h 易读格式）
du -ah /path/to/dir   # 显示目录下所有文件和子目录的详细大小
```
## Smartmontools

1. 安装 smartmontools
```bash
pacman -S smartmontools
```

2. 使用以下命令获取磁盘设备名称或路径
```bash
fdisk -l
```

3. 使用 `smartctl` 命令来查看磁盘信息：
```bash
smartctl -A /dev/sdx # 硬盘
smartctl -d sat -A /dev/sdx # USB 设备
```
   这两个命令的主要区别是添加了 `-d` 选项，该选项指定驱动程序，以便 `smartctl` 可以与相应的设备进行通信。
   如果需要更加详细的信息，请把 `-A` 改为 `-a` 。

## trim

1. 查看trim定时器状态
```bash
systemctl status fstrim.timer  
```

2. 手动 trim
```bash
fstrim -a -v
```
- `-a` 或 `--all`：这个选项指示`fstrim`命令尝试对所有挂载的文件系统进行trim操作，但它会跳过那些不支持trim的文件系统。
- `-v` 或 `--verbose`：这个选项让`fstrim`命令在执行时显示更多信息，特别是关于它对每个文件系统进行了多少trim操作的详细信息。

# 日志
## 日志清理
清理最早两周前的日志.
```bash
journalctl --vacuum-time=2s
```

清理日志使总大小小于 100M:
```bash
journalctl --vacuum-size=100M
```

## pacman日志
一般只会保留数天的记录，使用 cat 抓取就行
```bash
cat /var/log/pacman.log 
```

## 读取日志

检查上一次引导（boot）时的系统日志的命令（包括内核和用户空间日志）。依赖 `systemd-journald` 的持久化存储。
```bash
journalctl -b -1

# `-b` 表示启动周期（`boot`），`-1` 表示上一次启动（`-0` 是当前启动，默认值）。
```

---

直接读取 **内核环形缓冲区**（`kernel ring buffer`）的内容，记录内核启动过程中和运行时的日志（如硬件检测、驱动加载、内核错误等）。  
这些日志是内核直接生成的，**不依赖任何日志系统**（如 `systemd-journald`）。

```bash
sudo dmesg

# 默认显示当前启动周期的内核日志（重启后会丢失之前的日志）。  
# 可通过 `-T` 或 `--time-format` 选项显示时间戳（需内核支持）。
```

# 进程

## 1. 使用ps命令

`ps`命令是一个非常通用的命令，它在几乎所有的Linux发行版中都可用，包括Arch Linux。

查看当前系统中的所有进程：
```bash
ps -ef
```
  这里，`-e`表示显示所有进程，`-f`表示全格式显示。

查看与某个用户相关的进程：
```bash
ps -u <用户名>
```
  
查看某个特定进程的信息：
```bash
ps -p <进程ID>
```

## 2. 使用top命令

`top`命令提供了一个实时更新的进程列表，显示系统中进程的动态视图。

启动`top`命令：
```bash
top
```

`top`命令会显示一系列的进程信息，包括CPU使用率、内存使用率等。可以在`top`运行时按键来改变其行为:
1. 如按`P`排序显示CPU使用率最高的进程。
2. 按 `q` 退出。

>可以使用拥有更好交互图形界面的 htop。

# 蓝牙
开启蓝牙服务
```bash
systemctl start bluetooth
```

# WIFI-NetworkManager
## nmcli examples

https://wiki.archlinuxcn.org/wiki/NetworkManager

List nearby Wi-Fi networks:

```bash
$ nmcli device wifi list
```

Connect to a Wi-Fi network:

```bash
$ nmcli device wifi connect _SSID_or_BSSID_ password _password_
```

Connect to a hidden Wi-Fi network:

```bash
$ nmcli device wifi connect _SSID_or_BSSID_ password _password_ hidden yes
```

Get a list of connections with their names, UUIDs, types and backing devices:

```bash
$ nmcli connection show
```

Delete a connection:

```bash
$ nmcli connection delete _name_or_uuid_
```

See a list of network devices and their state:

```bash
$ nmcli device
```

Turn off Wi-Fi:

```bash
$ nmcli radio wifi off
```


# 临时粘贴版 fars.ee

官网： http://fars.ee/

~~用于把日志、配置贴到任何人都能访问的网络上，用于让大佬给你分析问题~~

>注意隐私

以下为基本使用，更多用法看官网
Create a paste from the output of 'dmesg':
```bash
$ dmesg | curl -F "c=@-" "http://fars.ee/"

:' output
long: AGhkV6JANmmQRVssSUzFWa_0VNyq
sha1: 686457a240366990455b2c494cc559aff454dcaa
short: VNyq
url: http://fars.ee/VNyq
uuid: 17c5829d-81a0-4eb6-8681-ba72f83ffbf3
'
```

# 显卡

首先确保已经安装了 `mesa-utils` 、`lib32-nvidia-utils`、`nvidia-utils`、`nvidia` 和 `nvidia-utils`等相关驱动（或者AMD的相关包）

## 查看当前的显卡渲染器

```bash
glxinfo | grep "OpenGL renderer"
```

## 监控 Nvidia GPU
```bash
nvidia-smi
```

