---
layout: post
title: kubernetes集群搭建（1. docker）
description: kubernetes集群搭建
categories:
  - Kubernetes
  - Docker
---


1. 操作系统

	- 系统选择：Ubuntu 16.04.2 LTS, 内核版本4.10.0-27-generic

	- 安装操作系统： 尽量保持`/var/`是最大的分区，docker和kubernete 的数据默认保存在此分区，采用xfs文件系统。
    如果`/var/`分区较小需要改变docker启动参数指定数据所在目录，例如：`-g /home/docker`

	- 操作系统调优：做好操作系统基本的调优比如`内核参数调优`，`设置ulimit`，`网卡软中断绑定`等。


2. 安装docker

	参照[Docker官方](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/)提供的方法安装docker-ce最新版本。

	```bash
	sudo apt-get install \
	apt-transport-https \
	ca-certificates \
	curl \
	software-properties-common

	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

	#可以使用第三方的镜像提高安装速度
	sudo add-apt-repository \
	"deb [arch=amd64] http://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
	$(lsb_release -cs) \
	stable"

	sudo apt-get update
	sudo apt-get install docker-ce
	```
  由于内核版本较新可以更改dockerd的启动参数采用overlay2驱动,  
	使用命令`sudo systemdctl edit docker`编辑docker的启动参数，更改之后的内容为：

	```bash
	[Service]
	ExecStart=
	ExecStart=/usr/bin/dockerd -H fd:// -s overlay2
	```

至此，宿主机docker相关环境就准备完成了。
