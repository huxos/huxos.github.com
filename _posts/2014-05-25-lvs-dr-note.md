---
layout: post
title: LVS部署DR模式方法备忘
description: LVS部署DR模式方法
categories:
  - LVS
---
### 1. VS机器设置

1. 安装相应的软件

    `yum install keepalived ipvsadm`


2. 关闭网卡的`LRO`,`GRO`

    2.1  编辑`/etc/rc.local`加入, 并且执行操作下面两行命令

    ```bash
    ethtool -K eth2 lro off
    ethtool -K eth2 gro off
   ```

3. 修改vs模块hash表长度


   编辑`/etc/modprobe.d/ip_vs.conf` 加入:

    ```shell
    options ip_vs  conn_tab_bits=20
    ```

    重新加载ip_vs 模块

    ```shell
    modprobe -r ip_vs && modprobe ip_vs
    ```

    执行`ipvsadm -Ln`确认参数是否生效

    ```shell
    IP Virtual Server version 1.2.1 (size=1048576) ##此处为默认的1024则不生效
    Prot LocalAddress:Port Scheduler Flags
   ```

4. 配置keepalived


    编辑`/etc/keepalived/keepalived.conf` 使用`man keepalived.conf` 查看配置示例. 下面是配置模板:

    需要要注意的是: router_id 在一个vlan不能重复, realserver要与VS在同一个网段, 前后段端口一致, 根据业务设置不同的持久连接时间

    ```
    global_defs {
        notification_email {
            hujun@example.com
        }
        notification_email_from noreply-lvs@example.com
        smtp_server 127.0.0.1
        smtp_connect_timeout 60
        router_id lvs_main
    }

    vrrp_instance VI_0 {
        state MASTER
        interface  eth2
        virtual_router_id 233 ##router_id 同一个vlan不能重复
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass lvs@example
        }
        virtual_ipaddress {
            10.33.125.233
        }
    }

    virtual_server 10.33.125.233 6083 {
        delay_loop 6
        lb_algo rr
        lb_kind DR   ##dr模式
        protocol TCP
        persistence_timeout 0 ##持久连接时间
        real_server 10.33.124.35 6083 {
            TCP_CHECK {
                connect_timeout 10
            }
        }
        real_server 10.33.124.18 6083 {
            TCP_CHECK {
                connect_timeout 10
            }
        }
    }

    virtual_server 10.33.125.234 6083 {
        delay_loop 6
        lb_algo rr
        lb_kind DR
        protocol TCP
        persistence_timeout 0
        real_server 10.33.124.35 6083 { ##realserverIP
            TCP_CHECK {
                connect_timeout 10
            }
        }
        real_server 10.33.124.18 6083 {
            TCP_CHECK {
                connect_timeout 10
            }
        }
    }
    ```

### 2. RealServer配置

1. 添加内核配置

    编辑`/etc/sysctl.conf` 加入如下内容, 并执行`sysctl -e -p` 抑制非本接口的arp响应, 避免realserver 上的arp广播抢占VIP

    ```bash
    net.ipv4.conf.lo.arp_ignore=1
    net.ipv4.conf.lo.arp_announce=2
    net.ipv4.conf.all.arp_ignore=1
    net.ipv4.conf.all.arp_announce=2
    ```

2. 设置VIP

    编辑`/etc/sysconfig/network-scripts/ifcfg-lo:0` 加入如下配置, 其中`10.33.125.233` 为需要设置的VIP

    ```bash
    DEVICE=lo:0
    IPADDR=10.33.125.233
    NETMASK=255.255.255.255
    NETWORK=10.33.125.233
    BROADCAST=10.33.125.233
    ONBOOT=yes
    NAME=lo0
    ```

    也可以添加多个文件绑定多个VIP,比如`/etc/sysconfig/network-scripts/ifcfg-lo:1`

    然后执行`ifup lo:0`启动相应的网卡.

3. 启动服务

    服务建议监听0.0.0.0 , VIP上监听是为了做负载均衡, 实IP上监听是LVS存活检测的需要.
