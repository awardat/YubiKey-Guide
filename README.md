> **译者注**：为了避免在实际操作的时候产生偏差或者参阅[原文](https://github.com/drduh/YubiKey-Guide)的时候看一句查一句，进行了简单的翻译，主要为机翻，人工对少量语句进行了调整。如果有看不懂的地方以原文为准。
> 本文档目前仅作为个人学习GPG的记录，未对所有过程进行验证，不保证指南中的内容全部可用。
> 根据个人使用情况进行更新，不保证跟随原文更新翻译。

这是使用[YubiKey](https://www.yubico.com/products/yubikey-hardware/)作为[智能卡](https://security.stackexchange.com/questions/38924/how-does-storing-gpg-ssh-private-keys-on-smart-cards-compare-to-plain-usb-drives)用于存储GPG加密、签名和身份验证密钥，以及用于SSH的指南。本文档中的许多原则也适用于其他智能卡设备。

与存储在磁盘上的基于文件的密钥不同，YubiKey上存储的密钥[不可导出](http://web.archive.org/web/20201125172759/https://support.yubico.com/hc/en-us/articles/360016614880-Can-I-Duplicate-or-Back-Up-a-YubiKey-)。并且更方便日常使用，在使用PIN解锁后只需触摸YubiKey即可解锁SSH/GPG密钥，而无需记住并输入密码。所有签名和加密操作都发生在卡上，而不是在操作系统内存中。

**提示** [drduh/Purse](https://github.com/drduh/Purse) 是一个密码管理器，它使用 GPG 和 YubiKey 来安全地存储和读取凭据。

> **安全说明**：如果您在 2021 年 1 月之前遵循本指南，您的 GPG *PIN* 和 *Admin PIN* 可能会保持其默认值（分别为“123456”和“12345678”）。 这将允许攻击者使用您的 Yubikey 或重置您的 PIN。 请参阅[更改 PIN](#change-pin) 部分，了解有关如何更改 PIN 的详细信息。

如果您有意见或建议，请在**原作者** GitHub 上提交[Issue](https://github.com/drduh/YubiKey-Guide/issues)。

# 目录

- [选购](#选购)
- [准备环境](#准备环境)
- [所需软件](#所需软件)
  - [Debian and Ubuntu](#debian-and-ubuntu)
  - [Fedora](#fedora)
  - [Arch](#arch)
  - [RHEL7](#rhel7)
  - [NixOS](#nixos)
  - [OpenBSD](#openbsd)
  - [macOS](#macos)
  - [Windows](#windows)
- [熵](#熵)
  - [YubiKey](#yubikey)
  - [OneRNG](#onerng)
- [创建密钥](#创建密钥)
  - [临时工作目录](#临时工作目录)
  - [加固配置](#加固配置)
- [主密钥](#主密钥)
- [使用现有密钥签名](#使用现有密钥签名)
- [子密钥](#子密钥)
  - [签名密钥](#签名)
  - [加密密钥](#加密)
  - [身份验证密钥](#身份验证)
  - [添加额外的身份](#添加额外的身份)
- [校验](#校验)
- [导出密钥](#导出密钥)
- [吊销证书](#吊销证书)
- [备份](#备份)
- [导出公钥](#导出公钥)
- [配置智能卡](#配置智能卡)
  - [启用KDF](#启用kdf)
  - [修改PIN](#修改pin)
  - [设置信息](#设置信息)
- [传输密钥](#传输密钥)
  - [签名密钥](#签名密钥)
  - [加密密钥](#加密密钥)
  - [身份验证密钥](#身份验证密钥)
- [校验智能卡](#校验智能卡)
- [多个YubiKey](#多个yubikey)
  - [在多个Yubikey间切换](#在多个yubikey间切换)
- [清理](#清理)
- [使用密钥](#使用密钥)
- [密钥轮替](#密钥轮替)
  - [设置环境](#设置环境)
  - [更新子密钥](#更新子密钥)
  - [轮替子密钥](#轮替子密钥)
- [添加符号](#添加符号)
- [SSH](#ssh)
  - [创建配置](#创建配置)
  - [替换代理程序](#替换代理程序)
  - [复制公钥](#复制公钥)
  - [(可选)为身份文件配置保存公钥](#可选为身份文件配置保存公钥)
  - [使用公钥认证连接](#使用公钥认证连接)
  - [导入SSH密钥](#导入SSH密钥)
  - [远程主机(SSH代理转发)](#远程主机SSH代理转发)
    - [使用ssh-agent](#使用ssh-agent)
    - [使用S.gpg-agent.ssh](#使用Sgpg-agentssh)
    - [链式SSH代理转发](#链式ssh代理转发)
  - [GitHub](#github)
  - [OpenBSD](#openbsd)
  - [Windows](#windows)
    - [WSL](#wsl)
      - [使用ssh-agent还是S.weasel-pegant](#使用ssh-agent还是sweasel-pegant)
      - [先决条件](#先决条件)
      - [WSL配置](#wsl配置)
      - [远程主机配置](#远程主机配置)
  - [macOS](#macos)
- [远程主机(GPG代理转发)](#远程主机GPG代理转发)
  - [旧发行版的步骤](#旧发行版的步骤)
  - [链式GPG代理转发](#链式gpg代理转发)
- [使用多个密钥](#使用多个密钥)
- [需要触摸](#需要触摸)
- [Email](#email)
  - [Mailvelope on macOS](#mailvelope)
  - [Mutt](#mutt)
- [重置](#重置)
- [重置后恢复](#重置后恢复)
- [注释](#注释)
- [疑难解答](#疑难解答)
- [备选方案](#备选方案)
  - [使用批处理创建密钥](#使用批处理创建密钥)
- [Links](#links)

# 选购

除蓝色“security key”和“Bio Series - FIDO Edition”之外的所有 YubiKey 均与本指南兼容（NEO 型号仅限于 2048 位 RSA 密钥）。 点击[此处]比较 YubiKeys (https://www.yubico.com/products/yubikey-hardware/compare-products-series/)。 点击[此处]查看与 OpenPGP 兼容的 YubiKey 列表(https://support.yubico.com/hc/en-us/articles/360013790259-Using-Your-YubiKey-with-OpenPGP)获得。 2021 年 5 月，Yubico 还发布了一份新闻稿和博客文章，介绍了在8.2或更高版本的 OpenSSH 环境下使用Yubikey 驻留 ssh 密钥（甚至括蓝色“security key”），请参阅[此处](https://www.yubico.com/blog/github-now-supports-ssh-security-keys/)了解详细信息。

要验证 YubiKey 是否为正版，请打开 [支持 U2F 的浏览器](https://support.yubico.com/support/solutions/articles/15000009591-how-to-confirm-your-yubico-device-is-genuine-with-u2f) 打开 [https://www.yubico.com/genuine/](https://www.yubico.com/genuine/)。 插入 Yubico 设备，然后选择_Verify Device_开始验证。 出现提示时触摸 YubiKey，如果询问，则允许其查看设备的品牌和型号。 如果您看到“Yubico device verified”，则该设备是正品。

该网站验证由一组 Yubico 证书颁发机构签署的 YubiKey 设备证明证书，以降低遭到[供应链攻击](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEF%20CON%2025%20-%20r00killah-and-securelyfitz-Secure-Tokin-and-Doobiekeys.pdf)的风险。

您还需要几个小型存储设备（比如microSD卡）来存储密钥的加密备份。

# 准备环境

为了创建加密密钥，建议使用可以合理保证不受对抗性控制的安全环境。 以下是常见环境的安全性排名：

1. 日常使用的操作系统
1. 在日常使用的操作系统中安装的虚拟机（比如 [virt-manager](https://virt-manager.org/)、VirtualBox 或 VMware）
1. 可以双启动的单独的强化 [Debian](https://www.debian.org/) 或 [OpenBSD](https://www.openbsd.org/)
1. LiveCD， 比如 [Debian Live](https://www.debian.org/CD/live/) 或 [Tails](https://tails.boum.org/index.en.html)
1. 安全硬件/固件（[Coreboot](https://www.coreboot.org/)、[Intel ME removed](https://github.com/corna/me_cleaner)）
1. 没有网络功能或使用网闸隔离的专用系统

本指南建议使用可启动的Debian Linux LiveCD 映像来提供这样的环境，但是，根据您的威胁模型，您可以选择安全性更高或者更低的环境。

下载最新的Debian LiveCD：

```bash
$ curl -LfO https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/SHA512SUMS

$ curl -LfO https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/SHA512SUMS.sign

$ curl -LfO https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/$(awk '/xfce.iso/ {print $2}' SHA512SUMS)
```

> **译者注：**可以使用较近的官方网站提供的镜像站以提高下载效率，但不应忘记验证签名以确保镜像未被篡改。

使用 GPG 验证哈希文件的签名：

```bash
$ gpg --verify SHA512SUMS.sign SHA512SUMS
gpg: Signature made Sat 17 Dec 2022 11:06:20 AM PST
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Can't check signature: No public key

$ gpg --keyserver hkps://keyring.debian.org --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: key 0xDA87E80D6294BE9B: public key "Debian CD signing key <debian-cd@lists.debian.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1

$ gpg --verify SHA512SUMS.sign SHA512SUMS
gpg: Signature made Sat 17 Dec 2022 11:06:20 AM PST
gpg:                using RSA key DF9B9C49EAA9298432589D76DA87E80D6294BE9B
gpg: Good signature from "Debian CD signing key <debian-cd@lists.debian.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: DF9B 9C49 EAA9 2984 3258  9D76 DA87 E80D 6294 BE9B
```

如果无法接收公钥，请尝试更改 DNS 解析器和/或使用不同的密钥服务器：

```bash
$ gpg --keyserver hkps://keyserver.ubuntu.com:443 --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
```

确保LiveCD镜像文件的 SHA512 哈希值与签名文件中的哈希值匹配 - 如果以下命令生成输出，则它是正确的：

```bash
$ grep $(sha512sum debian-live-*-amd64-xfce.iso) SHA512SUMS
SHA512SUMS:f9976e2090a54667a26554267941792c293628cceb643963e425bf90449e3c0eeb616e8ededc187070910401c8ab0348fdbc3292b6d04e29dcfb472ac258a542  debian-live-11.6.0-amd64-xfce.iso
```

更多有关信息，请参阅[验证 Debian CD 的真实性](https://www.debian.org/CD/verify)。

安装存储设备并将映像复制到其中：

**Linux**

```bash
$ sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
scsi host2: usb-storage 3-2:1.0
scsi 2:0:0:0: Direct-Access     TS-RDF5  SD  Transcend    TS3A PQ: 0 ANSI: 6
sd 2:0:0:0: Attached scsi generic sg1 type 0
sd 2:0:0:0: [sdb] 31116288 512-byte logical blocks: (15.9 GB/14.8 GiB)
sd 2:0:0:0: [sdb] Write Protect is off
sd 2:0:0:0: [sdb] Mode Sense: 23 00 00 00
sd 2:0:0:0: [sdb] Write cache: disabled, read cache: enabled, doesn't support DPO or FUA
sdb: sdb1 sdb2
sd 2:0:0:0: [sdb] Attached SCSI removable disk

$ sudo dd if=debian-live-*-amd64-xfce.iso of=/dev/sdb bs=4M status=progress ; sync
465+1 records in
465+1 records out
1951432704 bytes (2.0 GB, 1.8 GiB) copied, 42.8543 s, 45.5 MB/s
```

**OpenBSD**

```bash
$ dmesg | tail -n2
sd2 at scsibus4 targ 1 lun 0: <TS-RDF5, SD Transcend, TS3A> SCSI4 0/direct removable serial.0000000000000
sd2: 15193MB, 512 bytes/sector, 31116288 sectors

$ doas dd if=debian-live-*-amd64-xfce.iso of=/dev/rsd2c bs=4m
465+1 records in
465+1 records out
1951432704 bytes transferred in 139.125 secs (14026448 bytes/sec)
```

**Windows**

```
使用启动盘创建工具，如[Rufus](https://rufus.ie/)
```

关闭计算机并断开内部硬盘驱动器和所有不必要的外围设备。 如果在 VM 内运行，则可以跳过此部分，因为不应将此类设备附加到 VM，因为映像仍将作为“LiveCD”运行。

# 所需软件

启动LiveCD映像并配置网络。 

**注意** 如果屏幕锁定，请使用`user/live`解锁，或查阅LiveCD的相关文档取得默认用户名和口令。 

打开终端并安装所需的软件包。

## Debian and Ubuntu

```bash
$ sudo apt update ; sudo apt -y upgrade

$ sudo apt -y install wget gnupg2 gnupg-agent dirmngr cryptsetup scdaemon pcscd secure-delete hopenpgp-tools yubikey-personalization
```

> **译者注：[在Debian 12中不包含hopenpgp-tools包](https://github.com/drduh/YubiKey-Guide/issues/389)，这个包主要用于[校验](#校验)，也可以通过其他方式进行校验

**注意** Live Ubuntu 镜像 [可能需要修改](https://github.com/drduh/YubiKey-Guide/issues/116)  `/etc/apt/sources.list` 并且可能需要额外的软件包：

```bash
$ sudo apt -y install libssl-dev swig libpcsclite-dev
```

**可选** 安装 `ykman` 实用程序，该实用程序将允许您启用触摸策略（需要管理员 PIN）：

```bash
$ sudo apt -y install python3-pip python3-pyscard

$ pip3 install PyOpenSSL

$ pip3 install yubikey-manager

$ sudo service pcscd start

$ ~/.local/bin/ykman openpgp info
```

> **译者注：**在较新的Debian版本中修改了pip安装方式，如果使用`pip3 install`出现报错，则根据提示信息使用`apt install python3-xxx`命令安装，xxx为原pip包名称

```bash
$ sudo apt -y install pcscd python3-yubikey-manager python3-ykman

$ sudo service pcscd start

$ ykman openpgp info
```

**可选 **根据实际使用需求安装图形界面工具和PIV工具

```bash
$ sudo apt -y install yubico-piv-tool yubikey-manager-qt kleopatra
```

## Fedora

```bash
$ sudo dnf install wget
$ wget https://github.com/rpmsphere/noarch/raw/master/r/rpmsphere-release-34-2.noarch.rpm
$ sudo rpm -Uvh rpmsphere-release*rpm

$ sudo dnf install gnupg2 dirmngr cryptsetup gnupg2-smime pcsc-tools opensc pcsc-lite secure-delete pgp-tools yubikey-personalization-gui
```

## Arch

```bash
$ sudo pacman -Syu gnupg pcsclite ccid hopenpgp-tools yubikey-personalization
```

## RHEL7

```bash
$ sudo yum install -y gnupg2 pinentry-curses pcsc-lite pcsc-lite-libs gnupg2-smime
```

## NixOS

使用给定的配置生成网络隔离 NixOS LiveCD 映像：

```nix
# yubikey-installer.nix
let
  configuration = { config, lib, pkgs, ... }:
    with pkgs;
    let
      src = fetchGit "https://github.com/drduh/YubiKey-Guide";

      guide = "${src}/README.md";

      contrib = "${src}/contrib";

      drduhConfig = fetchGit "https://github.com/drduh/config";

      gpg-conf = "${drduhConfig}/gpg.conf";

      xserverCfg = config.services.xserver;

      pinentryFlavour = if xserverCfg.desktopManager.lxqt.enable || xserverCfg.desktopManager.plasma5.enable then
        "qt"
      else if xserverCfg.desktopManager.xfce.enable then
        "gtk2"
      else if xserverCfg.enable || config.programs.sway.enable then
        "gnome3"
      else
        "curses";

      # Instead of hard-coding the pinentry program, chose the appropriate one
      # based on the environment of the image the user has chosen to build.
      gpg-agent-conf = runCommand "gpg-agent.conf" {} ''
        sed '/pinentry-program/d' ${drduhConfig}/gpg-agent.conf > $out
        echo "pinentry-program ${pinentry.${pinentryFlavour}}/bin/pinentry" >> $out
      '';

      view-yubikey-guide = writeShellScriptBin "view-yubikey-guide" ''
        viewer="$(type -P xdg-open || true)"
        if [ -z "$viewer" ]; then
          viewer="${glow}/bin/glow -p"
        fi
        exec $viewer "${guide}"
      '';

      shortcut = makeDesktopItem {
        name = "yubikey-guide";
        icon = "${yubikey-manager-qt}/share/ykman-gui/icons/ykman.png";
        desktopName = "drduh's YubiKey Guide";
        genericName = "Guide to using YubiKey for GPG and SSH";
        comment = "Open the guide in a reader program";
        categories = [ "Documentation" ];
        exec = "${view-yubikey-guide}/bin/view-yubikey-guide";
      };

      yubikey-guide = symlinkJoin {
        name = "yubikey-guide";
        paths = [ view-yubikey-guide shortcut ];
      };

    in {
      nixpkgs.config = { allowBroken = true; };

      isoImage.isoBaseName = lib.mkForce "nixos-yubikey";
      # Uncomment this to disable compression and speed up image creation time
      #isoImage.squashfsCompression = "gzip -Xcompression-level 1";

      boot.kernelPackages = linuxPackages_latest;
      # Always copytoram so that, if the image is booted from, e.g., a
      # USB stick, nothing is mistakenly written to persistent storage.
      boot.kernelParams = [ "copytoram" ];
      # Secure defaults
      boot.cleanTmpDir = true;
      boot.kernel.sysctl = { "kernel.unprivileged_bpf_disabled" = 1; };

      services.pcscd.enable = true;
      services.udev.packages = [ yubikey-personalization ];

      programs = {
        ssh.startAgent = false;
        gnupg.agent = {
          enable = true;
          enableSSHSupport = true;
        };
      };

      environment.systemPackages = [
        # Tools for backing up keys
        paperkey
        pgpdump
        parted
        cryptsetup

        # Yubico's official tools
        yubikey-manager
        yubikey-manager-qt
        yubikey-personalization
        yubikey-personalization-gui
        yubico-piv-tool
        yubioath-desktop

        # Testing
        ent
        (haskell.lib.justStaticExecutables haskellPackages.hopenpgp-tools)

        # Password generation tools
        diceware
        pwgen

        # Miscellaneous tools that might be useful beyond the scope of the guide
        cfssl
        pcsctools

        # This guide itself (run `view-yubikey-guide` on the terminal to open it
        # in a non-graphical environment).
        yubikey-guide
      ];

      # Disable networking so the system is air-gapped
      # Comment all of these lines out if you'll need internet access
      boot.initrd.network.enable = false;
      networking.dhcpcd.enable = false;
      networking.dhcpcd.allowInterfaces = [];
      networking.interfaces = {};
      networking.firewall.enable = true;
      networking.useDHCP = false;
      networking.useNetworkd = false;
      networking.wireless.enable = false;
      networking.networkmanager.enable = lib.mkForce false;

      # Unset history so it's never stored
      # Set GNUPGHOME to an ephemeral location and configure GPG with the
      # guide's recommended settings.
      environment.interactiveShellInit = ''
        unset HISTFILE
        export GNUPGHOME="/run/user/$(id -u)/gnupg"
        if [ ! -d "$GNUPGHOME" ]; then
          echo "Creating \$GNUPGHOME…"
          install --verbose -m=0700 --directory="$GNUPGHOME"
        fi
        [ ! -f "$GNUPGHOME/gpg.conf" ] && cp --verbose ${gpg-conf} "$GNUPGHOME/gpg.conf"
        [ ! -f "$GNUPGHOME/gpg-agent.conf" ] && cp --verbose ${gpg-agent-conf} "$GNUPGHOME/gpg-agent.conf"
        echo "\$GNUPGHOME is \"$GNUPGHOME\""
      '';

      # Copy the contents of contrib to the home directory, add a shortcut to
      # the guide on the desktop, and link to the whole repo in the documents
      # folder.
      system.activationScripts.yubikeyGuide = let
        homeDir = "/home/nixos/";
        desktopDir = homeDir + "Desktop/";
        documentsDir = homeDir + "Documents/";
      in ''
        mkdir -p ${desktopDir} ${documentsDir}
        chown nixos ${homeDir} ${desktopDir} ${documentsDir}

        cp -R ${contrib}/* ${homeDir}
        ln -sf ${yubikey-guide}/share/applications/yubikey-guide.desktop ${desktopDir}
        ln -sfT ${src} ${documentsDir}/YubiKey-Guide
      '';
    };

  nixos = import <nixpkgs/nixos/release.nix> {
    inherit configuration;
    supportedSystems = [ "x86_64-linux" ];
  };

  # Choose the one you like:
  #nixos-yubikey = nixos.iso_minimal; # No graphical environment
  #nixos-yubikey = nixos.iso_gnome;
  nixos-yubikey = nixos.iso_plasma5;

in {
  inherit nixos-yubikey;
}
```

构建安装程序并将其复制到 USB 驱动器。

```bash
$ nix build -f yubikey-installer.nix -o installer nixos-yubikey

$ sudo cp -v installer/iso/*.iso /dev/sdb; sync
'installer/iso/nixos-yubikey-22.05beta-248980.gfedcba-x86_64-linux.iso' -> '/dev/sdb'
```

使用此映像，您无需手动创建[临时工作目录](#temporary-working-directory) 或[强化配置](#harden-configuration)，因为它是在创建映像时完成的。

## OpenBSD

```bash
$ doas pkg_add gnupg pcsc-tools
```

## macOS

下载并安装 [Homebrew](https://brew.sh/) 和以下软件包：

```bash
$ brew install gnupg yubikey-personalization hopenpgp-tools ykman pinentry-mac wget
```

**注意** 可能需要安装额外的 Python 包依赖项才能使用 [`ykman`](https://support.yubico.com/support/solutions/articles/15000012643-yubikey-manager-cli-ykman-user -guide) - `pip install yubikey-manager`

## Windows

下载并安装[Gpg4Win](https://www.gpg4win.org/)和[PuTTY](https://putty.org)。

您可能还需要更新版本的 [yubikey-personalization](https://developers.yubico.com/yubikey-personalization/Releases/) 和 [yubico-c](https://developers.yubico.com/yubico-c/Releases/)。

# 熵

生成加密密钥需要高质量的[随机性](https://www.random.org/randomness/)，以熵来衡量。

大多数操作系统使用基于软件的伪随机数生成器或基于 CPU 的硬件随机数生成器 (HRNG)。

或者，您可以使用单独的硬件设备，例如 [OneRNG](https://onerng.info/onerng/) 来[提高熵生成的速度](https://lwn.net/Articles/648550/)，并且可能 还有质量。

> **译者注**此OneRNG目前还没看到国内有销售或制作，猜测可能可以使用通用的CC2531开发板刷入软件实现。[开发资料](https://github.com/OneRNG/)

## YubiKey

YubiKey 固件版本 5.2.3 引入了“OpenPGP 3.4 支持的增强”——可以选择通过智能卡接口从 YubiKey 收集额外的熵。

要使用从 YubiKey 检索到的额外 512 字节作为内核 PRNG 的种子：

```bash
$ echo "SCD RANDOM 512" | gpg-connect-agent | sudo tee /dev/random | hexdump -C
```

## OneRNG

安装[rng-tools](https://wiki.archlinux.org/index.php/Rng-tools)软件：

```bash
$ sudo apt -y install at rng-tools python3-gnupg openssl

$ wget https://github.com/OneRNG/onerng.github.io/raw/master/sw/onerng_3.7-1_all.deb

$ sha256sum onerng_3.7-1_all.deb
b7cda2fe07dce219a95dfeabeb5ee0f662f64ba1474f6b9dddacc3e8734d8f57  onerng_3.7-1_all.deb

$ sudo dpkg -i onerng_3.7-1_all.deb

$ echo "HRNGDEVICE=/dev/ttyACM0" | sudo tee /etc/default/rng-tools
```

插入设备并重新启动 rng-tools：

```bash
$ sudo atd

$ sudo service rng-tools restart
```

# 创建密钥

## 临时工作目录

创建一个临时目录，并将其设置为GnuPG目录（该目录将在[重新启动](https://en.wikipedia.org/wiki/Tmpfs)时清除）：

```bash
$ export GNUPGHOME=$(mktemp -d -t gnupg_$(date +%Y%m%d%H%M)_XXX)
```

如果要保留工作环境，请将 GnuPG 目录设置为您的主文件夹：

```bash
$ export GNUPGHOME=~/gnupg-workspace
```

## 加固配置

使用以下选项在临时工作目录中创建加固配置：

```bash
$ wget -O $GNUPGHOME/gpg.conf https://raw.githubusercontent.com/drduh/config/master/gpg.conf

$ grep -ve "^#" $GNUPGHOME/gpg.conf
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
fixed-list-mode
no-comments
no-emit-version
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
no-symkey-cache
use-agent
throw-keyids
```

**重要** 在执行以下设置前禁用或断开网络。

# 主密钥

要生成的第一个密钥是主密钥。 它将仅用于认证：颁发用于加密、签名和身份验证的子密钥。

**重要** 主密钥应始终保持离线状态，并且仅在撤销或颁发新子密钥时才可访问。 密钥也可以在 YubiKey 本身上生成，以确保不存在其他副本。

系统会提示您输入并验证口令 - 请妥善保存，因为稍后您将多次需要它。

> **译者注：**passphrase、password等词汇在本文中统一译为口令

生成一个强口令，可以将其写在安全的地方或记住：

```bash
$ gpg --gen-random --armor 0 24
ydOmByxmDe63u7gqx2XI9eDgpvJwibNH
```

如果手写口令，请使用大写字母以提高可读性：

```bash
$ LC_ALL=C tr -dc '[:upper:]' < /dev/urandom | fold -w 20 | head -n1
BSSYMUGGTJQVWZZWOPJG
```

**重要** 将此凭证保存在永久、安全的位置，因为在过期后需要它来颁发新的子密钥、添加外的 YubiKey，以及您需要多次访问 Debian Live 环境剪贴板以生成密钥。

**提示** 在 Linux 或 OpenBSD 上，使用鼠标选择密码或双击密码将其复制到剪贴板。 使用鼠标中键或`Shift-Insert`进行粘贴。

使用 GPG 生成新密钥，选择“(8) RSA (set your own capabilities)”、仅选择“Certify”（认证）功能和和“4096”位密钥。

**不要**为主（认证）密钥设置过期时间 - 请参阅[注释 #3](#注释)。

> **译者注**鉴于5.2.3以后版本的Yibikey均支持ECC算法，可以根据需求选用ECC算法生成密钥，操作方法与RSA相同

```bash
$ gpg --expert --full-generate-key

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Sign Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for a RSA key: Sign Certify Encrypt Authenticate
Current allowed actions: Certify

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
```

输入任何姓名和电子邮件地址（不必是有效的）：

```bash
GnuPG needs to construct a user ID to identify your key.

Real name: Dr Duh
Email address: doc@duh.to
Comment: [Optional - leave blank]
You selected this USER-ID:
    "Dr Duh <doc@duh.to>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o

We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

gpg: /tmp.FLZC0xcM/trustdb.gpg: trustdb created
gpg: key 0xFF3E7D88647EBCDB marked as ultimately trusted
gpg: directory '/tmp.FLZC0xcM/openpgp-revocs.d' created
gpg: revocation certificate stored as '/tmp.FLZC0xcM/openpgp-revocs.d/011CE16BD45B27A55BA8776DFF3E7D88647EBCDB.rev'
public and secret key created and signed.

pub   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                              Dr Duh <doc@duh.to>
```

将密钥 ID 导出为[变量](https://stackoverflow.com/questions/1158091/defining-a-variable-with-or-without-export/1158231#1158231) (`KEYID`) 供以后使用：

```bash
$ export KEYID=0xFF3E7D88647EBCDB
```

# 使用现有密钥签名

（可选）如果您已经拥有 PGP 密钥，您可能需要使用旧密钥对新密钥进行签名，以证明新密钥由您控制。

导出现有密钥以将其移至工作密钥环：

```bash
$ gpg --export-secret-keys --armor --output /tmp/new.sec
```

然后签署新密钥：

```bash
$ gpg  --default-key $OLDKEY --sign-key $KEYID
```

# 子密钥

编辑主密钥以添加子密钥：

```bash
$ gpg --expert --edit-key $KEYID

Secret key is available.

sec  rsa4096/0xEA5DE91459B80592
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
[ultimate] (1). Dr Duh <doc@duh.to>
```

使用 4096 位 RSA 密钥。

子密钥的有效期为 1 年 - 可以使用离线主密钥对其进行续订。 请参阅[密钥轮替](#密钥轮替)。

## 签名

通过选择`addkey`然后选择`(4) RSA (sign only)`来创建[签名密钥](https://stackoverflow.com/questions/5421107/can-rsa-be-both-used-as-encryption-and-signature/5432623#5432623)：

```bash
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
[ultimate] (1). Dr Duh <doc@duh.to>
```

## 加密

接下来，通过选`(6) RSA (encrypt only)`创建一个[加密密钥](https://www.cs.cornell.edu/courses/cs5430/2015sp/notes/rsa_sign_vs_dec.php)：

```bash
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09       usage: E
[ultimate] (1). Dr Duh <doc@duh.to>
```

## 身份验证

最后，创建一个[身份验证密钥](https://superuser.com/questions/390265/what-is-a-gpg-with-authenticate-capability-used-for)。

GPG 不提供仅验证密钥类型，因此选择`(8) RSA (set your own capabilities)`并切换所需的功能，直到唯一允许的操作是`Authenticate`：

```bash
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? S

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? E

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions:

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? A

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? Q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Mon 10 Sep 2018 00:00:00 PM UTC
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
```

通过保存密钥来完成创建。

```bash
gpg> save
```

## 添加额外的身份

（可选）要添加其他电子邮件地址或身份，请使用`adduid`。

首先打开密钥环：
```bash
$ gpg --expert --edit-key $KEYID
```

然后添加新身份，并通过`trust`设置信任新增的身份：
```bash
gpg> adduid
Real name: Dr Duh
Email address: DrDuh@other.org
Comment:
You selected this USER-ID:
    "Dr Duh <DrDuh@other.org>"

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: never       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: never       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: never       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
[ unknown] (2). Dr Duh <DrDuh@other.org>

gpg> trust
sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: never       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: never       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: never       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
[ unknown] (2). Dr Duh <DrDuh@other.org>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: never       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: never       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: never       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
[ unknown] (2). Dr Duh <DrDuh@other.org>

gpg> uid 1

sec  rsa4096/0xFF3E7D88647EBCDB
created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: never       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: never       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: never       usage: A
[ultimate] (1)* Dr Duh <doc@duh.to>
[ unknown] (2). Dr Duh <DrDuh@other.org>

gpg> primary

sec  rsa4096/0xFF3E7D88647EBCDB
created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: never       usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: never       usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: never       usage: A
[ultimate] (1)* Dr Duh <doc@duh.to>
[ unknown] (2)  Dr Duh <DrDuh@other.org>

gpg> save
```

默认情况下，最后添加的身份将是主用户 ID - 使用`primary`进行更改。

# 校验

列出生成的密钥并验证输出：

```bash
$ gpg -K
/tmp.FLZC0xcM/pubring.kbx
-------------------------------------------------------------------------
sec   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                            Dr Duh <doc@duh.to>
ssb   rsa4096/0xBECFA3C1AE191D15 2017-10-09 [S] [expires: 2018-10-09]
ssb   rsa4096/0x5912A795E90DD2CF 2017-10-09 [E] [expires: 2018-10-09]
ssb   rsa4096/0x3F29127E79649A3D 2017-10-09 [A] [expires: 2018-10-09]
```

使用`adduid`命令添加您想要关联的任何其他身份或电子邮件地址。

**提示** 使用 OpenPGP [关键最佳实践检查器](https://riseup.net/en/security/message-security/openpgp/best-practices#openpgp-key-checks) 进行验证：

```bash
$ gpg --export $KEYID | hokey lint
```

输出将以红色文本显示密钥的任何问题。 如果一切都是绿色的，则您的密钥通过了每项测试。 如果它是红色的，则您的密钥未通过其中一项测试。

> hokey 可能会警告（橙色文本）有关身份验证密钥的交叉认证。 GPG 的 [签名子密钥交叉认证](https://gnupg.org/faq/subkey-cross-certify.html) 文档提供了有关交叉认证的更多详细信息，并且 gpg v2.2.1 说明“子密钥 <keyid> 不签名,并且因此不需要交叉认证”。 hokey 还可能表示主键上的“密钥过期时间：[]”存在问题（红色文本）（请参阅[注释 #3](#注释) 关于不为主键设置过期时间的信息）。

# 导出密钥

导出密钥时需要使用您先前输入的口令，并且导出的主密钥和子密钥使用该口令进行加密。

保存密钥的副本：

```bash
$ gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/mastersub.key

$ gpg --armor --export-secret-subkeys $KEYID > $GNUPGHOME/sub.key
```

在 Windows 上，请注意，使用除`.gpg`之外的任何扩展名或尝试 IO 重定向到文件都会导致密钥混乱，从而导致以后无法再次导入它：

```bash
$ gpg -o \path\to\dir\mastersub.gpg --armor --export-secret-keys $KEYID

$ gpg -o \path\to\dir\sub.gpg --armor --export-secret-subkeys $KEYID
```

# 吊销证书

尽管我们会将主密钥备份并存储在安全的地方，但最佳实践是永远不排除丢失主密钥或备份失败的可能性。 如果没有主密钥，就无法更新或轮替子密钥或生成吊销证书，PGP 身份将毫无用处。

更糟糕的是，我们无法以任何方式向使用我们密钥的人宣传这一事实。 可以合理地假设这种情况“将”在某个时刻发生，并且弃用孤立密钥的唯一剩余方法是吊销证书。

要创建吊销证书：

``` bash
$ gpg --output $GNUPGHOME/revoke.asc --gen-revoke $KEYID
```

`revoke.asc`证书文件应存储（或打印）在与主密钥不同的其他位置，以便在主备份失败时可以进行检索。

# 备份

一旦密钥移动到 YubiKey，就无法再次移动！ 在可移动介质上创建密钥环的**加密**备份，以便您可以将其离线保存在安全的地方。
	
**提示** ext2 文件系统（未加密）可以在 Linux 和 OpenBSD 上挂载。 请考虑使用 FAT32/NTFS 文件系统来实现 MacOS/Windows 兼容性。

作为额外的备份措施，请考虑使用密钥的[纸质副本](https://www.jabberwocky.com/software/paperkey/)。 [Linux 内核维护者 PGP 指南](https://www.kernel.org/doc/html/latest/process/maintainer-pgp-guide.html#back-up-your-master-key-for-disaster-recovery ）指出此类打印输出_仍然受口令保护_。 它建议*将口令写在纸上*，因为您不太可能记住创建纸质备份时使用的原始密钥口令。 显然，您需要一个非常好的地方来保存这样的打印输出。

强烈建议将加密的 OpenPGP 私钥材料保持离线状态，以阻止[密钥覆盖攻击](https://www.kopenpgp.com/)。

**Linux**

连接一个外部存储设备并检查其标签：

```bash
$ sudo dmesg | tail
mmc0: new high speed SDHC card at address a001
mmcblk0: mmc0:a001 SS16G 14.8 GiB

$ sudo fdisk -l /dev/mmcblk0
Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

写入随机数据以清除存储设备中可能残存的历史数据：

```bash
$ sudo dd if=/dev/urandom of=/dev/mmcblk0 bs=4M status=progress
```

擦除并创建新的分区表：

```bash
$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x3c1ad14a.

Command (m for help): g
Created a new GPT disklabel (GUID: 4E7495FD-85A3-3E48-97FC-2DD8D41516C3).

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

```

创建一个大小为 25 MB 的新分区：

```bash
$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.36.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-30261214, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-30261214, default 30261214): +25M

Created a new partition 1 of type 'Linux filesystem' and of size 25 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

使用 [LUKS](https://askubuntu.com/questions/97196/how-secure-is-an-encrypted-luks-filesystem) 加密新分区。 生成一个不同的口令，用于保护文件系统：

```bash
$ sudo cryptsetup luksFormat /dev/mmcblk0p1

WARNING!
========
This will overwrite data on /dev/mmcblk0p1 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/mmcblk0p1:
Verify passphrase:
```

挂载分区：

```bash
$ sudo cryptsetup luksOpen /dev/mmcblk0p1 secret
Enter passphrase for /dev/mmcblk0p1:
```

创建 ext2 文件系统：

```bash
$ sudo mkfs.ext2 /dev/mapper/secret -L gpg-$(date +%F)
```

挂载文件系统并复制包含密钥环的临时 GnuPG 目录：

```bash
$ sudo mkdir /mnt/encrypted-storage

$ sudo mount /dev/mapper/secret /mnt/encrypted-storage

$ sudo cp -avi $GNUPGHOME /mnt/encrypted-storage/
```

**可选** 备份 OneRNG 包：

```bash
$ sudo cp onerng_3.7-1_all.deb /mnt/encrypted-storage/
```

**注意** 如果您计划设置多个密钥，请保持备份介质挂载状态或记住在[保存](https://lists.gnupg.org/pipermail/gnupg-users/2016-July/056353.html)之前终止 gpg 进程。

卸载、关闭和断开加密卷：

```bash
$ sudo umount /mnt/encrypted-storage/

$ sudo cryptsetup luksClose secret
```

**OpenBSD**

连接 USB 盘并确定其标签：

```bash
$ dmesg | grep sd.\ at
sd2 at scsibus5 targ 1 lun 0: <TS-RDF5, SD Transcend, TS37> SCSI4 0/direct removable serial.00000000000000000000
```

查看现有分区以确保它是正确的设备：

```bash
$ doas disklabel -h sd2
```

通过创建 FS 类型为`RAID`且大小为 25 MB 的`a`分区来初始化磁盘：

```bash
$ doas fdisk -giy sd2
Writing MBR at offset 0.
Writing GPT.

$ doas disklabel -E sd2
Label editor (enter '?' for help at any prompt)
sd2> a a
offset: [64]
size: [31101776] 25M
FS type: [4.2BSD] RAID
sd2*> w
sd2> q
No label changes
```

使用 bioctl 加密：

```bash
$ doas bioctl -c C -l sd2a softraid0
New passphrase:
Re-type passphrase:
softraid0: CRYPTO volume attached as sd3
```

在新的加密卷和文件系统上创建一个`i`分区：

```bash
$ doas fdisk -giy sd3
Writing MBR at offset 0.
Writing GPT.

$ doas disklabel -E sd3
Label editor (enter '?' for help at any prompt)
sd3> a i
offset: [64]
size: [16001]
FS type: [4.2BSD]
sd3*> w
sd3> q
No label changes.

$ doas newfs sd3i
```

挂载文件系统复制包含密钥环的临时目录：

```bash
$ doas mkdir /mnt/encrypted-storage

$ doas mount /dev/sd3i /mnt/encrypted-storage

$ doas cp -avi $GNUPGHOME /mnt/encrypted-storage
```

**注意** 如果您计划设置多个密钥，请保持备份介质挂载状态或记住在[保存](https://lists.gnupg.org/pipermail/gnupg-users/2016-July/056353.html)之前终止 gpg 进程。

卸载、关闭和断开加密卷：

```bash
$ doas umount /mnt/encrypted-storage

$ doas bioctl -d sd3
```

请参阅 [OpenBSD FAQ#14](https://www.openbsd.org/faq/faq14.html#softraidCrypto) 了解更多信息。

# 导出公钥

**重要** 如果没有*公钥*，您将无法使用 GPG 来加密、解密或签署消息。 但是，您仍然可以使用 YubiKey 进行 SSH 身份验证。

在可移动存储设备上创建另一个分区来存储公钥，或者重新连接网络并上传到密钥服务器。

**Linux**

```bash
$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.36.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition number (2-128, default 2):
First sector (53248-30261214, default 53248):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (53248-30261214, default 30261214): +25M

Created a new partition 2 of type 'Linux filesystem' and of size 25 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

$ sudo mkfs.ext2 /dev/mmcblk0p2

$ sudo mkdir /mnt/public

$ sudo mount /dev/mmcblk0p2 /mnt/public/

$ gpg --armor --export $KEYID | sudo tee /mnt/public/gpg-$KEYID-$(date +%F).asc
```

**OpenBSD**

```bash
$ doas disklabel -E sd2
Label editor (enter '?' for help at any prompt)
sd2> a b
offset: [32130]
size: [31069710] 25M
FS type: [swap] 4.2BSD
sd2*> w
sd2> q
No label changes.

$ doas newfs sd2b

$ doas mkdir /mnt/public

$ doas mount /dev/sd2b /mnt/public

$ gpg --armor --export $KEYID | doas tee /mnt/public/gpg-$KEYID-$(date +%F).asc
```

**Windows**

```bash
$ gpg -o \path\to\dir\pubkey.gpg --armor --export $KEYID
```

**Keyserver**

（可选）将公钥上传到[公钥服务器](https://debian-administration.org/article/451/Submitting_your_GPG_key_to_a_keyserver)：

```bash
$ gpg --send-key $KEYID

$ gpg --keyserver pgp.mit.edu --send-key $KEYID

$ gpg --keyserver keys.gnupg.net --send-key $KEYID

$ gpg --keyserver hkps://keyserver.ubuntu.com:443 --send-key $KEYID
```

一段时间后，公钥将传播到[其他](https://pgp.key-server.io/pks/lookup?search=doc%40duh.to&fingerprint=on&op=vindex)[服务器](https:// pgp.mit.edu/pks/lookup?search=doc%40duh.to&op=index)。

# 配置智能卡

插入 YubiKey 并使用 GPG 将其配置为智能卡：

```bash
$ gpg --card-edit

Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: D2760001240102010006055532110000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: [not set]
Language prefs ...: [not set]
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

进入管理模式：

```bash
gpg/card> admin
Admin commands are allowed
```

**注意** 如果卡被锁定，请参见[重置](#重置)。

**Windows**

使用 [YubiKey Manager](https://developers.yubico.com/yubikey-manager) 应用程序（请注意，这不是名称类似的旧版 YubiKey NEO Manager）来启用 CCID 功能。

## 启用KDF

密钥派生函数 (KDF) 使 YubiKey 能够存储 PIN 的哈希值，从而防止 PIN 以纯文本形式传递。 请注意，这需要相对较新版本的 GnuPG 才能工作，并且可能与其他 GPG 客户端（尤其是移动客户端）不兼容。 这些不兼容的客户端将无法使用 YubiKey GPG 功能，因为 PIN 码将始终被拒绝。 如果您不确定仅在受支持的平台上使用 YubiKey，最好跳过此步骤。

```bash
gpg/card> kdf-setup
```

## 修改PIN

[GPG 接口](https://developers.yubico.com/PGP/) 与 Yubikey 上的其他模块分开，例如 [PIV 接口](https://developers.yubico.com/PIV/Introduction/YubiKey_and_PIV.html)。 GPG 界面有自己的 *PIN*、*Admin PIN* 和 *Reset Code* - 这些应该在使用前修改默认值！

错误输入用户 *PIN* 3 次将导致 PIN 被阻止； 可以使用*Admin PIN码*或*Reset Code*来解锁。

错误地输入*Admin PIN* 或*Reset Code* 3 将次会破坏卡上的所有 GPG 数据。必须重新配置 Yubikey。

名称       | 默认值 | 用途 
-----------|---------------|-------------------------------------------------------------
PIN        | `123456`      | 解密和身份验证 (SSH) 
Admin PIN  | `12345678`    | 重置*PIN*、更改*Reset Code*、添加密钥或所有者信息 
Reset code | ***无***      | 重置*PIN*（[更多信息](https://forum.yubico.com/viewtopicd01c.html?p=9055#p9055)） 

PIN的取值最多可包含 127 个 ASCII 字符，并且必须至少为 6位 (*PIN*) 或 8位（*Admin PIN*、*Reset Code*）字符。 有关详细信息，请参阅GnuPG 文档有关 [管理PIN](https://www.gnupg.org/howtos/card-howto/en/ch03s02.html) 的内容。

要更新 Yubikey 上的 GPG PIN：

```bash
gpg/card> passwd
gpg: OpenPGP card no. D2760001240102010006055532110000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
```

**注意** 稍后可以使用以下命令更改重试尝试次数，点击[此处](https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html#ykman-openpgp-access-set-retries-options-pin-retries-reset-code-retries-admin-pin-retries)查看详细说明：

```bash
$ ykman openpgp access set-retries 5 5 5 -f -a YOUR_ADMIN_PIN
```

## 设置信息

些字段是可选的。

```bash
gpg/card> name
Cardholder's surname: Duh
Cardholder's given name: Dr

gpg/card> lang
Language preferences: en

gpg/card> login
Login data (account name): doc@duh.to

gpg/card> list

Application ID ...: D2760001240102010006055532110000
Version ..........: 3.4
Manufacturer .....: unknown
Serial number ....: 05553211
Name of cardholder: Dr Duh
Language prefs ...: en
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: doc@duh.to
Private DO 4 .....: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: on
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]

gpg/card> quit
```

# 传输密钥

**重要** 使用`keytocard`将密钥传输到 YubiKey 是一种破坏性的单向操作。 确保您在继续之前已进行备份，`keytocard`将本地磁盘上的密钥转换为存根，这意味着磁盘上的副本不再可用于传输到后续的安全密钥设备或派生其他密钥。

以前的 GPG 版本需要在选择密钥之前使用“toggle”命令。 当前选择的密钥用`*`表示。 移动密钥时，一次只能选择一个密钥。

```bash
$ gpg --edit-key $KEYID

Secret key is available.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
```

## 签名密钥

系统将提示您输入主密钥口令和Admin PIN。

选择并传输签名密钥。

```bash
gpg> key 1

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb* rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xBECFA3C1AE191D15, created 2016-05-24
```

## 加密密钥

再次键入`key 1`以取消选择，并键入`key 2`以选择下一个密钥：

```bash
gpg> key 1

gpg> key 2

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb* rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2

[...]
```

## 身份验证密钥

再次键入`key 2`以取消选择，并键入`key 3`以选择最后一个密钥：

```bash
gpg> key 2

gpg> key 3

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb* rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
```

保存并退出：

```bash
gpg> save
```

# 校验智能卡

验证子密钥已移动到 YubiKey，如`ssb>`所示：

```bash
$ gpg -K
/tmp.FLZC0xcM/pubring.kbx
-------------------------------------------------------------------------
sec   rsa4096/0xFF3E7D88647EBCDB 2017-10-09 [C]
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                            Dr Duh <doc@duh.to>
ssb>  rsa4096/0xBECFA3C1AE191D15 2017-10-09 [S] [expires: 2018-10-09]
ssb>  rsa4096/0x5912A795E90DD2CF 2017-10-09 [E] [expires: 2018-10-09]
ssb>  rsa4096/0x3F29127E79649A3D 2017-10-09 [A] [expires: 2018-10-09]
```

# 多个YubiKey

要配置其他安全密钥，请恢复主密钥备份并重复[配置智能卡](#配置智能卡) 过程。

```bash
$ mv -vi $GNUPGHOME $GNUPGHOME.1
renamed '/tmp.FLZC0xcM' -> '/tmp.FLZC0xcM.1'

$ cp -avi /mnt/encrypted-storage/tmp.XXX $GNUPGHOME
'/mnt/encrypted-storage/tmp.FLZC0xcM' -> '/tmp.FLZC0xcM'

$ cd $GNUPGHOME
```

## 在多个Yubikey间切换

当您使用 *keytocard* 命令将 GPG 密钥添加到 Yubikey 时，GPG 会从密钥环中删除该密钥，并添加一个指向该确切 Yubikey 的 *存根*（存根标识 GPG KeyID 和 Yubikey 的序列号）。
	
但是，当您对第二个 Yubikey 执行相同的操作时，密钥环中的存根将被 *keytocard* 操作覆盖，现在存根指向您的第二个 Yubikey。 添加更多会重复此覆盖操作。

换句话说，存根将仅指向最后写入的 Yubikey。
	
当使用 GPG 密钥操作与您放置在 Yubikey 上的 GPG 密钥时，GPG 将请求特定的 Yubikey，要求您插入具有给定序列号（由存根引用）的 Yubikey。 如果没有人工干预，GPG 将无法识别具有不同序列号的另一个 Yubikey。
	
您可以强制 GPG 扫描卡并重新创建存根以指向另一个 Yubikey。

使用相同的 GPG 密钥（如上所述）创建两个（或更多 Yubikey），其中存根指向第二个 Yubikey：
	
插入第一个 Yubikey（具有不同的序列号）并运行以下命令：
	
```bash
$  gpg-connect-agent "scd serialno" "learn --force" /bye
```
然后，GPG 将扫描您的第一个 Yubikey 中的 GPG 密钥，并重新创建存根以指向第一个 Yubikey 的 GPG keyID 和 Yubikey 序列号。
	
要返回使用第二个 Yubikey，只需重复（插入其他 Yubikey 并重新运行命令）。
	
显然这个命令不容易记住，因此建议创建一个脚本或 shell 别名，以使其更加用户友好。

# 清理

在完成设置之前，请确保您已完成以下操作：

* 将加密、签名和身份验证子密钥保存到 YubiKey（执行`gpg -K`查看子密钥应显示`ssb>`）。
* 保存了已修改过的 YubiKey 用户和管理员 PIN。
* 将 GPG 主密钥的口令保存在安全、长期的位置。
* 将主密钥、子密钥和吊销证书的副本保存在离线存储的加密卷上。
* 将 LUKS 加密卷的口令保存在安全、长期的位置（与设备本身分别放置）。
* 将公钥的副本保存在以后易于访问的地方。

现在重新启动或[安全删除](http://srm.sourceforge.net/) `$GNUPGHOME` 并从 GPG 密钥环中删除密钥：

```bash
$ gpg --delete-secret-key $KEYID

$ sudo srm -r $GNUPGHOME || sudo rm -rf $GNUPGHOME

$ unset GNUPGHOME
```

**重要** 如果未使用临时环境，请确保您已安全删除所有生成的密钥和吊销证书！

# 使用密钥

下载 [drduh/config/gpg.conf](https://github.com/drduh/config/blob/master/gpg.conf):

```bash
$ cd ~/.gnupg ; wget https://raw.githubusercontent.com/drduh/config/master/gpg.conf

$ chmod 600 gpg.conf
```

安装所需的软件包并挂载之前创建的非加密卷：

**Linux**

```bash
$ sudo apt update && sudo apt install -y gnupg2 gnupg-agent gnupg-curl scdaemon pcscd

$ sudo mount /dev/mmcblk0p2 /mnt
```

**OpenBSD**

```bash
$ doas pkg_add gnupg pcsc-tools

$ doas mount /dev/sd2b /mnt
```

导入公钥文件：

```bash
$ gpg --import /mnt/gpg-0x*.asc
gpg: key 0xFF3E7D88647EBCDB: public key "Dr Duh <doc@duh.to>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

或者从密钥服务器下载公钥：

```bash
$ gpg --recv $KEYID
gpg: requesting key 0xFF3E7D88647EBCDB from hkps server hkps.pool.sks-keyservers.net
[...]
gpg: key 0xFF3E7D88647EBCDB: public key "Dr Duh <doc@duh.to>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

通过选择`trust`和`5`编辑主密钥以为其分配最终信任：

```bash
$ export KEYID=0xFF3E7D88647EBCDB

$ gpg --edit-key $KEYID

gpg> trust
pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: C
                               trust: unknown       validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09  usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09  usage: E
sub  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09  usage: A
[ unknown] (1). Dr Duh <doc@duh.to>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: C
                               trust: ultimate      validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09  usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09  usage: E
sub  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09  usage: A
[ unknown] (1). Dr Duh <doc@duh.to>

gpg> quit
```

移除并重新插入 YubiKey 并验证状态：

```bash
$ gpg --card-status
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D2760001240102010006055532110000
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: Dr Duh
Language prefs ...: en
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: doc@duh.to
Signature PIN ....: not forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: on
Signature key ....: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
      created ....: 2016-05-24 23:22:01
Encryption key....: 6F26 6F46 845B BEB8 BDF3  7E9B 5912 A795 E90D D2CF
      created ....: 2016-05-24 23:29:03
Authentication key: 82BE 7837 6A3F 2E7B E556  5E35 3F29 127E 7964 9A3D
      created ....: 2016-05-24 23:36:40
General key info..: pub  4096R/0xBECFA3C1AE191D15 2016-05-24 Dr Duh <doc@duh.to>
sec#  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb>  4096R/0xBECFA3C1AE191D15  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
ssb>  4096R/0x5912A795E90DD2CF  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
ssb>  4096R/0x3F29127E79649A3D  created: 2017-10-09  expires: 2018-10-09
                      card-no: 0006 05553211
```

`sec#`表示主密钥不可用（因为它应该已离线加密存储）。

**注意** 如果您在输出中看到 `General key info..: [none]` - 返回并使用上一步导入公钥。

使用您自己的密钥将消息加密（对于存储密码凭据和其他数据很有用）：

```bash
$ echo "test message string" | gpg --encrypt --armor --recipient $KEYID -o encrypted.txt
```

要为多个收件人（或多个密钥）加密：

```bash
$ echo "test message string" | gpg --encrypt --armor --recipient $KEYID_0 --recipient $KEYID_1 --recipient $KEYID_2 -o encrypted.txt
```

解密消息：

```bash
$ gpg --decrypt --armor encrypted.txt
gpg: anonymous recipient; trying secret key 0x0000000000000000 ...
gpg: okay, we are the anonymous recipient.
gpg: encrypted with RSA key, ID 0x0000000000000000
test message string
```

签署消息：

```bash
$ echo "test message string" | gpg --armor --clearsign > signed.txt
```

验证签名：

```bash
$ gpg --verify signed.txt
gpg: Signature made Wed 25 May 2016 00:00:00 AM UTC
gpg:                using RSA key 0xBECFA3C1AE191D15
gpg: Good signature from "Dr Duh <doc@duh.to>" [ultimate]
Primary key fingerprint: 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
     Subkey fingerprint: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
```

使用[shell函数](https://github.com/drduh/config/blob/master/zshrc)使加密文件更容易：

```
secret () {
        output=~/"${1}".$(date +%s).enc
        gpg --encrypt --armor --output ${output} -r 0x0000 -r 0x0001 -r 0x0002 "${1}" && echo "${1} -> ${output}"
}

reveal () {
        output=$(echo "${1}" | rev | cut -c16- | rev)
        gpg --decrypt --output ${output} "${1}" && echo "${1} -> ${output}"
}
```

```bash
$ secret document.pdf
document.pdf -> document.pdf.1580000000.enc

$ reveal document.pdf.1580000000.enc
gpg: anonymous recipient; trying secret key 0xFF3E7D88647EBCDB ...
gpg: okay, we are the anonymous recipient.
gpg: encrypted with RSA key, ID 0x0000000000000000
document.pdf.1580000000.enc -> document.pdf
```

# 密钥轮替

PGP 不提供前向保密 - 泄露的密钥可用于解密所有过去的消息。 尽管 YubiKey 上存储的密钥很难被窃取，但这并非不可能 - 例如，密钥和 PIN 可能会被盗取，或者可能会在密钥硬件或用于创建它们的随机数生成器中发现漏洞。 因此，偶尔轮换子密钥是一个很好的做法。

当子密钥过期时，可以更新或更换。 这两个操作都需要访问离线主密钥。 通过更新子密钥的过期日期来更新子密钥表明您仍然拥有离线主密钥，并且更加方便。

另一方面，替换密钥不太方便，但更安全：新的子密钥将**无法**解密以前的消息、使用 SSH 进行身份验证等。联系人将需要接收更新的公钥，并且任何加密的秘密都需要解密并重新加密为新的子密钥才能使用。 这一过程在功能上相当于“丢失”YubiKey 并配置一个新的。 但是，您始终可以使用原始密钥的离线加密备份来解密以前的消息。

两种轮换方法都不是优越的，并且取决于个人的身份管理理念和个人威胁模型来决定使用哪一种，或者是否使子密钥过期。 理想情况下，子密钥应该是短暂的：每个加密、签名和身份验证事件仅使用一次，但实际上这对于 YubiKey 来说并不可行也不值得。 高级用户可能希望使用离线设备来实现更频繁的密钥轮换和简化配置。

## 设置环境

要更新或轮替子密钥，请遵循与生成密钥相同的过程：启动到安全环境，安装所需的软件并断开网络连接。

连接离线存储主密钥的存储设备并识别磁盘标签：

```bash
$ sudo dmesg | tail
mmc0: new high speed SDHC card at address a001
mmcblk0: mmc0:a001 SS16G 14.8 GiB (ro)
mmcblk0: p1 p2
```

解密并挂载离线卷：

```bash
$ sudo cryptsetup luksOpen /dev/mmcblk0p1 secret
Enter passphrase for /dev/mmcblk0p1:

$ sudo mount /dev/mapper/secret /mnt/encrypted-storage
```

将主密钥和配置导入到临时工作目录。 请注意，Windows 用户应导入 mastersub.gpg：

```bash
$ export GNUPGHOME=$(mktemp -d -t gnupg_$(date +%Y%m%d%H%M)_XXX)

$ gpg --import /mnt/encrypted-storage/tmp.XXX/mastersub.key

$ cp -v /mnt/encrypted-storage/tmp.XXX/gpg.conf $GNUPGHOME
```

编辑主密钥：

```bash
$ export KEYID=0xFF3E7D88647EBCDB

$ gpg --expert --edit-key $KEYID

Secret key is available
[...]
```

## 更新子密钥

更新子密钥更简单：您不需要生成新密钥、将密钥移至 YubiKey 或更新链接到 GPG 密钥的任何 SSH 公钥。 您需要做的就是更改与公钥关联的到期时间（这需要访问您刚刚加载的主密钥），然后导出该公钥并将其导入到您希望使用 **GPG 的任何计算机上 **（与 SSH 不同）密钥。

要更改所有子密钥的到期日期，请首先选择所有子密钥：

```bash
$ gpg --edit-key $KEYID

Secret key is available.

sec  rsa4096/0xFF3E7D88647EBCDB
    created: 2017-10-09  expires: never       usage: C
    trust: ultimate      validity: ultimate
ssb  rsa4096/0xBECFA3C1AE191D15
    created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
    created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
    created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> key 1

Secret key is available.

sec  rsa4096/0xFF3E7D88647EBCDB
     created: 2017-10-09  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb* rsa4096/0xBECFA3C1AE191D15
     created: 2017-10-09  expires: 2018-10-09  usage: S
ssb  rsa4096/0x5912A795E90DD2CF
     created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
     created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> key 2

Secret key is available.

sec  rsa4096/0xFF3E7D88647EBCDB
     created: 2017-10-09  expires: never       usage: C
     trust: ultimate      validity: ultimate
ssb* rsa4096/0xBECFA3C1AE191D15
     created: 2017-10-09  expires: 2018-10-09  usage: S
ssb* rsa4096/0x5912A795E90DD2CF
     created: 2017-10-09  expires: 2018-10-09  usage: E
ssb  rsa4096/0x3F29127E79649A3D
     created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> key 3

Secret key is available.

sec   rsa4096/0xFF3E7D88647EBCDB
      created: 2017-10-09  expires: never       usage: C
      trust: ultimate      validity: ultimate
ssb*  rsa4096/0xBECFA3C1AE191D15
      created: 2017-10-09  expires: 2018-10-09  usage: S
ssb*  rsa4096/0x5912A795E90DD2CF
      created: 2017-10-09  expires: 2018-10-09  usage: E
ssb*  rsa4096/0x3F29127E79649A3D
      created: 2017-10-09  expires: 2018-10-09  usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
```

然后，使用`expire`命令设置新的到期日期。 （尽管这个命令的名称叫做“过期”，但这不会导致当前有效的密钥过期。）

```bash
gpg> expire
Changing expiration time for a subkey.
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
```
按照这些提示设置新的到期日期，然后`保存`以保存更改。

接下来，导出公钥：

```bash
$ gpg --armor --export $KEYID > gpg-$KEYID-$(date +%F).asc
```

将该公钥传输到您使用 GPG 密钥的计算机，然后使用以下命令导入：

```bash
$ gpg --import gpg-0x*.asc
```

这将延长您的 GPG 密钥的有效性，并允许您使用它进行 SSH 授权。 请注意，您不需要更新位于远程服务器上的 SSH 公钥。

## 轮替子密钥

轮替子密钥涉及更多一点。 首先，按照原来的步骤生成每个子密钥。 先前的子密钥可以保留或从身份中删除。

通过导出新密钥来完成：

```bash
$ gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/mastersub.key

$ gpg --armor --export-secret-subkeys $KEYID > $GNUPGHOME/sub.key
```

将**新**临时工作目录复制到加密的离线存储，刚才我们并没有卸载它：

```bash
$ sudo cp -avi $GNUPGHOME /mnt/encrypted-storage
```

现在应该至少备份两个版本的主密钥和子密钥：

```bash
$ ls /mnt/encrypted-storage
lost+found  tmp.ykhTOGjR36  tmp.2gyGnyCiHs
```

卸载并关闭加密卷：

```bash
$ sudo umount /mnt/encrypted-storage

$ sudo cryptsetup luksClose /dev/mapper/secret
```

导出更新后的公钥：

```bash
$ sudo mkdir /mnt/public

$ sudo mount /dev/mmcblk0p2 /mnt/public

$ gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

$ sudo umount /mnt/public
```

断开存储设备并按照原来的步骤将新密钥（4、5 和 6）传输到 YubiKey，替换现有密钥。 重新启动或安全擦除 GPG 临时工作目录。

# 添加符号

可以将符号添加到用户 ID 中，并可以与 [Keyicide](https://keyicide.org) 结合使用来创建 [OpenPGP 身份证明](https://keyoxy.org/guides/openpgp-proofs) 。

添加符号需要访问主密钥，因此我们可以按照本指南的此[部分](#设置环境)中的设置说明进行操作。

请注意，无需将 Yubikey 连接到设置环境，也无需生成新密钥、将密钥移至 YubiKey 或更新链接到 GPG 密钥的任何 SSH 公钥。

完成环境设置后，可以遵循 Keyoxy [“指南”](https://keyoxy.org/guides/) 页面 **up 中“添加证明”下列出的任何指南，直到使用保存符号 `save`命令**。

此时就可以导出公钥了：

```bash
$ gpg --export $KEYID > pubkey.asc
```

现在可以将公钥传输到使用 GPG 密钥的计算机，并使用以下命令导入：

```bash
$ gpg --import pubkey.asc
```

注意：可以发出`showpref`命令来确保正确添加概念。

现在可以继续遵循 Keyoxy 指南并将密钥上传到 WKD 或 keys.openpgp.org。

# SSH

**提示** 如果您只想将 YubiKey 用于 SSH（并且并不真正关心 PGP/GPG），那么 [自 OpenSSH v8.2 起](https://www.openssh.com/txt/release-8.2)您也可以简单地执行`ssh-keygen -t ed25519-sk`（不需要本指南中的任何其他内容！），如[本指南中](https://github.com/vorburger/vorburger.ch-Notes/blob/develop/security/ed25519-sk.md)。 Yubico 最近还宣布在其蓝色“security key”上支持 OpenSSH 8.2+ 下的驻留 ssh 密钥，如其[博客文章](https://www.yubico.com/blog/github-now-supports-ssh-security-keys/)。

[gpg-agent](https://wiki.archlinux.org/index.php/GnuPG#SSH_agent) 支持 OpenSSH ssh-agent 协议 (`enable-ssh-support`)，以及 Windows 上的 Putty's Pageant  (`enable-putty-support`)。 这意味着它可以用来代替传统的 ssh-agent / pageant。 与 ssh-agent 存在一些差异，值得注意的是 gpg-agent 不会 *缓存* 密钥，而是将它们转换、加密并持久存储为 GPG 密钥，然后将它们提供给 ssh 客户端。 您想要保留在`gpg-agent`中的任何现有 ssh 私钥应在导入到 GPG agent 后删除。

将密钥导入到`gpg-agent`时，系统会提示您输入口令以保护 GPG 密钥存储中的该密钥 - 您也可以使用与原始 ssh 版本相同的口令。 GPG 可以在确定的时间内缓存口令（参考 `gpg-agent` 的各种 `cache-ttl` 选项），并且自 2.1 版本起可以通过 macOS 钥匙串存储和获取密码。 请注意，在导入到`gpg-agent`后删除旧私钥时，请保留`.pub`密钥文件以用于指定 ssh 身份（例如`ssh -i /path/to/identity.pub`）。

也许`gpg-agent`的 ssh 代理支持中缺少的最大的事情就是能够删除密钥。 `ssh-add -d/-D` 没有效果。 相反，您需要使用`gpg-connect-agent`实用程序来查找密钥的 keygrip，将其与所需的 ssh 密钥指纹（MD5）进行匹配，然后删除该 keygrip。 [gnupg-users 邮件列表](https://lists.gnupg.org/pipermail/gnupg-users/2016-August/056499.html) 有更多信息。

## 创建配置

通过下载 [drduh/config/gpg-agent.conf](https://github.com/drduh/config/blob/master/gpg-agent.conf) 为 gpg-agent 创建加固配置：

```bash
$ cd ~/.gnupg

$ wget https://raw.githubusercontent.com/drduh/config/master/gpg-agent.conf

$ grep -ve "^#" gpg-agent.conf
enable-ssh-support
default-cache-ttl 60
max-cache-ttl 120
pinentry-program /usr/bin/pinentry-curses
```

**重要** 当使用 YubiKey 作为智能卡时，`cache-ttl`选项**不**适用，因为 PIN 是[由智能卡本身缓存](https://dev.gnupg.org/T3362) 。 因此，为了从缓存中清除 PIN（智能卡相当于 `default-cache-ttl` 和 `max-cache-ttl`），您需要拔下 YubiKey，或者在编辑卡时将 `forcesig` 标志设置为 每次都会提示输入 PIN。

**提示** 设置 `pinentry-program /usr/bin/pinentry-gnome3` 以获得基于 GUI 的提示。 如果 *pinentry* 图形对话框未显示，并且您收到此错误：`sign_and_send_pubkey: signing failed: agent refused operation`，您可能需要安装`dbus-user-session`包并重新启动计算机以使`dbus`用户会话被完全继承； 这是因为在幕后，`pinentry`抱怨`No $DBUS_SESSION_BUS_ADDRESS found`，回退到`curses`，但没有找到预期的`tty`。

在 macOS 上，使用`brew install pinentry-mac`并将程序路径设置为`pinentry-program /usr/local/bin/pinentry-mac`（对于 Intel处理器），`/opt/homebrew/bin/pinentry-mac`（对于 ARM处理器） 或 `pinentry-program /usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac`（如果使用 MacGPG 套件）。 要使配置生效，您必须运行`gpgconf --kill gpg-agent`。

## 替换代理程序

要启动`gpg-agent`以供 SSH 使用，请使用`gpg-connect-agent /bye`或`gpgconf --launch gpg-agent`命令。

将这些添加到 shell `rc` 文件中：

```bash
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
gpg-connect-agent updatestartuptty /bye > /dev/null
```

在现代系统上，如果可用时`gpgconf --list-dirs agent-ssh-socket`会自动将`SSH_AUTH_SOCK`设置为正确的值，并且比硬编码为`run/user/$UID/gnupg/S.gpg-agent.ssh`要好：

```bash
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

如果您使用fish，`config.fish`的正确行将如下所示（考虑根据您的用例将它们放入`is-interactive`块中）：
```fish
set -x GPG_TTY (tty)
set -x SSH_AUTH_SOCK (gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

请注意，如果您使用`ForwardAgent`进行 ssh 代理转发，则只需在插入 YubiKey 的*本地*笔记本电脑（工作站）上设置`SSH_AUTH_SOCK`。在我们通过 SSH 连接的`远程`服务器上， 当我们连接时，`ssh`会自动将`SSH_AUTH_SOCK`设置为`/tmp/ssh-mXzCzYT2Np/agent.7541`这样。 因此，我们**不要**在服务器上手动设置 `SSH_AUTH_SOCK` - 这样做会破坏 [SSH代理转发](#SSH代理转发)。

如果您使用 `S.gpg-agent.ssh` （请参阅 [SSH代理转发](#SSH代理转发) 了解更多信息），还应在 *远程主机* 上设置 `SSH_AUTH_SOCK`。 但是，不应在*远程主机*上设置`GPG_TTY`，该部分中指定了解释。

## 复制公钥

**注意** 使用 SSH **不需要**导入相应的 GPG 公钥。

将`ssh-add`的输出复制并粘贴到服务器的`authorized_keys`文件中：

```bash
$ ssh-add -L
ssh-rsa AAAAB4NzaC1yc2EAAAADAQABAAACAz[...]zreOKM+HwpkHzcy9DQcVG2Nw== cardno:000605553211
```

## (可选)为身份文件配置保存公钥

默认情况下，SSH 尝试使用通过代理可用的所有身份。 准确管理 SSH 将使用哪些密钥连接到服务器通常是一个好主意，例如分离不同的角色或[避免被不受信任的 ssh 服务器指纹识别](https://blog.filippo.io/ssh-whoami-filippo-io/)。 为此，您需要使用命令行参数`-i [identity_file]`或`.ssh/config`中的`IdentityFile`和 `IdentitiesOnly`选项。

提供给 `IdentityFile` 的参数传统上是 *私钥* 文件的路径（例如 `IdentityFile ~/.ssh/id_rsa`）。 对于 YubiKey - 事实上，一般来说，对于存储在 ssh 代理中的密钥 - `IdentityFile` 应指向 *公钥* 文件，`ssh` 将从通过 ssh 代理提供的密钥中选择适当的私钥。 要防止`ssh`尝试代理中的所有密钥，请对目标主机使用`IdentitiesOnly yes`选项以及一个或多个`-i`或`IdentityFile`选项。

重申一下，使用`IdentitiesOnly yes`，`ssh`不会自动枚举加载到`ssh-agent`或`gpg-agent`中的公钥。 这意味着除非通过`ssh -i [identity_file]`或在每个主机的`.ssh/config`中显式命名，否则`公钥`身份验证将不会进行。

在使用 YubiKey 的情况下，从 ssh 代理提取公钥：

```bash
$ ssh-add -L | grep "cardno:000605553211" > ~/.ssh/id_rsa_yubikey.pub
```

然后，您可以显式关联此 YubiKey 存储的密钥以与主机`github.com`一起使用，例如，如下所示：

```bash
$ cat << EOF >> ~/.ssh/config
Host github.com
    IdentitiesOnly yes
    IdentityFile ~/.ssh/id_rsa_yubikey.pub
EOF
```

## 使用公钥认证连接

```bash
$ ssh git@github.com -vvv
[...]
debug2: key: cardno:000605553211 (0x1234567890),
debug1: Authentications that can continue: publickey
debug3: start over, passed a different list publickey
debug3: preferred gssapi-keyex,gssapi-with-mic,publickey,keyboard-interactive,password
debug3: authmethod_lookup publickey
debug3: remaining preferred: keyboard-interactive,password
debug3: authmethod_is_enabled publickey
debug1: Next authentication method: publickey
debug1: Offering RSA public key: cardno:000605553211
debug3: send_pubkey_test
debug2: we sent a publickey packet, wait for reply
debug1: Server accepts key: pkalg ssh-rsa blen 535
debug2: input_userauth_pk_ok: fp e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
debug3: sign_and_send_pubkey: RSA e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
debug1: Authentication succeeded (publickey).
[...]
```

**提示** 要建立多个连接或安全地传输多个文件，请考虑使用 [ControlMaster](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) ssh 选项。 另请参阅 [drduh/config/ssh_config](https://github.com/drduh/config/blob/master/ssh_config)。

## 导入SSH密钥

如果您希望通过`gpg-agent`提供现有的 SSH 密钥，则需要导入它们。 然后您应该删除原始私钥。 导入密钥时，`gpg-agent` 使用密钥的文件名作为密钥的标签； 这使得更容易追踪密钥的来源。 在此示例中，我们从 YubiKey 的密钥开始并导入 `~/.ssh/id_rsa`：

```bash
$ ssh-add -l
4096 SHA256:... cardno:00060123456 (RSA)

$ ssh-add ~/.ssh/id_rsa && rm ~/.ssh/id_rsa
```

调用`ssh-add`时，它将提示输入 SSH 密钥的口令（如果存在），然后`pinentry`程序将提示并确认新的口令，用于加密 GPG 密钥存储中转换后的密钥。

迁移的密钥将在`ssh-add -l`中列出：

```bash
$ ssh-add -l
4096 SHA256:... cardno:00060123456 (RSA)
2048 SHA256:... /Users/username/.ssh/id_rsa (RSA)
```

或者显示带有 MD5 指纹的密钥，如 `gpg-connect-agent` 的 `KEYINFO` 和 `DELETE_KEY` 命令所使用的：

```bash
$ ssh-add -E md5 -l
4096 MD5:... cardno:00060123456 (RSA)
2048 MD5:... /Users/username/.ssh/id_rsa (RSA)
```

使用密钥时，将调用`pinentry`来请求密钥的口令。 口令将被缓存 10 分钟的空闲时间（最多 2 小时）。

## 远程主机(SSH代理转发)

**注意** SSH 代理转发可以[增加额外风险](https://matrix.org/blog/2019/05/08/post-mortem-and-remediations-for-apr-11-security-incident/# ssh-agent-forwarding-should-be-disabled) - 谨慎操作！

ssh-agent转发有两种方法，一种是OpenSSH提供的，另一种是GnuPG提供的。

后一种可能更不安全，因为原始套接字只是转发（不像`S.gpg-agent.extra`那样只有有限的功能； 如果 OpenSSH 实现的`ForwardAgent`只是转发原始套接字，那么它们同样不安全）。 但对于后一种情况，一个方便之处在于，只需转发一次，就可以在远程的任何地方使用该代理。 所以再次强调，谨慎行事！

例如，当您 ssh 到远程并附加旧的`tmux`会话时，`tmux`没有一些环境变量，例如`$SSH_AUTH_SOCK`。 在这种情况下，如果您使用`ForwardAgent`，则需要手动查找套接字并为每个 shell`export SSH_AUTH_SOCK=/tmp/ssh-agent-xxx/xxxx.socket`。 但是，由于`S.gpg-agent.ssh`位于固定位置，因此可以在 shell rc 文件中将其用作 ssh-agent。

### 使用ssh-agent 

通过以上步骤，您已经成功配置了本地 ssh-agent。

您现在应该能够在_本地_计算机上使用`ssh -A remote`来登录_远程主机_，然后应该能够使用YubiKey，就像它连接到远程计算机一样。 例如，使用例如 该远程计算机上的`ssh-add -l`应显示来自 YubiKey 的公钥（`cardno:`）。 （如果您不想必须记住使用`ssh -A`，则可以在`~/.ssh/config`中使用`ForwardAgent yes`。作为安全最佳实践，始终仅在以下情况下使用`ForwardAgent yes`： 单个`主机名`，绝不适用于所有服务器。）

### 使用S.gpg-agent.ssh

首先你需要通过[远程主机（GPG代理转发）](#远程主机（GPG代理转发）)，了解gpg-agent转发的条件并知道`S.gpg-agent.ssh`的位置在本地和远程主机上。

您可以使用以下命令：

```bash
$ gpgconf --list-dirs agent-ssh-socket
```

然后在你的`.ssh/config`中为该远程主机添加一句话

```
Host
  Hostname remote-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent.ssh /run/user/1000/gnupg/S.gpg-agent.ssh
  # RemoteForward [remote socket] [local socket]
  # Note that ForwardAgent is not wanted here!
```

成功 ssh 到远程后，您应该检查那里是否有`/run/user/1000/gnupg/S.gpg-agent.ssh`。

然后在 *远程主机* 中，您可以在命令行中键入或在 shell rc 文件中配置：

```bash
export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
```

输入或sourcing shell rc 文件后，使用 `ssh-add -l` 您现在应该可以找到您的 ssh 公钥。

**注意** 在此过程中，不涉及远程中的 gpg-agent，因此远程中的`gpg-agent.conf`没有用处。 pinentry 也在本地调用。

### 链式SSH代理转发

如果您使用 OpenSSH 提供的·ssh-agent·并希望将其转发到*第三个主机*，您只需在*远程主机*上使用`ssh -A third`。

同时，如果您使用`S.gpg-agent.ssh`，假设您已完成上述步骤并且*远程主机*上有`S.gpg-agent.ssh`，并且您希望将此代理转发到 *第三个主机*，首先您可能需要以与*远程主机*相同的方式配置*第三个主机*的`sshd_config`和`SSH_AUTH_SOCK`，然后在*远程主机*的ssh配置中，添加以下行

```bash
Host third
  Hostname third-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent.ssh /run/user/1000/gnupg/S.gpg-agent.ssh
  # RemoteForward [remote socket] [local socket]
  # Note that ForwardAgent is not wanted here!
```

您应该根据 *远程主机* 和 *第三个主机* 上的 `gpgconf --list-dirs agent-ssh-socket` 更改路径。

## GitHub

您可以使用 YubiKey 对 GitHub 的留言和tag进行签名。 它还可用于 GitHub SSH 身份验证，允许您无需口令即可推送、拉取和提交。

登录 GitHub 并在“设置”中上传 SSH 和 PGP 公钥。

配置签名密钥：

> git config --global user.signingkey $KEYID

确保 user.email 选项与与 PGP 身份关联的电子邮件地址匹配。

现在，要签署提交或标签，只需使用`-S`选项即可。 GPG 将自动查询 YubiKey 并提示您输入 PIN。

进行身份验证：

**Windows**

运行以下命令：

```bash
git config --global core.sshcommand "plink -agent"

git config --global gpg.program 'C:\Program Files (x86)\GnuPG\bin\gpg.exe'
```

然后，您可以将存储库 URL 更改为 `git@github.com:USERNAME/repository`，任何经过身份验证的命令都将由 YubiKey 授权。

**注意** 如果您遇到错误`gpg: signing failed: No secret key` - 在插入 YubiKey 的情况下运行`gpg --card-status`，然后再次尝试 git 命令。

## OpenBSD

安装并启用与 PC/SC 驱动程序、卡、读卡器一起使用的工具，然后重新启动以识别 YubiKey：

```bash
$ doas pkg_add pcsc-tools

$ doas rcctl enable pcscd

$ doas reboot
```

## Windows

Windows 已经安装了一些虚拟智能卡读卡器，例如为 Windows Hello 提供的读卡器。 为了确保 scdaemon 使用的 YubiKey 是正确的，您应该将其添加到其配置中。 您将需要设备的全名。 要查找设备的全名，请插入 YubiKey 并打开 PowerShell 以运行以下命令：

``` powershell
PS C:\WINDOWS\system32> Get-PnpDevice -Class SoftwareDevice | Where-Object {$_.FriendlyName -like "*YubiKey*"} | Select-Object -ExpandProperty FriendlyName
Yubico YubiKey OTP+FIDO+CCID 0
```

根据型号的不同，名称略有不同。 感谢 [Scott Hanselman](https://www.hanselman.com/blog/HowToSetupSignedGitCommitsWithAYubiKeyNEOAndGPGAndKeybaseOnWindows.aspx) 分享此信息。

* 创建或编辑 `%APPDATA%/gnupg/scdaemon.conf` 以添加：

```
reader-port <your yubikey device's full name, e.g. Yubico YubiKey OTP+FIDO+CCID 0>
```

* 创建或编辑 `%APPDATA%/gnupg/gpg-agent.conf` 以添加：

```
enable-ssh-support
enable-putty-support
```

* 打开命令控制台，重新启动代理：

```
> gpg-connect-agent killagent /bye
> gpg-connect-agent /bye
```

* 输入 `> gpg --card-status` 查看 YubiKey 详细信息。
* 导入 [公钥](#导出公钥): `> gpg --import <公钥文件的路径>`
* 信任主密钥]
* 检索公钥 id：`> gpg --list-public-keys`
* 从 GPG 导出 SSH 密钥：`> gpg --export-ssh-key <公钥 id>`

将此密钥复制到文件中以供以后使用。 它代表与 YubiKey 上的秘密密钥对应的公共 SSH 密钥。 您可以将此密钥上传到您希望通过 SSH 访问的任何服务器。

创建一个指向`gpg-connect-agent /bye`的快捷方式，并将其放置在启动文件夹`shell:startup`中，以确保代理在系统关闭后启动。 修改快捷方式属性，使其在“最小化”窗口中启动，以避免启动时不必要的噪音。

现在您可以使用 PuTTY 进行公钥 SSH 身份验证。 当服务器请求公钥验证时，PuTTY 会将请求转发给 GPG，GPG 将提示您输入 PIN 并授权使用 YubiKey 登录。

### WSL

这里的目标是使 WSL 内的 SSH 客户端与您正在使用的 Windows 代理（在我们的例子中为 gpg-agent.exe）一起工作。 这是我们要实现的目标：
![WSL agent architecture](media/schema_gpg.png)

**注意** 这仅适用于 SSH 代理转发。 实际上不支持真正的GPG转发（加密/解密）。 请参阅 [weasel-pageant](https://github.com/vuori/weasel-pageant) 了解更多信息或考虑使用 [wsl2-ssh-pageant](https://github.com/BlackReloaded/wsl2-ssh-pageant) ），支持 SSH 和 GPG 代理转发。

#### 使用ssh-agent还是S.weasel-pegant

一种转发方式是 `ssh -A` （仍然需要 eval weasel 来设置本地 ssh-agent），并且仅依赖于 OpenSSH。 在这个赛道中，可能会涉及到 ssh/sshd 配置中的 `ForwardAgent` 和 `AllowAgentForwarding`； 但是，如果您使用其他方式（gpg ssh 套接字转发），则不应在 ssh 配置中启用 `ForwardAgent`。 有关详细信息，请参阅 [SSH代理转发](#SSH代理转发)。

另一种方法是转发 gpg ssh 套接字，如下所述。

#### 先决条件

* 适用于 WSL 的 Ubuntu 16.04 或更高版本
* Kleopatra
* [Windows configuration](#windows)

#### WSL配置

下载或克隆 [weasel-pageant](https://github.com/vuori/weasel-pageant)。

将 `eval $(/mnt/c/<提取路径>/weasel-pageant -r -a /tmp/S.weasel-pageant)` 添加到 shell rc 文件中。 此处使用命名套接字，以便可以在`~/.ssh/config”的“RemoteForward`指令中使用它。 使用`source ~/.bashrc`获取它。

使用`$ ssh-add -l`显示 SSH 密钥

编辑 `~/.ssh/config` 为要使用代理转发的每个主机添加以下内容：

```
RemoteForward <remote SSH socket path> /tmp/S.weasel-pageant
```

**注意** 远程 SSH 套接字路径可以通过 `gpgconf --list-dirs agent-ssh-socket` 找到

#### 远程主机配置

您可能需要将以下内容添加到 shell rc 文件中。

```
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
```

将以下内容添加到 `/etc/ssh/sshd_config` 中：

```
StreamLocalBindUnlink yes
```

并重新加载 SSH 守护进程（例如`sudo service sshd reload`）。

拔下 YubiKey、断开连接或重新启动。 重新登录 Windows，打开 WSL 控制台并输入 `ssh-add -l` - 您应该看不到任何内容。

插入 YubiKey，输入相同的命令以显示 ssh 密钥。

登录远程主机，您应该会出现 pinentry 对话框，要求输入 YubiKey pin。

在远程主机上，输入`ssh-add -l` - 如果您看到 ssh 密钥，则意味着转发有效！

**注意** 代理转发可以通过多个主机链接 - 只需遵循相同的[协议](#远程主机配置) 来配置每个主机。 您还可以在[链式SSH代理转发](#链式SSH代理转发) 上阅读这部分内容。

## macOS

要在 macOS 上使用 gui 应用程序，[需要进行更多设置](https://jms1.net/yubikey/make-ssh-use-gpg-agent.md)。

使用以下内容创建`$HOME/Library/LaunchAgents/gnupg.gpg-agent.plist`：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>gnupg.gpg-agent</string>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <false/>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/MacGPG2/bin/gpg-connect-agent</string>
            <string>/bye</string>
        </array>
    </dict>
</plist>
```

```bash
launchctl load $HOME/Library/LaunchAgents/gnupg.gpg-agent.plist
```

创建包含以下内容的`$HOME/Library/LaunchAgents/gnupg.gpg-agent-symlink.plist`：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/ProperyList-1.0/dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>gnupg.gpg-agent-symlink</string>
        <key>ProgramArguments</key>
        <array>
            <string>/bin/sh</string>
            <string>-c</string>
            <string>/bin/ln -sf $HOME/.gnupg/S.gpg-agent.ssh $SSH_AUTH_SOCK</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
    </dict>
</plist>
```

```bash
launchctl load $HOME/Library/LaunchAgents/gnupg.gpg-agent-symlink.plist
```

您需要重新启动或注销并重新登录才能激活这些更改。

# 远程主机(GPG代理转发)

本节与[SSH](#ssh)中的ssh-agent转发不同，因为gpg-agent转发具有更广泛的用途，不仅限于ssh。

要使用 YubiKey 在远程主机上签署 git 提交，或在远程主机上签署电子邮件/解密文件，请配置和使用 GPG 代理转发。 要通过另一个网络进行 ssh，尤其是使用 ssh 向 GitHub 推送/从 GitHub 拉取，请参阅[远程主机（SSH 代理转发）](#远程主机（SSH 代理转发）) 了解更多信息。

为此，您需要访问远程计算机，并且必须在主机上设置 YubiKey。

gpg-agent 转发后，几乎与将 YubiKey 插入远程相同。 因此，除了`gpg-agent.conf`之外，远程的配置可以与本地的配置相同。

**重要** `gpg-agent.conf` 对于远程来说是没有用的，因此 `$GPG_TTY` 对于远程来说也是没有用的。 其机制是，转发后，远程`gpg`直接与`S.gpg-agent`通信，而无需在远程*启动*`gpg-agent`。

在远程计算机上，编辑“/etc/ssh/sshd_config”以设置`StreamLocalBindUnlink yes`

**可选** 如果您没有远程计算机的根访问权限来编辑`/etc/ssh/sshd_config`，则需要删除套接字（位于`gpgconf --list-dir agent-socket`） 转发工作之前的远程计算机。 例如，`rm /run/user/1000/gnupg/S.gpg-agent`。 更多信息可以在 [AgentForwarding GNUPG wiki 页面](https://wiki.gnupg.org/AgentForwarding) 上找到。

将公钥导入到远程计算机。 这可以通过从密钥服务器获取来完成。 在本地计算机上，将公钥复制到远程计算机：

```bash
$ scp ~/.gnupg/pubring.kbx remote:~/.gnupg/
```

在现代发行版上，例如 Fedora 30，通常不需要在`~/.ssh/config`中设置`RemoteForward`，如下一章所述，因为正确的事情会自动发生。

如果现代发行版发生任何错误（或者远程中没有`gpg-agent.socket`），您可以执行下一节中的配置步骤。

## 旧发行版的步骤

在本地计算机上运行：

```bash
$ gpgconf --list-dirs agent-extra-socket
```

这应该返回代理额外套接字的路径 - `/run/user/1000/gnupg/S.gpg-agent.extra` - 尽管在较旧的 Linux 发行版（和 macOS）上它可能是 `/home/<user> /.gnupg/S/gpg-agent.extra`

在**远程**机器上找到代理套接字：

```bash
$ gpgconf --list-dirs agent-socket
```

这应该返回一个路径，例如`/run/user/1000/gnupg/S.gpg-agent`

最后，通过将以下内容添加到本地计算机的 ssh 配置文件`~/.ssh/config`（您的代理套接字可能不同）来启用给定计算机的代理转发：

```
Host
  Hostname remote-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent.extra
  # RemoteForward [remote socket] [local socket]
```

如果您仍然遇到问题，可能需要编辑 *本地* 计算机上的 `gpg-agent.conf` 文件以添加以下信息：

```
pinentry-program /usr/bin/pinentry-gtk-2
extra-socket /run/user/1000/gnupg/S.gpg-agent.extra
```

**注意** pinentry 程序在*本地*计算机上启动，而不是远程。 因此，当需要输入PIN码时，您需要在本地机器上找到提示。

**重要** 可以使用除`pinentry-tty`或`pinentry-curses`之外的任何 pinentry 程序。 这是因为本地 `gpg-agent` 可能会无头启动（通过 systemd 没有在本地设置 `$GPG_TTY` 来告诉它所在的 tty），从而无法获取 pin。 远程主机上的错误可能会产生误导，称存在*IO 错误*。 （是的，内部实际上存在一个*IO错误*，因为它发生在写入/读取tty而没有找到可使用的tty时，但对于最终用户来说这并不友好。）

有关更多信息和故障排除，请参阅[问题 #85](https://github.com/drduh/YubiKey-Guide/issues/85)。

## 链式GPG代理转发

假设您已完成上述步骤，并且*远程主机*上有`S.gpg-agent`，并且您想将此代理转发到*第三个*主机，首先您可能需要配置*第三个主机*的`sshd_config`  与*远程主机* 的方式相同，然后在*远程主机* 的ssh 配置中添加以下行：

```bash
Host third
  Hostname third-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent
  # RemoteForward [remote socket] [local socket]
```

您应该根据 *远程主机* 和 *第三个主机* 上的 `gpgconf --list-dirs agent-socket` 更改路径。

**注意** 在 *本地主机* 上，您有 `S.gpg-agent.extra`，而在 *远程主机* 和 *第三个主机* 上，您只有 `S.gpg-agent`。

# 使用多个密钥

要使用具有多个 YubiKey 的单一身份 - 或用另一张卡替换丢失的卡 - 发出以下命令来切换密钥：

```bash
$ gpg-connect-agent "scd serialno" "learn --force" /bye
```

或者，使用脚本删除存储卡序列号的 GnuPG 隐藏密钥（请参阅 [GnuPG #T2291](https://dev.gnupg.org/T2291)）：

```bash
$ cat >> ~/scripts/remove-keygrips.sh <<EOF
#!/usr/bin/env bash
(( $# )) || { echo "Specify a key." >&2; exit 1; }
KEYGRIPS=$(gpg --with-keygrip --list-secret-keys "$@" | awk '/Keygrip/ { print $3 }')
for keygrip in $KEYGRIPS
do
    rm "$HOME/.gnupg/private-keys-v1.d/$keygrip.key" 2> /dev/null
done

gpg --card-status
EOF

$ chmod +x ~/scripts/remove-keygrips.sh

$ ~/scripts/remove-keygrips.sh $KEYID
```

请参阅问题 [#19](https://github.com/drduh/YubiKey-Guide/issues/19) 和 [#112](https://github.com/drduh/YubiKey-Guide/issues/112) 中的讨论了解更多信息和故障排除步骤。

# 需要触摸

**注意** 这在 YubiKey NEO 上是不可能的。

默认情况下，插入密钥并首次使用 PIN 解锁后，YubiKey 将执行加密、签名和身份验证操作，无需用户执行任何操作。

如需每次按键操作都需要触摸，请安装 [YubiKey Manager](https://developers.yubico.com/yubikey-manager/) 并调用Admin PIN：

**注意** 旧版本的 YubiKey Manager 在以下命令中使用`touch`而不是`set-touch`。

身份验证：

```bash
$ ykman openpgp keys set-touch aut on
```

签名：

```bash
$ ykman openpgp keys set-touch sig on
```

加密：

```bash
$ ykman openpgp keys set-touch enc on
```

根据 YubiKey 的使用方式，您可能需要查看每个选项的策略选项并相应地调整上述命令。 可以使用以下命令查看它们：

```
$ ykman openpgp keys set-touch -h
Usage: ykman openpgp keys set-touch [OPTIONS] KEY POLICY

  Set touch policy for OpenPGP keys.

  KEY     Key slot to set (sig, enc, aut or att).
  POLICY  Touch policy to set (on, off, fixed, cached or cached-fixed).

  The touch policy is used to require user interaction for all operations using the private key on the YubiKey. The touch policy is set individually for each key slot. To see the current touch policy, run

      $ ykman openpgp info

  Touch policies:

  Off (default)   No touch required
  On              Touch required
  Fixed           Touch required, can't be disabled without a full reset
  Cached          Touch required, cached for 15s after use
  Cached-Fixed    Touch required, cached for 15s after use, can't be disabled
                  without a full reset

Options:
  -a, --admin-pin TEXT  Admin PIN for OpenPGP.
  -f, --force           Confirm the action without prompting.
  -h, --help            Show this message and exit.
```

如果要在打开和验证加密邮件的电子邮件客户端中使用 YubiKey，则可能需要`Cached`或`Cached-Fixed`。

YubiKey 在等待触摸时会闪烁。 在 Linux 上，您还可以使用 [yubikey-touch- detector](https://github.com/maximbaz/yubikey-touch- detector) 来显示 YubiKey 正在等待触摸的指示器或通知。

# Email

YubiKey 上的 GPG 密钥可轻松使用 [Thunderbird](https://www.thunderbird.net/)、[Enigmail](https://www.enigmail.net) 和 [Mutt](http://www.mutt.org/)。 Thunderbird 支持 OAuth 2 身份验证，可以与 Gmail 一起使用。 有关详细说明，请参阅 EFF 的[指南](https://ssd.eff.org/en/module/how-use-pgp-linux)。 Mutt 从 2.0 版开始就支持 OAuth 2。

## Mailvelope

[Mailvelope](https://www.mailvelope.com/en) 允许 YubiKey 上的 GPG 密钥与 Gmail 等一起使用。

**重要** 如果在 `gpg.conf` 中设置了 `throw-keyids` 选项,则Mailvelope [不起作用](https://github.com/drduh/YubiKey-Guide/issues/178)。

在 macOS 上，使用 Homebrew 安装 gpgme：

```bash
$ brew install gpgme
```

要允许 Chrome 运行 gpgme，请编辑 `~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/gpgmejson.json` 并添加：

```json
{
    "name": "gpgmejson",
    "description": "Integration with GnuPG",
    "path": "/usr/local/bin/gpgme-json",
    "type": "stdio",
    "allowed_origins": [
        "chrome-extension://kajibbejlbohfaggdiogboambcijhkke/"
    ]
}
```

编辑默认路径以允许 Chrome 查找 GPG：

```bash
$ sudo launchctl config user path /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

最后，从 Chrome 应用商店安装 [Mailvelope 扩展](https://chrome.google.com/webstore/detail/mailvelope/kajibbejlbohfaggdiogboambcijhkke)。

## Mutt

Mutt同时具有CLI和TUI界面，后者为日常电子邮件处理提供了强大的功能。 此外，还可以集成PGP，这样签名/加密/验证/解密就可以在不离开TUI的情况下完成。

要启用 GnuPG 支持，只需使用 mutt 提供的配置文件`gpg.rc`即可，安装后通常位于`/usr/share/doc/mutt/samples/gpg.rc`。 只需要编辑`pgp_default_key`、`pgp_sign_as`和`pgp_autosign`等选项即可。 编辑后，`source`此 `rcfile`即可以在其主`muttrc`中来使用它。

**重要** 据报道，如果一个人在`gpg-agent.conf`中使用`pinentry-tty`作为自己的 pinentry 程序，它会弄乱一个人的 Mutt TUI。 这是因为 Mutt TUI 使用诅咒，而 tty 输出可能会损害格式。 推荐使用`pinentry-curses`或其他图形pinentry程序。

# 重置

如果超过 PIN 尝试次数，卡将被锁定，并且必须[重置](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html)并使用加密备份重新设置。

将以下脚本复制到文件并运行`gpg-connect-agent -r $file`以锁定并终止卡。 然后重新插入YubiKey进行重置。

```bash
/hex
scd serialno
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 e6 00 00
scd apdu 00 44 00 00
/echo Card has been successfully reset.
```

或者使用 `ykman` （有时在 `~/.local/bin/` 中）：

```bash
$ ykman openpgp reset
WARNING! This will delete all stored OpenPGP keys and data and restore factory settings? [y/N]: y
Resetting OpenPGP data, don't remove your YubiKey...
Success! All data has been cleared and default PINs are set.
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

## 重置后恢复

如果出于某种原因您需要从主密钥备份（例如[备份](#备份)中描述的存储在加密 USB 上的备份）恢复 YubiKey，请按照[轮替密钥](#轮替密钥)中的以下步骤操作 ）设置您的环境，然后再次按照[配置智能卡](#配置智能卡)的步骤操作。

在卸载备份之前，问问自己是否应该再制作一个以防万一。

# 注释

1. YubiKey 有两种配置：一种是短按调用，另一种是长按调用。 默认情况下，短按模式配置为 HID OTP - 短暂触摸将发出以`cccccccc`开头的 OTP 字符串。 如果您很少使用 OTP 模式，可以通过 YubiKey 个性化工具将其切换到第二种配置。 如果您“从不”使用 OTP，则可以使用 [YubiKey Manager](https://developers.yubico.com/yubikey-manager) 应用程序完全禁用它（注意，这不是类似名称的旧版 YubiKey NEO Manager）。 使用 ykman 禁用 OTP 的命令是`ykman config usb -d OTP`。
1. 已进行过 GPG 密钥编程的 YubiKey 仍然允许您使用其其他配置 - [U2F](https://en.wikipedia.org/wiki/Universal_2nd_Factor)、[OTP](https://www.yubico.com/faq/what-is-a-one-time-password-otp/)和[静态密码](https://www.yubico.com/products/services-software/personalization-tools/static-password/)模式。
1. 设置过期实际上会强制您管理您的子密钥并向世界其他地方宣布您正在这样做。 在主密钥上设置过期时间对于防止密钥丢失是无效的 - 拥有主键的人可以简单地延长其过期时间。 吊销证书[更适合](https://security.stackexchange.com/questions/14718/does-openpgp-key-expiration-add-to-security/79386#79386)用于此目的。 设置子密钥的到期日期可能适合您的用例。
1. 要在不同密钥上的两个或多个身份之间切换 - 拔下第一个密钥并使用 `pkill gpg-agent ; pkill ssh-agent ; pkill pinentry ; eval $(gpg-agent --daemon --enable-ssh-support)`，然后插入另一个密钥并运行 `gpg-connect-agent updatestartuptty /bye` - 然后它应该可以使用了。
1. 要在多台带有 gpg 的计算机上使用 yubikeys： 初始设置后，在第二个工作站上导入公钥。 确认 gpg 可以通过 `gpg --card-status` 看到该卡，信任您最终导入的公钥（如上所述）。 此时`gpg --list-secret-keys`应该显示您的（可信）密钥。

# 疑难解答

- 使用`man gpg`来了解 GPG 选项和命令行参数。

- 要获取有关潜在错误的更多信息，请重新启动`gpg-agent`进程，并使用 `pkill gpg-agent; gpg-agent --daemon --no-detach -v -v --debug-level advanced --homedir ~/.gnupg`将debug信息输出到控制台。

- 如果您在使用 GPG 连接到 YubiKey 时遇到问题 - 尝试拔下并重新插入 YubiKey，然后重新启动 `gpg-agent` 进程。

- 如果您收到错误`gpg: decryption failed: secret key not available` - 您可能需要安装 GnuPG 版本 2.x。 另一种可能性是 PIN 码有问题，例如 它太短或被阻止。

- 如果您收到错误`Yubikey core error: no yubikey present` - 请确保 YubiKey 已正确插入，如果已插入请重新插入。 插入时它应该闪烁一次。

- 如果您仍然收到错误 `Yubikey core error: no yubikey present` - 您可能需要安装较新版本的 yubikey-personalize，如 [所需软件](#所需软件) 中所述。

- 如果您收到错误`Yubikey core error: write error` - YubiKey 可能已锁定。 安装并运行 yubikey-personalization-gui 以解锁它。

- 如果您收到错误`Key does not match the card's capability` - 您可能需要使用RSA-2048。**译者注：**也可能是固件版本过低

- 如果您收到错误`sign_and_send_pubkey: signing failed: agent refused operation` - 请确保将`ssh-agent`替换为`gpg-agent`。

- 如果您仍然收到错误，`sign_and_send_pubkey: signing failed: agent refused operation` - [运行命令](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394)`gpg-connect-agent updatestartuptty /bye`

- 如果您仍然收到错误`sign_and_send_pubkey: signing failed: agent refused operation` - 编辑`~/.gnupg/gpg-agent.conf`以设置有效的`pinentry`程序路径，例如 macOS 上的 `pinentry-program /usr/local/bin/pinentry-mac`。

- 如果您仍然收到错误`sign_and_send_pubkey: signing failed: agent refused operation` - 这是一个[已知问题](https://bbs.archlinux.org/viewtopic.php?id=274571)，在openssh 8.9p1 及更高版本中使用 YubiKey 存在问题。 将`KexAlgorithms -sntrup761x25519-sha512@openssh.com` 添加到`/etc/ssh/ssh_config`通常可以解决该问题。

- 如果您从`ssh-add -L`收到错误`The agent has no identities`，请确保您已安装并启动`scdaemon`。

- 如果您从`ssh-add -L`收到错误`Error connecting to agent: No such file or directory`，则代理用于与其他进程通信的 UNIX 文件套接字可能未正确设置。 在 Debian 上，尝试 `export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"`。 另外，执行`gpgconf --list-dirs agent-ssh-socket`返回现有`S.gpg-agent.ssh`套接字的路径。

- 如果您收到错误`Permission denied (publickey)`，请使用`-v`参数查看 ssh 详细输出，并确保已提供来自卡的公钥：`Offering public key: RSA SHA256:abcdefg... cardno:00060123456`。 如果是，请确保您以目标系统上的正确用户身份进行连接，而不是以本地系统上的用户身份进行连接。 否则，请确保该主机未[启用](https://github.com/FiloSottile/whosthere#how-do-i-stop-it)`IdentitiesOnly`。

- 如果 SSH 身份验证仍然失败 - 向`ssh`客户端添加最多 3 个`-v`参数以增加输出的详细程度。

- 如果仍然失败，则停止服务器上的后台`sshd`守护进程进程服务（例如使用`sudo systemctl stop sshd`）并使用`/usr/sbin/sshd -eddd`在前台启动它以查看调试输出。 请注意，当前状态服务器只会处理一个连接，因此在每次·ssh·测试后都必须重新启动服务。

- 如果您收到错误消息`Please insert the card with serial number: *`，请参阅[使用多个密钥](#使用多个密钥)。

- 如果您收到错误`There is no assurance this key belongs to the named user`或`encryption failed: Unusable public key`，请使用`gpg --edit-key`将`trust`设置为`5 = I trust ultimately`。

- 如果当您尝试上述`--edit-key`命令时，收到错误`Need the secret key to do this` - 使用`trust-key [key ID]`指令手动指定对`~/.gnupg/gpg.conf`中密钥的信任。

- If, when using a previously provisioned YubiKey on a new computer with `pass`, you see the following error on `pass insert`, you need to adjust the trust associated with the key. See the note above. **译文：**如果在带有`pass`的新计算机上使用之前配置的 YubiKey 时，您在`pass insert`上看到以下错误，则需要调整与密钥关联的信任。 请参阅上面的注释。

```
gpg: 0x0000000000000000: There is no assurance this key belongs to the named user
gpg: [stdin]: encryption failed: Unusable public key
```

- 如果您收到错误`gpg: 0x0000000000000000: skipped: Unusable public key`、`signing failed: Unusable secret key`或`encryption failed: Unusable public key`，则子密钥可能已过期，无法再用于加密或签名消息。 但是，它仍然可以用于解密和身份验证。

- 如果您丢失了 GPG 公钥，请按照[本指南](https://www.nicksherlock.com/2021/08/recovering-lost-gpg-public-keys-from-your-yubikey/)从 YubiKey 恢复它。

- 请参阅 Yubico 文章 [GPG 问题疑难解答](https://support.yubico.com/hc/en-us/articles/360013714479-Troubleshooting-Issues-with-GPG) 了解更多指导。

# 备选方案

* [`piv-agent`](https://github.com/smlx/piv-agent) 是一个 SSH 和 GPG 代理，您可以将其与 PIV 硬件安全设备（例如 Yubikey）一起使用。
* [`keytotpm`](https://www.gnupg.org/documentation/manuals/gnupg/OpenPGP-Key-Management.html) 是在 TPM 系统中使用 GnuPG 的一个选项。

## 使用批处理创建密钥

还可以使用模板文件和`batch`参数生成密钥 - 请参阅 [GnuPG 文档](https://www.gnupg.org/documentation/manuals/gnupg/Unattended-GPG-key-generation.html)。

从 [gen-params-rsa4096](contrib/gen-params-rsa4096) 模板开始。 如果您使用的是 GnuPG v2.1.7 或更高版本，您还可以使用 [gen-params-ed25519](contrib/gen-params-ed25519) 模板。这些模板不会将主密钥设置为过期 - 请参阅[注释 #3](#注释)。

生成主密钥：

```bash
$ gpg --batch --generate-key gen-params-rsa4096
gpg: Generating a basic OpenPGP key
gpg: key 0xEA5DE91459B80592 marked as ultimately trusted
gpg: revocation certificate stored as '/tmp.FLZC0xcM/openpgp-revocs.d/D6F924841F78D62C65ABB9588B461860159FFB7B.rev'
gpg: done
```

验证结果：

```bash
$ gpg --list-key
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
/tmp.FLZC0xcM/pubring.kbx
-------------------------------
pub   rsa4096/0xFF3E7D88647EBCDB 2021-08-22 [C]
       Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                   [ultimate] Dr Duh <doc@duh.to>
```

密钥指纹（`011C E16B D45B 27A5 5BA8 776D FF3E 7D88 647E BCDB`）将用于创建用于签名、身份验证和加密的三个子密钥。

现在创建用于签名、身份验证和加密的三个子密钥。 子密钥的有效期为 1 年 - 可以使用离线主密钥更新它们，请参阅[轮替密钥](#轮替密钥)。

我们将使用 GNUPG 的快速密钥操作界面（使用 `--quick-add-key`），请参阅[文档](https://www.gnupg.org/documentation/manuals/gnupg/Unattended-GPG-key-generation.html#Unattended-GPG-key-generation)。

创建一个[签名子密钥](https://stackoverflow.com/questions/5421107/can-rsa-be-both-used-as-encryption-and-signature/5432623#5432623)：

```bash
$ gpg --quick-add-key "011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB" \
  rsa4096 sign 1y
```

现在创建一个[加密子密钥](https://www.cs.cornell.edu/courses/cs5430/2015sp/notes/rsa_sign_vs_dec.php)：

```bash
$ gpg --quick-add-key "011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB" \
  rsa4096 encrypt 1y
```

最后，创建一个[身份验证子密钥](https://superuser.com/questions/390265/what-is-a-gpg-with-authenticate-capability-used-for)：

```bash
$ gpg --quick-add-key "011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB" \
  rsa4096 auth 1y
```

继续本指南的[验证](#验证)部分。

# Links

* https://alexcabal.com/creating-the-perfect-gpg-keypair/
* https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO
* https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/
* https://blog.onefellow.com/post/180065697833/yubikey-forwarding-ssh-keys
* https://developers.yubico.com/PGP/
  * https://developers.yubico.com/PGP/Card_edit.html
* https://developers.yubico.com/yubikey-personalization/
* https://evilmartians.com/chronicles/stick-with-security-yubikey-ssh-gnupg-macos
* https://gist.github.com/ageis/14adc308087859e199912b4c79c4aaa4
* https://github.com/herlo/ssh-gpg-smartcard-config
* https://github.com/tomlowenthal/documentation/blob/master/gpg/smartcard-keygen.md
* https://help.riseup.net/en/security/message-security/openpgp/best-practices
* https://jclement.ca/articles/2015/gpg-smartcard/
* https://rnorth.org/gpg-and-ssh-with-yubikey-for-mac
* https://trmm.net/Yubikey
* https://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/
* https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/
* https://www.hanselman.com/blog/HowToSetupSignedGitCommitsWithAYubiKeyNEOAndGPGAndKeybaseOnWindows.aspx
* https://www.void.gr/kargig/blog/2013/12/02/creating-a-new-gpg-key-with-subkeys/
* https://mlohr.com/gpg-agent-forwarding/
* https://www.ingby.com/?p=293
* https://support.yubico.com/support/solutions/articles/15000027139-yubikey-5-2-3-enhancements-to-openpgp-3-4-support
* https://github.com/dhess/nixos-yubikey
