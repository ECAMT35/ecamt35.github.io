---
title: KVM
date: 2026-06-15 00:00:00
description: KVM的使用体验
categories: 
- GNU/Linux
tags:
- KVM
---
参考：
https://wiki.archlinuxcn.org/wiki/KVM

QEMU（Quick Emulator）是一款开源的**硬件虚拟化软件**，它可以在不同的主机平台上运行虚拟机。QEMU 采用**全系统模拟**方式，可以模拟完整的计算机系统（包括处理器、内存、存储和外围设备）。

KVM（Kernel-based Virtual Machine）是**基于内核的虚拟机**，它是 Linux 内核的一部分。KVM 利用硬件扩展（如 Intel VT 或 AMD-V）实现高效的**硬件辅助虚拟化**。

**KVM 和 QEMU 通常是结合使用的**，被称为 "qemu-kvm"。KVM 负责 CPU 和内存的虚拟化，而 QEMU 负责模拟各种硬件设备。这样既保证了性能，又保持了灵活性。

# 检查 KVM 支持

# 安装 QEMU

与其他的虚拟化程序如 [VirtualBox](https://wiki.archlinuxcn.org/wiki/VirtualBox "VirtualBox") 和 [VMware](https://wiki.archlinuxcn.org/wiki/VMware "VMware") 不同, QEMU 不提供管理虚拟机的 GUI（运行虚拟机时出现的窗口除外），也不提供创建具有已保存设置的持久虚拟机的方法。除非您已创建自定义脚本以启动虚拟机，否则必须在每次启动时在命令行上指定运行虚拟机的所有参数。

## QEMU 后端

Arch Linux 中相关安装包的详细信息

- [qemu-desktop](https://archlinux.org/packages/?name=qemu-desktop) 包 包提供了 `x86_64` 架构的模拟器， 可以进行全系统模拟 (`qemu-system-x86_64`)。 [qemu-emulators-full](https://archlinux.org/packages/?name=qemu-emulators-full) 包包提供了 `x86_64` 用户模式的模拟 (`qemu-x86_64`)。 对于其他支持的架构，这个包都提供了全系统模拟和用户模拟两个变种 (比如说 `qemu-system-arm` 和 `qemu-arm`)。
- 对应于这些包的无界面版本 (仅适用于全系统模拟模式) 分别是 [qemu-base](https://archlinux.org/packages/?name=qemu-base) 包 (仅 `x86_64`) 和 [qemu-emulators-full](https://archlinux.org/packages/?name=qemu-emulators-full) 包 (其余架构)。
- 可以用如下独立的安装包中的 QEMU 模块扩展全系统模拟的功能: [qemu-block-gluster](https://archlinux.org/packages/?name=qemu-block-gluster) 包, [qemu-block-iscsi](https://archlinux.org/packages/?name=qemu-block-iscsi) 包, 和 [qemu-guest-agent](https://archlinux.org/packages/?name=qemu-guest-agent) 包。
- [qemu-user-static](https://archlinux.org/packages/?name=qemu-user-static) 包 为所有 QEMU 支持的架构提供了一个带用户模式和静态链接模式的变种。 它的 QEMU 命令依照 `qemu-_target_architecture_-static` 的规则命名, 例如, `qemu-x86_64-static` 代表目标架构为 intel 64 位 CPU。

如果只需要模拟 x86_64 的架构的话，qemu-desktop 是最好的选择：

```bash
sudo pacman -S qemu-desktop
```

## QEMU 的图形前端

[Libvirt](https://wiki.archlinuxcn.org/wiki/Libvirt) 提供了一种管理 QEMU 虚拟机的便捷方式。有关可用的前端，请参阅 [libvirt 客户端列表](https://wiki.archlinuxcn.org/wiki/Libvirt#客户端 "Libvirt")。

**[Virt-manager](https://wiki.archlinuxcn.org/wzh/index.php?title=Virt-manager&action=edit&redlink=1)** — 用于图形化使用 libvirt 管理 KVM，Xen 或是 LXC。

那么使用 Virt-manager 来作为前端管理就够了。

```bash
sudo pacman -S virt-manager
```

对于网络连接，需要安装这些包：

- [dnsmasq](https://archlinux.org/packages/?name=dnsmasq) 包 用于 [默认的](https://wiki.libvirt.org/VirtualNetworking.html#the-default-configuration) NAT/DHCP 网络
- [openbsd-netcat](https://archlinux.org/packages/?name=openbsd-netcat) 包 用于 [SSH](https://wiki.archlinuxcn.org/wiki/SSH "SSH") 远程管理

其他可选依赖项可能提供所需或扩展功能，例如 [dmidecode](https://archlinux.org/packages/?name=dmidecode) 包 用于支持 DMI 系统信息。建议在阅读 [libvirt](https://archlinux.org/packages/?name=libvirt) 包 的 pacman 输出后，将所需组件 [作为依赖项](https://wiki.archlinuxcn.org/wiki/Pacman#Installation_reason "Pacman") 安装。

# 使用

ibvirt 可通过两种模式管理 QEMU 虚拟机：system（系统模式）和 session（会话模式）：

- **system** URI 连接到以 root 身份运行的 libvirtd daemon（该进程在系统启动时加载）。使用 'system' 模式创建和运行的虚拟机默认以 root 身份启动，除非另行配置（例如通过 `/etc/libvirt/qemu.conf`）。
- **session** URI 会以当前本地用户身份启动 libvirtd 实例，所有虚拟机均以该用户权限运行。

**模式对比分析**：

- 仅 'system' 模式支持虚拟机随宿主机启动自动运行，且 root 权限的 libvirtd 实例拥有通过 bridge 或虚拟网络配置完整网络连接的权限。`qemu:///system` 通常是 virt-manager 等工具的默认选项。
- `qemu:///session` 存在显著缺陷：由于 libvirtd 实例权限不足，仅支持 qemu 的 usermode networking（用户模式网络），该模式存在隐性限制，因此不建议使用（[详见qemu 网络配置](https://web.archive.org/web/20210426155203/https://people.gnome.org/~markmc/qemu-networking.html)）。

`qemu:///session` 的优势在于权限简化：用户可直接在 $HOME 存储磁盘镜像、串行 PTY 设备归属用户自身等。

对于 **系统** 级别的管理任务（如：全局配置和镜像卷位置），libvirt 要求至少要 [设置授权](https://wiki.archlinuxcn.org/wiki/Libvirt#设置授权) 和 [启动守护进程](https://wiki.archlinuxcn.org/wiki/Libvirt#守护进程)。

对于 **用户会话** 级别的管理任务，守护进程的安装和设置不是必须的。授权总是仅限本地，前台程序将启动一个 **libvirtd** 守护进程的本地实例。

|维度|**System 模式（系统模式）**|**Session 模式（会话模式）**|
|---|---|---|
|**运行身份**|由**系统级 `libvirtd.service`** 守护进程管理，运行身份为 `root`（或 `libvirt` 系统用户）。|由**当前普通用户的会话进程**管理，运行身份为当前登录用户（如你的 Arch 用户名）。|
|**进程归属**|属于系统进程，独立于用户会话，开机自启，用户注销 / 登录不影响。|属于用户会话进程，用户注销后会话结束，对应的虚拟机进程会被终止（除非特殊配置）。|
|**配置文件路径**|虚拟机配置保存在 `/etc/libvirt/qemu/`（全局可见），网络配置在 `/etc/libvirt/qemu/networks/`。|虚拟机配置保存在 `~/.config/libvirt/qemu/`（仅当前用户可见），网络配置在 `~/.config/libvirt/qemu/networks/`。|
|**套接字路径**|系统套接字：`/var/run/libvirt/libvirt-sock`|用户套接字：`/run/user/$UID/libvirt/libvirt-sock`|
|**默认启用状态**|需手动启用 `libvirtd.service`（系统级服务），是 Arch 中 KVM 的**默认推荐模式**。|无需系统服务，用户直接运行 `virsh`/`virt-manager` 即可自动启动，无需额外配置。|

---

资源访问权限：

System 模式拥有 `root` 权限，可访问系统级硬件资源：
- 支持 PCI/USB/GPU 设备直通（如 VFIO 显卡直通、USB 设备 Passthrough）；
- 可创建桥接网络（与物理网卡桥接，让虚拟机接入局域网）；
- 可访问系统级存储（如 `/dev/sda` 物理磁盘、LVM 卷、NFS 共享存储）；
- 可绑定特权端口（如 80/443）。

Session 模式仅拥有当前用户的普通权限，资源访问受限：
- 不支持 PCI/GPU 直通（权限不足，无法访问 `/dev/vfio` 等设备）；
- 默认无网络（需手动创建用户级 NAT 网络，且不支持桥接网络）；
- 仅能访问用户目录下的文件（如 `~/VMs/` 中的磁盘镜像），无法访问系统级存储；
- 无法绑定特权端口。

---

网络能力：

System 模式下，自带默认的系统级 NAT 网络（`default` 网络），支持桥接网络、macvtap 等高级网络配置，是大多数场景的选择。

Session 模式下，Libvirt 默认使用用户模式网络（user mode network）：
- 完全在 QEMU 进程内部实现 TCP/IP 协议栈，不依赖内核网络功能
- 虚拟机可访问外部网络 (通过主机 NAT 转发)，但外部无法直接访问虚拟机
- 性能较差 (所有网络数据包需在用户空间处理)
- 不支持 ICMP(无法在虚拟机中使用 ping 命令)
- 虚拟机会获得一个私有 IP(通常是 10.0.2.15)，主机在同一子网的 IP 为 10.0.2.2

---

 虚拟机持久化与可见性

System 模式：虚拟机配置是全局持久化的，重启系统后虚拟机列表仍存在，所有有权限的用户（加入 `libvirt` 组的用户）均可查看和管理。

Session 模式：虚拟机配置仅**对当前用户可见**，且依赖用户会话（用户注销后，虚拟机进程终止，配置虽保存在用户目录，但需重新启动会话才能访问）。

---

访问控制

System 模式：通过 `polkit` 控制普通用户的访问权限（如加入 `libvirt` 组的用户可免 `sudo` 管理），支持细粒度的权限配置。

Session 模式：仅当前用户可访问，无需额外权限配置，无多用户共享能力。

## 授权

给当前用户授权 libvirt 和 kvm 组的权限，用于访问相关资源

```bash
sudo usermod $USER -aG libvirt,kvm 
```

然后重新登陆生效

## 守护进程

用户会话级别管理可以跳过这一步：

```bash
systemctl start libvirtd
# or
systemctl enable --now libvirtd
```

## 测试

测试 libvirt 在系统级工作是否正常：

```bash
virsh -c qemu:///system
```

测试 libvirt 在用户会话级工作是否正常：

```bash
virsh -c qemu:///session
```

# 使用 Virt-manager 使用 Window 虚拟机

## 基础安装

![](20260615112158.png)
![](20260615112304.png)
![](20260615112416.png)
![](20260615112716.png)
![](20260615113151.png)

确认后继续前进，进行更详细的配置
![](20260615113312.png)

![](20260615113522.png)

![](20260615113740.png)

![](20260615113929.png)

添加 VirtIO 驱动以加速硬件驱动
![](20260615114220.png)

![](20260615115139.png)

然后左上角点开始安装，安装完系统后已经可以正常使用了。

但是原驱动比较烂，显示分辨率也只有 1200x800 非常模糊，所以加下来进行在虚拟机中安装 VirtIO 驱动

![](20260615115452.png)

安装完后可以在系统设置里选择更高的分辨率了。

## 添加硬件

基础安装一般只自带键盘、鼠标或者是触摸板等硬件设备。
如果需要使用摄像头等硬件，需要手动添加。
路径：添加硬件 -> pci 主机设备 -> 找到你的设备添加后应用

![](20260615135154.png)
