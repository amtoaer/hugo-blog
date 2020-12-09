---
title: "为 Arch Linux 配置安全启动"
date: 2020-12-09T10:39:19+08:00
draft: true
---

最近为自己的`Arch Linux`配置了安全启动( `security boot` )，在此记录一下流程以及遇到的问题。

## 实现安全启动

关于安全启动的实现方法，[Arch Wiki](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface/Secure_Boot#Implementing_Secure_Boot)内有完整的描述，但由于其采用wiki方式写作，对流程描述较为复杂。本文旨在描述实现安全启动的最简方法，跳过更加详细深入的概念性内容。

**注：本文仅描述了为Arch Linux设置安全启动，关于进一步的实现与Windows安全启动共存，请自行阅读Arch Wiki[相应条目](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface/Secure_Boot#Microsoft_Windows)。**

### 生成Key

安全启动的实现包括四种Key：

+ Platform Key(PK)
+ Key Exchange Key(KEK)
+ Signature Database(db)
+ Forbidden Signatures Database(dbx)

为了正常使用安全启动，我们至少需要前三种Key。为了方便，我们可以使用脚本自动生成：

```bash
# 后文自动处理内核更新等操作的 sbupdate 工具默认使用位于/etc/efi-keys的Key。

# 为了方便，这里直接创建此目录并在该目录生成Key

mkdir /etc/efi-keys
cd !$
curl -L -O https://www.rodsbooks.com/efi-bootloaders/mkkeys.sh
chmod +x mkkeys.sh
./mkkeys.sh
<随意输入一个嵌入在key中的名字，例如"Secure Boot">
```

### 自动签名

为了在内核、引导更新时自动签名，我们可以使用`sbupdate`工具。

#### 安装

通过如下命令安装：

```bash
yay -S sbupdate-git
```

#### 配置

安装后，我们需要为其写入配置，指明需要自动签名的内容。

打开/etc/sbupdate.conf，修改配置文件，以下给出我的设置样例以及选项说明：

```bash
# Key所在的目录
KEY_DIR="/etc/efi-keys"
# EFI引导分区位置
ESP_DIR="/boot"
# 生成的单独EFI引导文件位置
OUT_DIR=""
# 生成的单独EFI引导文件所携带的内核参数
CMDLINE_DEFAULT="loglevel=3 quiet nomce"
# 需要额外签名的文件，一般包括自己的启动加载器文件（例如grub，本例使用systemd-boot）、内核等内容
EXTRA_SIGN=('/boot/EFI/BOOT/BOOTX64.EFI' '/boot/EFI/systemd/systemd-bootx64.efi' '/boot/vmlinuz-linux-zen')
```

特别说明，`sbupdate`工具会为内核生成一个直接的引导文件（位于`${ESP_DIR}/${OUT_DIR}`，文件名一般为`linux-signed.efi`），其中包含了内核参数，但个人在本机通过`EFI`直接引导以及使用`systemd-boot`间接引导该文件均告失败，因此不详细叙述。（*当它没有就行啦，反正咱有咱的boot loader*）

**