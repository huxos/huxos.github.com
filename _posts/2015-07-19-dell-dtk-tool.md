---
layout: post
title: 使用Dell DTK批量设置服务器BIOS
description: 使用Dell DTK批量设置服务器BIOS
categories:
  - Hardware
---

#### 下载DELL DTK

  >DTK全称为`Dell OpenManage Deployment Toolkit`可以用来更改dell服务器的bios等配置

  [下载地址](http://www.dell.com/support/home/us/en/19/Drivers/DriversDetails?driverId=8PRMX)
  下载得到一个集成了DTK的centos的系统,可通过pxe启动,用于在未安装操作系统的服务器上面更改BIOS

  DTK也有各发行版的软件包,可以在已有操作系统的服务器上面安装.DEBIAN下载地址为: [http://linux.dell.com/repo/community/debian/pool/wheezy/](http://linux.dell.com/repo/community/debian/pool/wheezy/)


#### 设置pxe启动dtk工具

  解压之后得到SA.1 SA.2 放入到tftp目录的dtk下,编辑pxe启动菜单

    label dtk_bios_set
        kernel dtk/SA.1
        append initrd=dtk/SA.2 ramdisk_size=65536

  启动之后得到shell,可以执行管理命令.

#### 引导并且自动执行命令

  编辑pxe引导之后自动执行的脚本, 其中`/opt/dell/toolkit/bin/syscfg` 就是DTK中用于设置BIOS的工具.

  ```
  #!/bin/sh -e

  /opt/dell/toolkit/bin/syscfg --virtualization=disable
  #/opt/dell/toolkit/bin/syscfg --cpucle=enable
  /opt/dell/toolkit/bin/syscfg --logicproc=disable
  #/opt/dell/toolkit/bin/syscfg --turbomode=enable
  #/opt/dell/toolkit/bin/syscfg --cstates=enable
  /opt/dell/toolkit/bin/syscfg --serialcomm=on
  #/opt/dell/toolkit/bin/syscfg --conboot=enable
  #/opt/dell/toolkit/bin/syscfg --memintleave=disable
  /opt/dell/toolkit/bin/syscfg power --profile=maxperformance --setuppwdoverride #关闭节能模式，性能最大化

  # 可以在此处使用ipmitool等工具来更改IPMI的配置 ...

  ## 脚本最后重启服务器
  reboot
  ```
  保存到pxe启动的dtk目录下面,文件名称为bios.sh

  编辑上面提到的pxe引导配置，传递参数脚本路径、脚本名字、tftp服务器IP。(也支持从http获取脚本)

    label dtk_bios_set
          kernel dtk/SA.1
          append initrd=dtk/SA.2 ramdisk_size=65536 share_type=tftp share_location=dtk share_script=bios.sh tftp_ip=<tftp服务器IP地址>

  这样下次引导到DTK之后就会自动修改bios并且重启
