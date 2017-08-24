---
layout: post
title: preseed自动化部署debian
description: 采用preseed 批量自动化部署debian
categories:
  - Debian
---

### 使用dnsmasq搭建网络启动服务器

安装dnsmasq 编辑配置文件/etc/dnsmasq.conf

    interface=vboxnet0              #绑定的网卡
    domain=jumeicd
    dhcp-range=192.168.100.3,192.168.100.253,255.255.255.0,1h #dhcp 服务器设置
    dhcp-option=3,192.168.100.1     #默认网关
    dhcp-option=6,192.168.100.1     #客服端 获取到的dns服务器地址
    dhcp-boot=pxelinux.0,pxeserver,192.168.100.2
    pxe-service=x86PC, "Install Linux", pxelinux
    enable-tftp                     #开启tftp
    tftp-root=/srv/tftp             #tftp目录
    conf-dir=/etc/dnsmasq.d

### 使用官方提供的网络启动镜像实现网络开机

1. 下载镜像

    wget http://mirrors.ustc.edu.cn/debian/dists/squeeze/main/installer-amd64/current/images/netboot/netboot.tar.gz -P  /tmp

2. 保存在tftp的根目录

    cd /srv/tftp
    tar -xf /tmp/netboot.tar.gz .

3. 尝试网络启动

    在bios中启用并默认使用网络启动，开机，如果看到debian的安装界面表示pxe启动配置成功。

### 加入preseed 实现自动化安装

