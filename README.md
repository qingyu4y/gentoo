# Gentoo Install

## 注意事项

进入宿主系统之后和 chroot 进入新环境之后，应该首先设置时区和调整时间。如果时间不正确可能会导致一些意想不到的错误。

## Install && Configure Gentoo

### 参考

[Gentoo AMD64 Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/zh-cn)

[最快的Gentoo Linux安装思路！](https://www.bilibili.com/video/BV1nS4y1j7Uw?spm_id_from=333.880.my_history.page.click)

### 开始

1. 同步时间
   ```bash
   ntpd -q -g
   ```

2. 安装环境使用 archlinux 的镜像即可，挺好用的。

3. 分区并挂载 `/` `/home` `/boot/efi` `swap` （这一步和安装 archlinux 时一样）

4. 下载 Gentoo 的 [stage3](https://mirrors.ustc.edu.cn/gentoo/releases/amd64/autobuilds/) 包，并解压到准备的新环境中

   ```bash
   tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
   ```

5. 生成 fstab（同 archlinux）
   使用 `git clone https://github.com/qingyu4y/gentoo` 下载 archlinux 的 genfstab 工具

6. 复制 DNS 信息到新环境

   ```bash
   # --dereference 选项可以保障如果/etc/resolv.conf是一个符号链接的话，复制的是那个目标文件而不是这个符号文件自己
   cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
   ```

7. 挂载必要的文件系统

   ```bash
   mount --types proc /proc /mnt/gentoo/proc
   mount --rbind /sys /mnt/gentoo/sys
   mount --rbind /dev /mnt/gentoo/dev
   mount --bind /run /mnt/gentoo/run
   # 如果使用 systemd，还需要挂载下面三个文件系统
   mount --make-rslave /mnt/gentoo/sys
   mount --make-rslave /mnt/gentoo/dev
   mount --make-rslave /mnt/gentoo/run
   # 如果使用的不是官方的安装镜像，则还需要执行下面的步骤；因为使用的是 archlinux 的安装镜像，所以下面的步骤是不要的。使用 archlinux 安装镜像时，通常只需要执行第二个挂载命令即可，如果要确保万无一失，可以再执行一下第三个命令。 
   test -L /dev/shm && rm /dev/shm && mkdir /dev/shm
   mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm
   chmod 1777 /dev/shm /run/shm
   ```

8. 进入新环境

   ```bash
   chroot /mnt/gentoo /bin/bash
   source /etc/profile
   export PS1="(chroot) ${PS1}"
   ```

9.  配置 Portage

   - */etc/portage/make.conf*

     ```
     # These settings were set by the catalyst build script that automatically
     # built this stage.
     # Please consult /usr/share/portage/config/make.conf.example for a more
     # detailed example.
     COMMON_FLAGS="-march=znver3 -O2 -pipe"
     CFLAGS="${COMMON_FLAGS}"
     CXXFLAGS="${COMMON_FLAGS}"
     FCFLAGS="${COMMON_FLAGS}"
     FFLAGS="${COMMON_FLAGS}"
     
     # NOTE: This stage was built with the bindist Use flag enabled
     PORTDIR="/var/db/repos/gentoo"
     DISTDIR="/var/cache/distfiles"
     PKGDIR="/var/cache/binpkgs"
     
     # This sets the language of build output to English.
     # Please keep this setting intact when reporting bugs.
     LC_MESSAGES=C
     
     # 使用 cpuid2cpuflags 获取，如果没有，使用 emerge --ask app-portage/cpuid2cpuflags 命令安装
     CPU_FLAGS_X86=""
     # EMERGE_DEFAULT_OPTS="--keep-going --with-bdeps=y --autounmask-write=y --jobs=2 -l"
     EMERGE_DEFAULT_OPTS="--jobs=2"
     NO_DE="-wayland -gnome -gnome-shell -gnome-online-accounts -gnome-keyring -nautilus -kde -plasma"
     OTHERS="X systemd pipewire -grub -ipv6 -bindist cjk emoji"
     USE="${NO_DE} ${OTHERS}"
     GENTOO_MIRRORS="https://mirrors.ustc.edu.cn/gentoo/"
     ACCEPT_LICENSE="*"
     MAKEOPTS="-j5"
     VIDEO_CARDS="nvidia"
     GRUB_PLATFORMS="efi-64"
     ACCEPT_KEYWORDS="amd64"
     # LUA_SINGLE_TARGET="luajit"
     # USE_EXPAND="en_US zh_CN en zh"
     L10N="en-US zh-CN en zh"
     LINGUAS="en_US zh_CN en zh"
     ```

   - */etc/portage/repos.conf*

     因为默认没有安装 git，所以现在只能使用 rsync 的同步方式。

     ```
     [DEFAULT]
     main-repo = gentoo
     
     [gentoo]
     location = /var/db/repos/gentoo
     sync-type = rsync
     sync-uri = rsync://rsync.mirrors.ustc.edu.cn/gentoo-portage
     auto-sync = yes
     ```

11. 更新 Portage

   ```bash
   # 推荐使用第一种方式，虽然它使用快照的方式不能保证软件是最新的，但其实差不了多少。
   emerge-webrsync
   # 这种方式主要是速度太慢了
   emerge --sync
   ```

11. （可选）选择 profile

    通常当前环境的默认的 profile 与下载的 stage3 的是一致的。比如下载的是 amd64 + systemd + desktop 版本的 stage3 包，那么当前环境的 profile 默认设置的就是 `default/linux/amd64/17.1/desktop/systemd`。所以一般无需更改。

    ```bash
    # 使用如下命令查看全部可用 profile
    eselect profile list
    # 设置
    eselect profile set <number>
    ```

12. （可选）更新系统

    如果没有特殊情况并且是下载的最新的 stage3 包，其实不用太急着更新整个系统，因为真的很耗时。

    ```bash
    emerge --ask --verbose --update --deep --newuse @world
    ```

13. 进行一些必要的配置

    - 时区

      ```bash
      # OpenRC
      echo "Asia/Shanghai" > /etc/timezone
      
      # systemd
      ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
      ```

    - locale

      先编辑 locale.conf 然后执行 `locale-gen` 命令来应用修改（这同 archlinux 一致）。最后这里需要使用 `eselect locale set <number>` 来设置一下。

    - 更新环境

      ```bash
      env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
      ```

14. 安装固件程序

    为一些硬件，特别是无线设备提供了驱动。如果不安装可能导致开机启动失败。

    ```bash
    emerge -av sys-kernel/linux-firmware
    ```

15. 安装内核

    为了减少安装时间，使用 gentoo 编译好的内核。

    ```bash
    emerge --ask sys-kernel/gentoo-kernel-bin
    ```

16. 配置网络 iwd

    因为只使用无线网络，所以只要 iwd 就够。

    安装

    ```bash
    emerge -av net-wireless/iwd
    ```

    配置 */etc/iwd/main.conf*

    ```
    [General]
    EnableNetworkConfiguration=true
    
    [Network]
    NameResolvingService=systemd
    ```

    如果使用的 OpenRC，则还需要安装 net-dns/openresolv，并且在编译时需要开启 `standalone` flag。如果这里设置错误，很显然将会导致域名不能解析为具体的 ip 地址。

    ```bash
    emerge -av net-dns/openresolv
    ```

    相应的需要修改配置

    ```
    [General]
    EnableNetworkConfiguration=true
    [Network]
    RoutePriorityOffset=200
    NameResolvingService=resolvconf
    ```

17. 安装引导程序

- 使用 grub
  ```bash
  emerge -av sys-boot/grub
  grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=Gentoo
  vi /etc/default/grub
  grub-mkconfig -o /boot/grub/grub.cfg
  ```
  `/etc/default/grub`
  ```
  GRUB_CMDLINE_LINUX="quiet amd_iommu=on iommu=pt loglevel=5"
  ```

- 使用 LiLo

    如果使用的是传统的 MBR，那么可以选择更轻量的 LiLo 引导程序。

    安装

    ```bash
    emerge -av sys-boot/lilo
    ```

    配置

    ```
    # Author: Ultanium
    
    #
    # Start LILO global section
    #
    
    # Faster, but won't work on all systems:
    compact
    # Should work for most systems, and do not have the sector limit:
    lba32
    # If lba32 do not work, use linear:
    #linear
    
    # MBR to install LILO to:
    boot = /dev/sda
    map = /boot/.map
    
    # If you are having problems booting from a hardware raid-array
    # or have a unusual setup, try this:
    #disk=/dev/ataraid/disc0/disc bios=0x80  # see this as the first BIOS disk
    #disk=/dev/sda bios=0x81                 # see this as the second BIOS disk
    #disk=/dev/hda bios=0x82                 # see this as the third BIOS disk
    
    # Here you can select the secondary loader to install.  A few
    # examples is:
    #
    #    boot-text.b
    #    boot-menu.b
    #    boot-bmp.b
    #
    # install = /boot/boot-menu.b   # Note that for lilo-22.5.5 or later you
                                  # do not need boot-{text,menu,bmp}.b in
                                  # /boot, as they are linked into the lilo
                                  # binary.
    
    menu-scheme=Wb
    prompt
    # If you always want to see the prompt with a 15 second timeout:
    timeout=50
    delay = 50
    # Normal VGA console
    vga = normal
    # VESA console with size 1024x768x16:
    #vga = 791
    
    #
    # End LILO global section
    #
    
    #
    # Linux bootable partition config begins
    #
    image = /boot/vmlinuz-5.15.25-gentoo-dist
    	root = /dev/sda2
    	#root = /devices/discs/disc0/part3
    	label = Gentoo
    	read-only # read-only for checking
    	append="quiet"
    ```

1.  最后的配置

    - 设置 root 密码

    - 新建用户并设置密码

    - 设置 hostname

      - OpenRC

        */etc/conf.d/hostname*

        ```bash
        hostname=<name>
        ```

      - systemd

        ```
        hostnamectl hostname <name>
        ```
    - 配置硬件时钟
  
      `vi /etc/conf.d/hwclock`
      ```
      clock="local"
      ```

    - 配置密码规则
  
      `vi /etc/security/passwdqc.conf`

2.  现在可以愉快的重启进入新系统了



## 遇到问题的解决思路

### 开机启动失败

使用安装环境 chroot 进去然后使用 `dmesg` 命令查看启动日志。
