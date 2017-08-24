---
layout: post
title: 使用irq绑定＋rps 提升机器网络性能
description: 使用irq绑定＋rps 提升机器网络性能
categories:
  - Linux
---

#### 网卡软中断绑定irq设置

使用lspci -vvv 查看网卡时候支持多个收发队列，关键字为`MSI-X`，并且有Enable+。

    ...............
    Capabilities: [58] MSI: Enable- Count=1/16 Maskable- 64bit+
        Address: 0000000000000000  Data: 0000
    Capabilities: [a0] MSI-X: Enable+ Count=9 Masked-
        Vector table: BAR=0 offset=0000c000
        PBA: BAR=0 offset=0000e000
    ..................

通过 `cat /proc/interrupts |grep eth0` 查看相应的网络中断号，对每一个中断号绑定到指定的CPU。

#### rps 设置

解决网卡队列数目和cpu核心数不一致的情况，通过配置`/sys/class/net/eth0/queues/rx-*/rps_cpus`对CPU进行绑定.

#### 相关设置的脚本

  ```python
  #!/usr/bin/env python
  # encoding: utf-8

  def smp_affinity():

      #每8位插入一个","
      def strConv(s):
          import re
          s = str(s)
          while True:
              (s, count) = re.subn(r"(\w)(\w{8})((:?,\w{8})*)$", r"\1,\2\3", s)
              if count == 0:
                  break
          return s

      import re
      regex = re.compile(r'(eth[0-9]-(tx|rx|txrx|fp|[-]?[0-9]))', re.IGNORECASE)
      num_cpus = int(
          len([x for x in open('/proc/cpuinfo') if x.startswith('processor')]))
      net_dev_list = [x.split()[-1]
                      for x in open('/proc/interrupts') if 'eth' in x]
      net_dev_irq = [x.split()[0].strip(':')
                     for x in open('/proc/interrupts') if 'eth' in x]

      if re.search(regex, ','.join(net_dev_list)):
          smp_affinity = ['/proc/irq/%s/smp_affinity' % x for x in net_dev_irq]
          bit_cpus = [strConv(str(hex(1 << x)).replace('0x', '')) for x in range(
              num_cpus)][::-1] * (len(net_dev_irq) / num_cpus + 1)
      else:
          smp_affinity = ['/proc/irq/%s/smp_affinity' % x for x in net_dev_irq]
          bit_cpus = [
              strConv(hex(int('1' * num_cpus, 2)).replace('0x', ''))] * len(smp_affinity)

      smp_affinity_list = zip(bit_cpus, smp_affinity)

      active_nics = list(set([x.split('-')[0] for x in net_dev_list]))
      if(len(active_nics)):
          from glob import glob
          rps_flow_cnt = 0
          for nic in active_nics:
              rps = glob('/sys/class/net/%s/queues/rx-*/rps_cpus' % nic)
              if len(rps):
                  if len(rps) != num_cpus:
                      bit_cpus = [
                          strConv(hex(int('1' * num_cpus, 2)).replace('0x', ''))] * len(rps)
                      smp_affinity_list.extend(
                          zip([4096] * len(rps), [x.replace('rps_cpus', 'rps_flow_cnt') for x in rps]))
                      rps_flow_cnt = rps_flow_cnt + 4096 * len(rps)
                  else:
                      bit_cpus = [0] * len(rps)
                  smp_affinity_list.extend(zip(bit_cpus, rps))
              xps = glob('/sys/class/net/%s/queues/tx-*/xps_cpus' % nic)
              if len(xps) >= num_cpus:
                  bit_cpus = [
                      strConv(str(hex(1 << x)).replace('0x', '')) for x in range(num_cpus)][::-1]
                  smp_affinity_list.extend(zip(bit_cpus, rps))

              print 'ethtool -C %s  rx-usecs 333' % nic

          if rps_flow_cnt != 0:
              smp_affinity_list.append(
                  (rps_flow_cnt, '/proc/sys/net/core/rps_sock_flow_entries'))

      for smp in smp_affinity_list:
          print 'echo %s > %s' % (smp[0], smp[1])

  if __name__ == '__main__':
      smp_affinity()
  ```

#### 参考连接

1.  [https://github.com/fastos/fastsocket/blob/master/scripts/nic.sh](https://github.com/fastos/fastsocket/blob/master/scripts/nic.sh)
2.  [Red Hat Customer Portal](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Performance_Tuning_Guide/sect-Red_Hat_Enterprise_Linux-Performance_Tuning_Guide-Networking-Configuration_tools.html#sect-Red_Hat_Enterprise_Linux-Performance_Tuning_Guide-Configuration_tools-Configuring_Receive_Packet_Steering_RPS)
