---
title: 在 Docker 中运行 aTrust
date: 2026-10-10 00:00:00
updated: 2026-10-10 00:00:00
description: 在 Docker 中运行 aTrust
categories:
- 其他
tags:
- Docker
- aTrust
---

# 在 Docker 中运行 aTrust

向公司申请了一台电脑用来日常办公。但是我主系统还是想用 ArchLinux，剩下的就交给 KVM Windows 吧（

虽然是这么想，但是全交给 KVM Windows 感觉还是太重量级了，这和直接使用 Windows 有啥区别。并且我发现公司里并没有啥软件是 Windows 定制专有的，所以直接在 Linux 办公的方案似乎是可行的。理论上只要装个 VPN（公司用的aTrust）就行了。yay 已经有人打包，但我并不想直接安装。首先是因为权限问题，一般这种都需要 root 权限来创建 TUN 接口、改路由表、加 iptables 规则，很可能影响我现有的环境。然后是这个软件的公司是深信服，我对这家公司印象不是特别好，怕它到处拉屎修改我的文件。

所以我的方案是在 Linux 主机上使用 Docker 运行 Sangfor aTrust，通过 TigerVNC 登录 aTrust，再让宿主机浏览器、IDEA、Git 啥的软件通过容器中的 SOCKS5 或 HTTP 代理访问公司内网。

SOCKS5 由镜像附带的 Dante（`danted`）提供，HTTP 代理由 Tinyproxy 提供；aTrust 负责建立到公司网络的 VPN 连接。

```text
宿主机浏览器（配置应用代理）
    │ 127.0.0.1:1080 / 127.0.0.1:8888
    │ Docker 端口映射
    ▼
容器内的 Dante / Tinyproxy
    │ 发起到公司内网的连接
    ▼
容器网络命名空间内的 VPN 接口（此镜像配置为 utun7）
    │ aTrust 经宿主机 NAT 连接公司 VPN 服务端
    ▼
公司内网
```

这里有两层：**浏览器通过端口使用代理，容器内的 VPN 使用 TUN。** 端口映射只让浏览器连到代理服务，并不会自行建立到公司内网的通道。对于本文使用的 `hagb/docker-atrust` 镜像，仍需提供 `/dev/net/tun` 和 `NET_ADMIN`；这不要求把宿主机默认路由切到 VPN，也不需要在宿主机的网络命名空间创建 VPN 接口。

`/dev/net/tun` 是 Linux 内核提供的 TUN/TAP 设备入口。容器共享宿主机内核，通过这个入口在自己的网络命名空间创建 TUN 接口。

## 一、准备条件

需要准备：

- Docker 和 Docker Compose；
- 宿主机可用的 `/dev/net/tun` 设备，供容器创建 VPN 虚拟接口。

检查 TUN 设备（在宿主机终端执行）：

```bash
ls -l /dev/net/tun
```

如果设备不存在，可以尝试：

```bash
sudo modprobe tun
```

## 二、Docker Compose 配置

创建目录：

```bash
mkdir -p ~/docker/atrust
cd ~/docker/atrust
```

Compose 配置如下：

```yaml
services:
  atrust:
    image: hagb/docker-atrust:latest
    container_name: atrust
    restart: unless-stopped

    # 容器内创建 VPN 虚拟接口所需，和宿主机浏览器的代理设置是两回事
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN

    # 只允许宿主机访问，不暴露到局域网
    ports:
      - "127.0.0.1:5901:5901"   # VNC 图形界面
      - "127.0.0.1:1080:1080"   # SOCKS5 代理
      - "127.0.0.1:8888:8888"   # HTTP 代理
      - "127.0.0.1:54631:54631" # aTrust Web 认证通信端口

    volumes:
      - ./data:/root

    environment:
      PASSWORD: vnc
      URLWIN: "1"
      EXIT: "1"

    sysctls:
      - net.ipv4.conf.default.route_localnet=1
```