1.  下载debian 官方的preseed [配置实例](https://www.debian.org/releases/squeeze/example-preseed.txt) 里面会有详细的配置说明

    修改其中的配置，符合自己的需求。这个文件的作用实际上是帮助你回答debian的安装程序一些的问题，从而达到自动化安装的效果。

2.  编辑/srv/tftp/pxelinux.cfg/default

        # D-I config version 2.0
        include debian-installer/amd64/boot-screens/menu.cfg
        default debian-installer/amd64/boot-screens/vesamenu.c32
        prompt 0
        timeout 1               #默认等待时间，如果不设置网络安装将停留在引导界面

        default DEBIAN_AUTO_INSTALL           #默认的启动选项
        label DEBIAN_AUTO_INSTALL
            kernel debian-installer/amd64/linux       #引导的内核
            append  interface=auto netcfg/dhcp_timeout=120 auto=true priority=critical \
            url=http://192.168.100.1/squeeze.seed vga=normal \
            initrd=debian-installer/amd64/initrd.gz -- quiet #内核参数


    auto=true priority=critical url=http://192.168.100.1/squeeze.seed 就是关于自动化安装的配置。priority=critical 可以一定程度上避免出错而安装终止。
    url指定配置preseed文件路径，可以为自己搭建http服务器。

### example config

    d-i debian-installer/locale string en_US
    #locale

    d-i console-keymaps-at/keymap select us
    #键盘

    d-i keyboard-configuration/xkb-keymap select us
    d-i netcfg/confirm_static boolean true
    d-i netcfg/get_hostname string unassigned-hostname
    d-i netcfg/get_domain string unassigned-domain
    d-i mirror/country string manual
    d-i mirror/http/hostname string mirrors.aliyun.com
    #安装使用的源

    d-i mirror/http/directory string /debian
    d-i mirror/http/proxy string
    d-i mirror/suite string oldstable
    d-i passwd/root-login boolean false
    #用户名，密码

    d-i passwd/user-fullname string Jumei Sysop
    d-i passwd/username string sysop
    d-i passwd/user-password password p@ssW0rd
    d-i passwd/user-password-again password p@ssW0rd
    d-i clock-setup/utc boolean true
    #时区

    d-i time/zone string Asia/Chongqing
    d-i clock-setup/ntp boolean false
    #是否使用ntp

    d-i partman-lvm/device_remove_lvm boolean true
    #删除存在的lvm

    d-i partman-md/device_remove_md boolean true
    #删除磁盘阵列

    d-i partman-auto/method string regular
    #分区类型

    d-i partman-auto/choose_recipe select atomic
    #自动分区

    d-i partman-partitioning/confirm_write_new_label boolean true
    #确认写入

    d-i partman/choose_partition select finish
    #分区完成

    d-i partman/confirm boolean true
    d-i partman/confirm_nooverwrite boolean true
    d-i base-installer/install-recommends boolean false
    #关闭安装建议

    d-i base-installer/kernel/linux/initramfs-generators string initramfs-tools
    #引导文件生成

    d-i apt-setup/non-free boolean false
    #是否使用non-free的镜像

    d-i apt-setup/contrib boolean false
    #是否使用contrib镜像

    d-i apt-setup/use_mirror boolean false
    #是否使用本地源

    d-i apt-setup/services-select multiselect main
    #软件源

    d-i debian-installer/allow_unauthenticated boolean true
    #是否允许未经验证的软件包

    tasksel tasksel/first multiselect standard
    #安装standard

    d-i pkgsel/upgrade select full-upgrade
    #全部升级

    popularity-contest popularity-contest/participate boolean false
    d-i grub-installer/only_debian boolean true
    #grub安装选项

    d-i finish-install/reboot_in_progress note
    #完成重启

### 搭建本地debian镜像源

可以使用debmirror同步互联网的debian镜像到本地, 方便快速批量部署.

1.  nginx 配置文件

    ```
    server {
        listen 80;
        server_name 192.168.100.1;
        location / {
            root /var/www;
            autoindex on;
            index index.html index.htm;
        }
    }
    ```
    将上面的squeeze.seed 放到`/var/www`下面.

2.  使用debmirror同步软件源镜像

    #安装debmirror
    apt-get install debmirror

    #同步软件包main,main/debian-installer 大约有32G,需要很长时间

        debmirror -v -d squeeze -s main,main/debian-installer -e http -h mirrors.ustc.edu.cn --ignore-release-gpg \
                  --ignore-missing-release --nosource --diff=none /var/www/debian -a amd64

        #目录结构为
        .
        ├── debian
        │   ├── dists
        │   │   ├── oldstable -> squeeze
        │   │   └── squeeze
        │   │       ├── main
        │   │       │   ├── binary-amd64
        │   │       │   └── debian-installer
        │   │       ├── Release
        │   │       └── Release.gpg
        │   ├── pool
        │   │   └── main

        .....

        └── squeeze.seed

### production preseed config

    d-i debian-installer/locale string en_US
    d-i console-keymaps-at/keymap select us
    d-i keyboard-configuration/xkb-keymap select us
    d-i netcfg/confirm_static boolean true
    d-i netcfg/get_hostname string unassigned-hostname
    d-i netcfg/get_domain string unassigned-domain
    d-i mirror/country string manual
    d-i mirror/http/hostname string mirrors.aliyun.com
    d-i mirror/http/directory string /debian
    d-i mirror/http/proxy string
    d-i mirror/suite string oldstable
    d-i passwd/root-login boolean false
    d-i passwd/user-fullname string Jumei Sysop
    d-i passwd/username string sysop
    d-i passwd/user-password password p@ssW0rd
    d-i passwd/user-password-again password p@ssW0rd
    d-i clock-setup/utc boolean true
    d-i time/zone string Asia/Chongqing
    d-i clock-setup/ntp boolean false
    d-i partman-lvm/device_remove_lvm boolean true
    d-i partman-md/device_remove_md boolean true
    d-i partman-auto/method string regular

    d-i partman-auto/expert_recipe string                       \
            boot-root ::                                        \
                4000  16000 200% linux-swap                     \
                    method{ swap } format{ }                    \
                .                                               \
                51200 20480 102400 ext4                         \
                        method{ format } format{ }              \
                        use_filesystem{ } filesystem{ ext4 }    \
                        mountpoint{ / }                         \
                .                                               \
                128 937000 1000000 ext4                         \
                    method{ format } format{ }                  \
                    use_filesystem{ } filesystem{ ext4 }        \
                    mountpoint{ /home }                         \
                .                                               \
    #分区表，最小值 ，优先级，最大值

    d-i partman-partitioning/confirm_write_new_label boolean true
    d-i partman/choose_partition select finish
    d-i partman/confirm boolean true
    d-i partman/confirm_nooverwrite boolean true
    d-i base-installer/install-recommends boolean false
    d-i base-installer/kernel/linux/initramfs-generators string initramfs-tools
    d-i apt-setup/non-free boolean false
    d-i apt-setup/contrib boolean false
    d-i apt-setup/use_mirror boolean false
    d-i apt-setup/services-select multiselect main

    d-i apt-setup/local0/repository string \
        http://debian.saltstack.com/debian squeeze-saltstack main
    d-i apt-setup/local0/comment string saltstack
    d-i apt-setup/local0/key string \
        http://debian.saltstack.com/debian-salt-team-joehealy.gpg.key
    #源配置，地址，注释，key

    d-i apt-setup/local1/repository string \
        deb http://mirrors.ustc.edu.cn/debian-backports/ squeeze-backports main
    d-i apt-setup/local1/comment string debian backports
    #可以添加多个额外的源的配置

    d-i debian-installer/allow_unauthenticated boolean true

    tasksel tasksel/first multiselect standard
    d-i pkgsel/include string openssh-server less htop vim-nox lsb-release zip unzip curl
    #需要另外安装的软件包
    d-i pkgsel/upgrade select none

    popularity-contest popularity-contest/participate boolean false

    d-i grub-installer/only_debian boolean true
    d-i finish-install/reboot_in_progress note
    d-i debian-installer/exit/poweroff boolean true
    #安装完成之后关机,

    d-i preseed/late_command string chroot /target sh -c "/usr/bin/curl -L http://192.168.20.130/postinstall | /bin/sh"
    #安装完成之后执行的脚本，需要chenge root 到安装目录

### postinstall script

  调整网络的配置可以在这步完成, 将dhcp分配到的IP地址静态写入到网卡配置文件.

  ```bash
  #!/bin/sh
  address=`ip addr show eth0 | sed -n  's#\s*inet\b\s*\([0-9\.]*\).*$#\1#p'`
  cat <<EOF  > /etc/network/interfaces

  auto lo
  iface lo inet loopback

  auto eth0
  allow-hotplug eth0
  iface eth0 inet static
  address ${address}
  netmask 255.255.255.0
  gateway 192.168.20.1
  dns-nameservers 192.168.21.2
  EOF
  ```
