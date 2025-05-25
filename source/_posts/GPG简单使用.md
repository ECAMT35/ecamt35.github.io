---
title: GPG简单使用
date: 2025-05-18 00:00:00
description: 这是仅你所知的秘密; 这是证明我的唯一方式
categories: 
- GNU/Linux
tags:
- GPG
---

参考：
https://wiki.archlinux.org/title/GnuPG

介绍：GnuPG 是完整实现了 [RFC4880](https://tools.ietf.org/html/rfc4880)（即PGP）所定义的 [OpenPGP](https://openpgp.org/about/) 标准的自由软件。GnuPG 可以加密和签名你的数据和通讯信息，包含一个通用的密钥管理系统以及用于各种公钥目录的访问模块。GnuPG，简称 GPG，是一个易于与其它程序整合的命令行工具，拥有很多前端程序和函数库。GnuPG 还支持 S/MIME 和 Secure Shell (ssh)。

# 快速使用

## 加密解密

使用非对称加解密：

```bash
gpg -r ECAMT -e 1.png 
```

使用对称加解密：

```bash
gpg -c 1.png 
```

查看：
```bash
ls -l
总计 12960
-rw-r--r-- 1 ECAMT ECAMT 6633916 12月 8日 1.png
-rw-r--r-- 1 ECAMT ECAMT 6634809  5月18日 1.png.gpg

---
# 使用非对称加密
file 1.png.gpg 
1.png.gpg: data

# 使用对称加密
file 1.png.gpg 
1.png.gpg: PGP symmetric key encrypted data - AES with 256-bit key salted & iterated - SHA256 .
```

无论哪种方式加都是使用 `-d` 解密（使用 `>` 代替 `-o` 输出到 `2.png`）：
```bash
gpg -d 1.png.gpg > 2.png
```

---

使用例：加密归档文件

可以一口气完成归档压缩加密：

```bash
bsdtar -cf - -J <path/direction> | gpg -c > <xx.gpg>
```

>压缩应在加密前：加密后的数据是随机字节，难以被压缩。因此，若需压缩，务必先压缩再加密。

>仍建议分开归档压缩和加密的步骤，如 `xz`（LZMA算法）在压缩数据时，其行为会受到输出目标的类型（文件 vs 管道）的影响，使用管道传输传递给gpg时，xz 使用流式压缩模式，每个数据块可能包含额外的头信息和校验值，导致最后的体积略大。

## 签名校验

签名主要用于验证数据的完整性、来源真实性以及防止数据被篡改。

常用签名：
```bash
gpg --output <doc.sig> --sign <doc>
```

校验：
```bash
gpg --verify <doc.sig>
```

---

使用例：git提交并签名

提交时签名：

```bash
git commit -S --gpg-sign=ECAMT
```

查看（需要提前导入公钥）：

```bash
git log --show-signature
```

git设置为指定自动私钥签名（仍需要 `-S` 参数）：
```bash
git config set user.signingKey <user-id>

# 校验
git config get user.signingKey <user-id>
```

git设置每次自动签名：
```bash
git config commit.gpgsign true
```

电子邮件完整验证也能用到。

# 配置

# 主目录

GnuPG 套件将密钥环和私钥存储在 GnuPG 主目录，并从中读取配置。默认路径为 `~/.gnupg`。有两种方法可以改变主目录的路径：

- 设置 `$GNUPGHOME` [环境变量](https://wiki.archlinuxcn.org/wiki/%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F "环境变量")。
- 使用 `--homedir` 参数，如 `$ gpg --homedir </path/to/dir>`

默认情况下，**主目录的[权限](https://wiki.archlinuxcn.org/wiki/Permissions "Permissions")设置为 `700`，其包含的文件的权限设置为 `600`**。只有目录的所有者有权读取、写入和访问文件。这是出于安全目的，不应更改。如果此目录或其中的任何文件不遵循此安全措施，您将收到有关不安全文件和主目录权限的警告。

## 配置文件

GnuPG 的所有行为都可以通过命令行参数进行配置。对于您希望成为默认参数的参数，可以将它们添加到相应的配置文件中：

- gpg 检查 `gnupg_home/gpg.conf`（用户）和 `/etc/gnupg/gpg.conf`（全局）。由于 gpg 是 GnuPG 的主要入口点，因此大部分感兴趣的配置都在这里。请参阅 [GPG 选项](https://www.gnupg.org/documentation/manuals/gnupg/GPG-Options.html) 获取可能的选项。
- dirmngr 会检查 `gnupg_home/dirmngr.conf` 和 `/etc/gnupg/dirmngr.conf` 两个配置文件。dirmngr 是由 gpg 内部调用的程序，用于访问 PGP 密钥服务器。请参阅 [Dirmngr 选项](https://www.gnupg.org/documentation/manuals/gnupg/Dirmngr-Options.html) 以了解可能的选项。

这两个配置文件涵盖了常见用例，但 GnuPG  套件中还有更多带有自己选项的辅助程序。请参阅 [GnuPG 手册](https://www.gnupg.org/documentation/manuals/gnupg/index.html) 获取详细列表。

创建所需的文件，并按照 主目录 章节中讨论的方法设置其权限为600。

在这些文件中添加任何你想要的长选项。不要写两个破折号，只需写选项的名称和所需的参数。例如，要始终使GnuPG在特定路径上使用密钥环，就像使用 `gpg --no-default-keyring --keyring keyring-path ...` 调用它一样： 

```
gnupg_home/gpg.conf (或 /etc/gnupg/gpg.conf)
---
no-default-keyring
keyring <keyring-path>
```

## 新用户的默认选项

要给新建用户设定一些默认选项，把配置文件放到 `/etc/skel/.gnupg/`。系统创建新用户时，就会把文件复制到 GnuPG 目录。还有一个 _addgnupghome_ 命令可以为已有用户创建新 GnuPG 主目录：

```bash
addgnupghome user1 user2
```

此命令会检查 `/home/user1/.gnupg/` 和 `/home/user2/.gnupg/`，并从 skeleton 目录复制文件过去。具有已存在的 GnuPG 主目录的用户只需跳过即可。

# 用法

>注意：
>- 如果需要一个 _`user-id`_，可以使用 key ID、指纹、用户名或电邮地址的部分等替代，GnuPG 对此的处理很灵活。
>- 如果需要一个 _`key-id`_，可以给命令加上 `--keyid-format=long` 选项来查询。例如，如果想要查看主密匙，可以使用`gpg --list-secret-keys --keyid-format=long user-id`命令，_key-id_ 是和 _sec_ 同一行的十六进制散列值。

## 创建密钥对

用下面命令创建一个密钥对：

```bash
gpg --full-gen-key
```

使用 `--expert` 选项可以选择其它的加密算法，尤其是较新的[ECC（椭圆曲线加密）](https://zh.wikipedia.org/wiki/%E6%A4%AD%E5%9C%86%E6%9B%B2%E7%BA%BF%E5%AF%86%E7%A0%81%E5%AD%A6 "zhwp:椭圆曲线密码学")。

命令执行后会需要用户回答一些问题，大部分用户应该需要的是：

- 默认的“RSA 和 RSA”用于加密和解密。
- 默认的密钥长度，即 3072。增大长度到 4096“成本极高，但获益很少”。[这个帖子说明了为何 GPG 不默认使用 RSA-4096](https://www.gnupg.org/faq/gnupg-faq.html#no_default_of_rsa4096)。
- 过期日期。大部分用户可以选择一年。这样即使无法访问密钥环，用户也知道密钥已经过期。如果有需要，可以不重新签发密钥就延长过期时间。
- 用户名和电子邮件。可以给同样的密钥不同的身份，比如给同一个密钥关联多个电子邮件。
- **不填写**可选注释。注释字段并没有被[很好地定义](https://lists.gnupg.org/pipermail/gnupg-devel/2015-July/030150.html)，作用有限。
- 一个安全的密钥口令。可参考[如何选择安全的密码](https://wiki.archlinuxcn.org/wiki/Security#选择安全的密码 "Security")。

>注意：任何导入密钥的人都可以看到这里的用户名和电子邮件地址。

>提示：较简单的 `--gen-key` 选项对密钥类型、密钥长度、过期时间均使用默认值，仅询问姓名和电邮地址。

##  查看密钥

查看公钥：

```bash
gpg --list-keys
```

查看私钥:

```bash
gpg --list-secret-keys
```

## 导出公钥

GPG 的主要用途是通过公钥加密信息以确保其私密性。你可以分发自己的公钥，而其他人通过该公钥加密发给你的信息。而你的私钥必须**始终**保密，否则将会威胁信息的私密性。相关内容，请参见[公开密钥加密](https://zh.wikipedia.org/wiki/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86 "zhwp:公开密钥加密")。

所以其他人需要有你的公钥才能给你发加密信息。

以下命令可生成公钥的 ASCII 版本（`--armor` 参数）（例如用于以电子邮件发布）：

```bash
gpg --export --armor --output <public-key.asc> <user-id>
```

此外，还可以通过[密钥服务器](https://wiki.archlinuxcn.org/wiki/GnuPG#使用公钥服务器)分发公钥。

>提示：
>- 使用 `--no-emit-version` 可以避免打印版本号，通过配置文件也可以进行此设置。
>- 可以省略 `user-id` 以导出密钥环内所有的公钥。这可以用来分享多个身份，或是将其导入到另一个程序，比如 [Thunderbird](https://wiki.archlinuxcn.org/wiki/Thunderbird#Use_OpenPGP_with_external_GnuPG "Thunderbird")。

## 导入公共密钥

要给其他人发送加密信息，或者验证他们的签名，就需要他们的公钥。通过文件 `public.key` 导入公钥到密钥环：

```bash
gpg --import <public.key.asc>
```

此外，还可以通过[#密钥服务器](https://wiki.archlinuxcn.org/wiki/GnuPG#密钥服务器)导入公钥。

## 使用公钥服务器

### 发布公钥

你可以将你的公钥注册到一个公共的密钥服务器，这样其他人不用联系你就能获取到你的公钥：

```bash
gpg --send-keys <key-id>
```

>警告：一旦一个公钥被发送到密钥服务器，它就无法从服务器上删除。[这个网页](https://pgp.mit.edu/faq.html)解释了原因。

>注意：与公钥相关联的电邮地址一旦公开，可能会被垃圾邮件发送者盯上。请做好相应的防护措施。

### 搜索和接收公钥

要查询公钥的详细信息而不是导入，执行：

```bash
gpg --search-keys <user-id>
```

要导入一个公钥：

```bash
gpg --receive-keys <key-id>
```

要使用密钥服务器中的最新版本刷新/更新钥匙串：

```bash
gpg --refresh-keys
```

>警告：
>- 您应该通过将其指纹与所有者在独立来源（例如直接联系该人）上发布的指纹进行比较，以验证检索到的公钥的真实性。请参阅 [Wikipedia:Public key fingerprint](https://en.wikipedia.org/wiki/Public_key_fingerprint "wikipedia:Public key fingerprint") 获取更多信息。
>- 接收密钥时，建议使用长密钥 ID 或完整指纹。使用短密钥 ID 可能会导致冲突。所有具有短密钥 ID 的密钥都将被导入，参见 [在野外发现的伪密钥](https://lore.kernel.org/lkml/20160815153401.9EC2BADC2C@smtp.postman.i2p/)作为示例。

>提示：将 `auto-key-retrieve` 添加到 [GPG 配置文件](https://wiki.archlinuxcn.org/wiki/GnuPG#Configuration_files)中，将在需要时自动从密钥服务器获取密钥。这不会对安全性造成妥协，但可以被视为**侵犯隐私**；请参阅[gpg(1)](https://man.archlinux.org/man/gpg.1)中的"web bug"。

### 公钥服务器

常见的公钥服务器：

- [Ubuntu Keyserver](https://keyserver.ubuntu.com)：联盟式（federated）、没有验证、公钥不可删除。
- [Mailvelope Keyserver](https://keys.mailvelope.com)：中心式、验证电邮 ID、公钥可删除。
- [keys.openpgp.org](https://keys.openpgp.org)：中心式、验证电邮 ID、公钥可删除、没有第三方签名（即不支持信任网络）。

[维基百科（英文）](https://en.wikipedia.org/wiki/Key_server_\(cryptographic\)#Keyserver_examples "wikipedia:Key server (cryptographic)")上有更多的服务器。

备选公钥服务器可以在 配置文件 中的 `keyserver` 选项中注明，例如：

```
~/.gnupg/dirmngr.conf
---
keyserver hkp://keyserver.ubuntu.com
```

当常规服务器无法正常工作时，临时使用另一台服务器很方便。例如，可以通过以下方法实现：

```bash
gpg --keyserver <hkps://keys.openpgp.org/> --search-keys <user-id>
```

# 加密与解密

## 非对称加解密

在加密（参数`--encrypt`或`-e`）一个文件或一条信息给另外一个人（参数`--recipient`或`-r`）之前，你需要先导入他的公钥。如果你还没有创建自己的密钥对，请先创建。

要加密一个名为 doc 的文件：

```bash
gpg --recipient <user-id> --encrypt <doc>
```

要解密（参数 `--decrypt` 或 `-d`）一个用你的公钥加密的、名为 _doc_.gpg 的文件：

```bash
gpg --output <doc> --decrypt <doc.gpg>
```

_gpg_ 会提示你输入密钥口令，并将 _doc_.gpg 中的数据解密到 _doc_。如果你忽略了参数 `-o`（`--output`），_gpg_ 将会直接输出解密的信息。

>提示：
>- 使用参数 `--armor` 以 ASCII 编码的形式加密文件（适用于复制与粘贴文本文件格式的消息）。
>- 使用 `-R <user-id>` 或 `--hidden-recipient <user-id>` 代替 `-r` 可以不将收件人的指纹 ID 放入加密的消息中。这有助于隐藏收件人的信息，是针对流量分析的一个有限对策。
>- 使用 `--no-emit-version` 以避免打印版本号。也可将相应配置添加到你的配置文件中。
>- 你可以使用 GPG 将自己作为收件人来加密敏感文件，但是每次只能压缩一个文件——尽管你可以将多个文件压缩后再进行加密。如果需要加密一个目录或一整个文件系统，请参见 [Data-at-rest encryption#Available methods](https://wiki.archlinuxcn.org/wiki/Data-at-rest_encryption#Available_methods "Data-at-rest encryption")。

## 对称加解密

对称加密不需要生成密钥对，可用来简单地给文件加上密码。使用 `-c`/`--symmetric` 参数来进行对称加密：

```bash
gpg -c <doc>
```

下面的例子：

- 用口令给 `doc` 进行了对称加密
- 用 AES-256 加密算法对口令进行加密
- 用 SHA-512 摘要算法对口令进行打乱
- 打乱 65536 次

```bash
gpg -c --s2k-cipher-algo AES256 --s2k-digest-algo SHA512 --s2k-count 65536 doc
```

下面的命令可解密以口令对称加密的 `doc.gpg` 文件，并将解密的文档输出到同一目录下的 `doc` 文件中：

```bash
gpg --output <doc> --decrypt <doc>.gpg
```

解密时有时不需要输入密码，原因是 gpg-agent 缓存了。

## 目录操作

可用 [gpgtar(1)](https://man.archlinux.org/man/gpgtar.1) 对目录进行加密和解密。

加密：

```bash
gpgtar -c -o <dir.gpg> <dir>
```

解密：

```bash
gpgtar -d <dir.gpg>
```

# 签名

签名用于认证和时间戳文档。如果文档被修改，验证签名将失败。与使用公钥加密文档不同，签名是使用用户的私钥创建的。文档的接收者然后使用发送者的公钥验证签名。

## 签署文件

要签署文件，请使用`-s`/`--sign`标志：

```bash
gpg --output <doc.sig> --sign <doc>
```

`doc.sig`包含原始文件`doc`的压缩内容和以二进制格式表示的签名，但文件并未加密。但是，您可以将签名与加密结合使用。

## 以可读形式签名文件或消息

要签署文件而无需将其压缩为二进制格式，请使用：

```bash
gpg --output <doc>.sig --clearsign <doc>
```

在这里，原始文件`doc`的内容和签名以可读形式存储在`doc.sig`中。

## 创建独立的签名文件

要创建一个单独的签名文件，以便与文档或文件本身份开分发，请使用`--detach-sig`标志：

```bash
gpg --output <doc.sig> --detach-sig <doc>
```

在这里，签名存储在`doc.sig`中，但`doc`的内容不会存储在其中。这种方法常用于分发软件项目，以允许用户验证程序未被第三方修改。

## 验证签名

要验证签名，请使用`--verify`标志：

```bash
gpg --verify <doc.sig>
```

其中`doc.sig`是包含您要验证的签名的已签名文件。

如果您要验证一个已分离签名，验证时必须同时存在已签名的数据文件和签名文件。例如，要验证 Arch Linux 的最新 iso 文件，您可以执行以下操作：

```bash
gpg --verify <archlinux-version.iso.sig>
```

其中`archlinux-version.iso`必须位于相同的目录中。

您还可以使用第二个参数指定已签名的数据文件：

```bash
gpg --verify <archlinux-version.iso.sig> </path/to/archlinux-version.iso>
```

如果一个文件除了被签名外还被加密，只需[解密](https://wiki.archlinuxcn.org/wiki/GnuPG#加密和解密)该文件，其签名也将被验证。

# 密钥维护

## 备份你的私钥

用如下命令备份你的私钥。

```bash
gpg --export-secret-keys --armor --output <private-key.asc> <user-id>
```

请注意，上述命令将要求您输入密钥的密码。这是因为，否则任何获得上述导出文件访问权限的人都可以像您一样对文档进行加密和签名，而无需知道您的密码。

**警告：**

- 口令通常是密钥安全方面最薄弱的环节。最好把导出的文件放在另一个系统或者设备里，比如物理保险柜或者加密驱动器中。这是当你遇到设备被盗、磁盘故障等情况时恢复对密钥控制权的唯一安全措施。
- 这种备份方式有一些安全局限性，这篇文章 [https://web.archive.org/web/20210803213236/https://habd.as/post/moving-gpg-keys-privately/](https://web.archive.org/web/20210803213236/https://habd.as/post/moving-gpg-keys-privately/) 中有关于用 _gpg_ 备份和导入密钥的更加安全的办法。

用如下命令导入你的私钥备份

```bash
gpg --import <private-key.asc>
```

**提示：** 你可以用 [Paperkey](https://wiki.archlinuxcn.org/wiki/Paperkey "Paperkey") 来把私钥导出为明文文本或条形码，并打印出来存档。

## 备份你的吊销证书

若使用gpg公钥服务器

生成新密钥对的时候会同时生成吊销证书，默认存放在 `~/.gnupg/openpgp-revocs.d/` 下，证书的文件名是对应的密钥的指纹。 你也可以用以下命令手动生成吊销证书：

```bash
gpg --gen-revoke --armor --output <revcert.asc> <user-id>
```

如果密钥丢失或泄露，此证书可用于 [#吊销密钥](https://wiki.archlinuxcn.org/wiki/GnuPG#吊销密钥)。如果你无法访问密钥，则无法使用上述命令生成新的吊销证书，那么备份将非常有用。吊销证书很短，你可以把他打印出来然后在需要使用的时候手动输入到电脑里。

>警告：任何能接触到吊销证书的人都可以吊销你的密钥对，而且无法撤消。所以请像保护私钥一样保护你的吊销证书。

## 编辑你的密钥

运行 `gpg --edit-key <user-id>` 命令将会出现一个菜单，该菜单使你能够执行大部分密钥管理相关的任务。

在编辑密钥子菜单中输入 `help` 命令可以显示完整的命令列表。以下是一些有用的命令：

> passwd       # 修改密码短语
> clean        # 压缩任何不再可用的用户ID（例如已撤销或已过期）
> revkey       # 撤销密钥
> addkey       # 向该密钥添加子密钥
> expire       # 更改密钥过期时间
> adduid       # 添加附加的名称、注释和电子邮件地址
> addphoto     # 向密钥添加照片（必须是JPG格式，推荐大小为240x288，当提示时输入完整路径）

>提示：如果你有多个电子邮件账户，你可以使用 `adduid` 命令将每个账户都添加为一个身份。然后你可以将你最喜欢的账户设置为 `primary`。

## 删除密钥对

删除私钥：

```bash
gpg --delete-secret-key <user-id>
```

删除私钥后，需单独删除公钥：

```bash
gpg --delete-key <user-id>
```

校验：

```bash
gpg --list-keys
gpg --list-secret-keys
```

# gpg-agent

_gpg-agent_ 主要用作守护进程，用于请求和缓存密钥链的密码。这在外部程序（如邮件客户端）使用 GnuPG 时十分有用。 [gnupg](https://archlinux.org/packages/?name=gnupg)包 带有默认自动启动的 [systemd/用户](https://wiki.archlinuxcn.org/wiki/Systemd/%E7%94%A8%E6%88%B7 "Systemd/用户")套接字。这些套接字分别是 `gpg-agent.socket`、`gpg-agent-extra.socket`、`gpg-agent-browser.socket`、`gpg-agent-ssh.socket` 和 `dirmngr.socket`。

- _gpg_ 使用 `gpg-agent.socket` 连接到 _gpg-agent_ 守护进程。
- `gpg-agent-extra.socket` 的作用是在本地建立一个转发自远程系统的 Unix 域套接字。这样就可以在远程系统上使用 _gpg_，而无需向远程系统公开私钥。有关详细信息，请参阅 [gpg-agent(1)](https://man.archlinux.org/man/gpg-agent.1)。
- `gpg-agent-browser.socket` 允许 Web 浏览器访问 _gpg-agent_ 守护进程。
- [SSH](https://wiki.archlinuxcn.org/wiki/SSH "SSH") 使用 `gpg-agent-ssh.socket` 缓存 _ssh-add_ 程序添加的 [SSH keys](https://wiki.archlinuxcn.org/wiki/SSH_keys "SSH keys")。有关必要的配置，请参阅 [#SSH agent](https://wiki.archlinuxcn.org/wiki/GnuPG#SSH_agent)。
- `dirmngr.socket` 启动一个 GnuPG 守护进程来处理与 keyserver 的连接。

>注意：如果您没有使用默认的 GnuPG [#目录位置](https://wiki.archlinuxcn.org/wiki/GnuPG#目录位置), 您需要[编辑](https://wiki.archlinuxcn.org/wiki/Systemd#修改现存单元文件 "Systemd")所有套接字文件让其使用 `gpgconf --list-dirs` 的值。 套接字名称使用 [非默认 GnuPG 主目录的哈希](https://github.com/gpg/gnupg/blob/260bbb4ab27eab0a8d4fb68592b0d1c20d80179c/common/homedir.c#L710-L713)，您可以硬编码它不用担心它的改变。

## 配置

gpg-agent 用 `~/.gnupg/gpg-agent.conf` 文件配置。配置选项列在 [gpg-agent(1)](https://man.archlinux.org/man/gpg-agent.1) 中。例如，您可以更改默认密钥的缓存 ttl：

```
~/.gnupg/gpg-agent.conf
---
default-cache-ttl 3600
```

>提示：要缓存整个会话的密码(passphrase)，请运行以下命令：

```bash
/usr/lib/gnupg/gpg-preset-passphrase --preset XXXXX
```

其中 XXXXX 是 keygrip。您可以在运行 `gpg --with-keygrip -K` 时获取它的值。密码(passphrase)将一直保存到 `gpg-agent` 重新启动为止。如果设置了 `default-cache-ttl` 值，会优先采用它。

在 Linux 中，为了允许预设的密码短语，需要通过使用 `--allow-preset-passphrase` 启动 gpg-agent，或在 `~/.gnupg/gpg-agent.conf` 中设置`allow-preset-passphrase`。

## 重新加载 gpg-agent

在修改完配置之后，用 _gpg-connect-agent_ 重新加载 gpg-agent：

```bash
gpg-connect-agent reloadagent /bye
```

该命令应该输出 `OK`。

但是在某些情况下，只是重新启动可能不够，比如当 `keep-screen` 被添加到 gpg-agent 配置中时。在这种情况下，您首先需要终止正在进行的 gpg-agent 进程，然后按上述方法重新启动它。

## pinentry

`gpg-agent` 可以在 `pinentry-program` 中设定，以便使用特定的 [pinentry](https://archlinux.org/packages/?name=pinentry)包 用户界面来提示用户输入(passphrase)。例如：

```
~/.gnupg/gpg-agent.conf
---
pinentry-program /usr/bin/pinentry-curses
```

还有其他 pinentry 程序可选，参考 `pacman -Ql pinentry | grep /usr/bin/` 的输出结果。

>提示：
>- 为了使用 `/usr/bin/pinentry-kwallet` 您需要安装软件包 [kwalletcli](https://aur.archlinux.org/packages/kwalletcli/)AUR。
>- 所有的默认 pinentry 程序（除了 `/usr/bin/pinentry-emacs`）都支持 [DBus Secret Service API](https://specifications.freedesktop.org/secret-service/) ，它允许通过一个兼容的管理器(如 [GNOME Keyring](https://wiki.archlinuxcn.org/wiki/GNOME_Keyring "GNOME Keyring") 或 [KeePassXC](https://wiki.archlinuxcn.org/wzh/index.php?title=KeePass&action=edit&redlink=1 "KeePass（页面不存在）"))记住密码。

记得在修改完配置后要[#重新加载 gpg-agent](https://wiki.archlinuxcn.org/wiki/GnuPG#重新加载_gpg-agent)。

## 缓存密码

`max-cache-ttl` 和 `default-cache-ttl` 定义 gpg-agent 的密码缓存时间（秒）。要在会话中只输入一次密码，设置一个非常高的值即可，例如：

```
gpg-agent.conf
---
max-cache-ttl 60480000
default-cache-ttl 60480000
```

对于 SSH 仿真模式下的密码缓存，需要设置 `default-cache-ttl-ssh` 和 `max-cache-ttl-ssh`，例如：

```
gpg-agent.conf
---
default-cache-ttl-ssh 60480000
max-cache-ttl-ssh 60480000
```

## Unattended passphrase

从 GnuPG 2.1.0 开始，需要使用 gpg-agent 和 pinentry，这可能会破坏使用 `--passphrase-fd 0` 命令行选项从 STDIN 传入的密码短语的向后兼容性。为了拥有与旧版本相同类型的功能，必须做两件事：

首先，编辑 gpg-agent 配置允许 _loopback_ pinentry 模式 :

```bash
~/.gnupg/gpg-agent.conf
---
allow-loopback-pinentry
```

如果 gpg-agent 正在运行，[重新加载](https://wiki.archlinuxcn.org/wiki/GnuPG#重新加载_gpg-agent)它使配置生效。

其次，要么应用程序需要更新，以包括一个命令行参数来使用回环模式，例如：

```bash
gpg --pinentry-mode loopback ...
```

如果不可能这样做，则可以将选项添加到配置中：

```bash
~/.gnupg/gpg.conf
---
pinentry-mode loopback
```

>上游作者指出，在 `gpg.conf` 中设置 `pinentry-mode loopback` 可能会破坏其他用法，如果可能，最好使用命令行选项。
>https://dev.gnupg.org/T1772