`devices` 将 TUN 设备入口提供给容器；`NET_ADMIN` 允许容器管理自身网络命名空间内的接口、路由和防火墙。本配置使用 Docker 默认的独立容器网络，没有使用 `network_mode: host`。

`PASSWORD` 是 VNC 密码，可以改成自己的密码，建议不超过 8 位。端口绑定到 `127.0.0.1`，表示只有宿主机可以访问 VNC 和代理。`EXIT: "1"` 配合 `restart: unless-stopped`，让 aTrust 前端退出时由 Docker 重启整个容器，而不是在容器内不断重启前端和后端；原因和验证方法见下文“前端反复崩溃、CPU 和温度升高”。

## 三、启动 aTrust 容器

创建新数据目录：

```bash
mkdir -p ./data
chmod 755 ./data
```

检查 Compose 配置：

```bash
sudo docker compose config
```

拉取镜像并启动：

```bash
sudo docker compose pull
sudo docker compose up -d
```

查看状态：

```bash
sudo docker compose ps
```

正常情况下应看到容器状态为 `Up`。查看最近日志(可能有一部分报错，但是不影响使用)：

```bash
sudo docker compose logs --tail=100 atrust
```

日志中出现下面内容，表示 VNC 服务已启动：

```text
New Xtigervnc server ... on port 5901
```

## 四、安装并连接 TigerVNC

Arch Linux 安装 TigerVNC：

```bash
sudo pacman -S tigervnc
```

连接 Docker 容器中的 VNC 桌面：

```bash
vncviewer 127.0.0.1::5901
```

注意这里是两个冒号：

```text
127.0.0.1::5901
```

VNC 密码就是 Compose 文件中的 `PASSWORD`，本例为：

```text
vnc
```

连接后可以看到容器里的 Linux 图形桌面。

## 五、登录 aTrust

在 VNC 桌面中启动 aTrust，完成登录。

登录成功后，aTrust 会在容器内部建立 VPN 连接。本镜像使用的 VPN 接口名为 `utun7`，可以在宿主机终端检查容器内的接口和路由：

```bash
sudo docker exec atrust ip addr show dev utun7
sudo docker exec atrust ip route
```

## 六、配置宿主机代理

容器启动时会启动代理服务；完成 aTrust 登录后，代理才能利用 VPN 访问受授权的公司内网资源。宿主机上的访问地址为：

| 类型 | 地址 |
|---|---|
| SOCKS5 | `127.0.0.1:1080` |
| HTTP | `127.0.0.1:8888` |


### Firefox

打开：

```text
设置 → 网络设置 → 设置 → 手动配置代理
```

填写：

```text
SOCKS 主机：127.0.0.1
端口：1080
```

并勾选：

```text
使用 SOCKS v5 时代理 DNS
```

如果只能使用 HTTP 代理，则填写：

```text
HTTP 代理：127.0.0.1
端口：8888
```

### Chromium/Chrome

使用 SOCKS5 启动：

```bash
chromium --proxy-server="socks5://127.0.0.1:1080"
```

使用 HTTP 代理启动：

```bash
chromium --proxy-server="http://127.0.0.1:8888"
```

也可以使用浏览器代理扩展，只对公司内网域名启用代理。

### 其他工具

如ssh,git,lftp等工具，如果需要访问内网也需要单独设置，此处不展开。

## 七、测试代理

使用设置了代理的浏览器，访问公司内网。

## 八、常用 Docker 操作

查看状态：

```bash
sudo docker compose ps
```

查看最近日志：

```bash
sudo docker compose logs --tail=100 atrust
```

持续查看日志：

```bash
sudo docker compose logs -f atrust
```

按 `Ctrl+C` 只会退出日志查看，不会停止容器。

停止并删除容器：

```bash
sudo docker compose down
```

重新创建：

```bash
sudo docker compose up -d --force-recreate
```

重启：

```bash
sudo docker compose restart
```

不要随意使用：

```bash
sudo docker compose down -v
```

`-v` 会额外删除关联的匿名卷和 Compose 声明的命名卷（外部卷除外），但不会删除 `./data:/root` 这类绑定挂载的宿主机 `./data` 目录。日常停止或重建无需这个选项。

## 九、常见日志说明

### `ulimit` 警告

```text
ulimit: open files: cannot modify limit: Operation not permitted
```

这是镜像内部启动脚本尝试提高文件描述符上限产生的提示。官方运行方式没有额外设置 `ulimits`。只要容器状态是 `Up`、VNC 可以连接、代理可以访问公司内网，通常可以暂时忽略。

### `Failed to connect to bus`

```text
Failed to connect to bus: No such file or directory
```

容器内没有完整的 systemd/dbus，而 aTrust 的部分脚本会尝试调用它们，因此可能出现该提示。

### 前端反复崩溃、CPU 和温度升高

本机曾遇到：容器运行时温度由约 60°C 升到约 80°C，`docker stats` 中 CPU 多次达到 50%～165%，内存和进程数也反复波动；停止容器后温度回到约 60°C。近期日志反复出现：

```text
/usr/local/bin/start-sangfor.sh: line 22: ... Segmentation fault (core dumped) $VPN_UI --no-sandbox
/usr/local/bin/start-sangfor.sh: line 3: ... Killed "$@"
```

这说明 aTrust 图形前端 `aTrustTray` 崩溃，而镜像的启动脚本会循环启动前端并清理、重启后端；VNC 登录后可能黑屏，即使关掉 TigerVNC 查看器，容器仍可能继续耗 CPU。曾同时出现 `/tmp/aTrustShell.conf: Permission denied`：该文件属于 `root`，改为 `sangfor` 所有后权限报错消失，但段错误和 CPU 峰值依旧，因此不能把改文件所有者当成此故障的根本修复。

本机采用的变通方案是在上面的 Compose `environment:` 中加入 `EXIT: "1"`，保留已有的 `restart: unless-stopped`。`EXIT` 让前端退出后容器退出，再交给 Docker 重启整个容器，避免镜像内部的快速重启循环。这一思路也见于[项目的相似问题报告](https://github.com/docker-easyconnect/docker-easyconnect/issues/489)，但不同环境下不保证有效。改动 Compose 后执行：

```bash
cd ~/docker/atrust
sudo docker compose config -q
sudo docker compose up -d --force-recreate
```

重建会短暂中断 VPN，但不会删除 `./data`；本次操作不需要 `docker compose down -v`。此次修改后，用户观察到 VPN 仍启用，温度回到约 60°C；**尚未通过长期内网连通性与重启次数验证稳定性**。使用时应通过 `127.0.0.1:1080` 或 `127.0.0.1:8888` 代理访问实际公司内网页面，并隔几分钟比较：

```bash
sudo docker inspect atrust --format '重启次数={{.RestartCount}}'
sudo docker stats --no-stream atrust
```

如果重启次数不断增加、内网访问中断或温度再次升高，说明问题仍在；`EXIT` 只改变重启方式，不修复 `aTrustTray` 的段错误。可从 Compose 中移除 `EXIT: "1"`，再次执行 `sudo docker compose up -d --force-recreate` 回退。不要为了压住症状就盲目添加 `privileged: true`、放宽整个 `/tmp` 的权限或提高文件描述符上限。

### VNC 启动成功

```text
New Xtigervnc server ... on port 5901
```

说明可以使用：

```bash
vncviewer 127.0.0.1::5901
```

连接图形界面。

## 十、日常使用流程

```bash
cd ~/docker/atrust
sudo docker compose up -d
vncviewer 127.0.0.1::5901
```

然后：

1. 使用 TigerVNC 登录容器桌面；
2. 在 aTrust 中登录公司 VPN；
3. 在宿主机浏览器中配置 `127.0.0.1:1080` SOCKS5 代理；
4. 访问公司内部网站或系统；
5. 使用 `docker compose ps` 和 `docker compose logs` 检查运行状态。
