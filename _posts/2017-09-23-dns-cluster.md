---
layout: post
title: 基于mysql存储的DNS（bind）部署方案和步骤
description: kubernetes集群搭建
categories:
  - Bind
---

#### bind部署的架构

![bind架构](/images/bind-cluster.png)

其中master节点采用dlz的方式与使用mysql作为数据后端，slave节点通过zone transfer与master节点同步。  
Master不作为服务节点，Slave节点使用LVS作为负载均衡提供服务，这样可以很方便的通过对数据库的CURD操作进行dns记录的变更，同时也可以保留bind原生的性能。  
由于master节点的数据保存在mysql中，当数据发生变化之后无法及时感知，slave节点的数据可能会出现与mster不一致的情况。这时我们需要对salve发送notify指令来主动同步master的数据。
可以使用[https://github.com/huxos/dns-notify](https://github.com/huxos/dns-notify) 来发送notify指令。

#### 部署步骤

PS：部署操作系统为：`Centos 7`, Bind版本为`9.11.0-P5`。

1．编译bind支持mysql

  安装编译依赖

```shell
yum install openssl-devel mariadb-devel gcc
ln -sf /usr/lib64/mysql/libmysqlclient.so.18 /lib64/libmysqlclient.so
```

  编译bind

```shell
cd /usr/src
wget http://ftp.isc.org/isc/bind9/9.11.0-P5/bind-9.11.0-P5.tar.gz
tar -zxf bind-9.11.0-P5.tar.gz
cd bind-9.11.0-P5
./configure --with-dlz-mysql --enable-largefile --enable-threads=no --prefix=/usr/local/bind
make
make install
```

  创建bind运行的用户

```Shell
groupadd -r -g 25 named
useradd -r -u 25 -s /bin/nologin -d /usr/local/named -g named named
mkdir -p /var/cache/bind/data
chown named:named /var/cache/bind
```

2．创建数据库的表结构

连接上mysqi创建数据库表

```Sql
CREATE TABLE `dns_records` (
    `zone` varchar(255) DEFAULT NULL,
    `host` varchar(255) DEFAULT NULL,
    `type` varchar(255) DEFAULT NULL,
    `data` varchar(255) NOT NULL DEFAULT '',
    `ttl` int(11) DEFAULT NULL,
    `mx_priority` varchar(255) DEFAULT NULL,
    `refresh` int(11) DEFAULT NULL,
    `retry` int(11) DEFAULT NULL,
    `expire` int(11) DEFAULT NULL,
    `minimum` int(11) DEFAULT NULL,
    `serial` bigint(20) DEFAULT NULL,
    `resp_person` varchar(255) DEFAULT NULL,
    `primary_ns` varchar(255) DEFAULT NULL,
    KEY `zone_index` (`zone`),
    KEY `host_index` (`host`),
    KEY `type_index` (`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```sql
CREATE TABLE `xfr_table` (
    `zone` varchar(255) NOT NULL,
    `client` varchar(255) NOT NULL,
    KEY `zone_client_index` (`zone`(30)),
    KEY `client_index` (`client`(20))
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
其中`dns_records`表用于储存dns记录值，`xfr_table`是用来做DNS复制使用的。

插入dns记录

```Sql
INSERT INTO `dns_records` VALUES
('bindtest.example.com','@','SOA','bindtest.example.com.',300,NULL,120,900,604800,600,2018120210,'admin.bindtest.example.com.','ns1.bindtest.example.com.'),
('bindtest.example.com','@','NS','ns1.bindtest.example.com.',300,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL),
('bindtest.example.com','@','NS','ns2.bindtest.example.com.',300,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL),
('bindtest.example.com','ns1','A','192.168.23.23',300,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL),
('bindtest.example.com','ns2','A','192.168.23.24',300,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL),
('bindtest.example.com','@','A','192.168.20.81',300,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL);
```

需要注意的是NS记录的值为辅助dns的IP。

```Sql
INSERT INTO `xfr_table` VALUES
('bindtest.example.com','192.168.23.23'),
('bindtest.example.com','192.168.23.24');
```

需要将辅助DNS的IP插入到这张表中。

3．配置bind

 我们编译的bind目录在`/usr/local/bind/`，配置文件位于`/usr/local/bind/etc` 。由于我们编译的bind并没有一些基础的配置，首先需要把yum源里面的bind的一些基础的配置和默认的zone配置拷贝到我们定义的目录中。

```Shell
cp /usr/share/doc/bind-9.9.4/sample/var/named/named.* /var/cache/named/
cp /usr/share/doc/bind-9.9.4/sample/etc/named.rfc1912.zones /usr/local/bind/etc
```

编辑`usr/local/bind/etc/named.conf`：

```
key "rndc-key" {
	algorithm hmac-md5;
	secret "0FpSonU53LYhtZGbpMhR8w==";
};

controls {
	inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { "rndc-key"; };
};

statistics-channels {
  	inet 127.0.0.1 port 8053 allow { 127.0.0.1; };
};

include "/usr/local/bind/etc/named.conf.options";
include "/usr/local/bind/etc/named.conf.logging";
include "/usr/local/bind/etc/bind.keys";

zone "." IN {
	type hint;
	file "named.ca";
};

include "/usr/local/bind/etc/named.rfc1912.zones";
include "/usr/local/bind/etc/named.dlz.zones";
```

编辑`/usr/local/bind/etc/named.conf.options`：其中需要注意的是`also-notify`和`allow-transfer` 需要设置成辅助dns的IP。

```
options {
  directory "/var/cache/bind";
  zone-statistics yes;
  statistics-file "/var/cache/bind/data/named-stats.out";
  allow-query {
	any;
  };
  allow-transfer {
	192.168.23.23;
	192.168.23.24;
  };
  notify yes;
  also-notify {
	192.168.23.23;
	192.168.23.24;
  };
  recursion no;
  pid-file "/run/named/named.pid";
  session-keyfile "/run/named/session.key";
  #forwarders {
  #      192.168.1.1;
  #      192.168.1.2;
  #      192.168.1.3;
  #};
  #forward only;
};
```

编辑`/usr/local/bind/etc/named.conf.logging`：

```
logging {
	channel default-log {
		file "data/default.log" versions 20 size 50m;
		severity info;
		print-severity yes;
		print-time yes;
		print-category yes;
	};
	channel query-log {
		file "data/query.log" versions 20 size 50m;
		severity info;
		print-severity yes;
		print-time yes;
		print-category yes;
	};
	channel security-log {
		file "data/security.log" versions 20 size 50m;
		severity info;
		print-severity yes;
		print-time yes;
		print-category yes;
	};
	channel other-log {
		file "data/other.log" versions 20 size 50m;
		severity info;
		print-severity yes;
		print-time yes;
		print-category yes;
	};
	channel database-log {
		file "data/database.log" versions 20 size 50m;
		severity info;
		print-severity yes;
		print-time yes;
		print-category yes;
	};
	category default {default-log;};
	category queries { query-log;};
	category security { security-log;};
	category database { database-log;};
	category lame-servers { null; };
	category client { other-log;};
	category config { other-log;};
	category general { other-log;};
};
```

编辑`/usr/local/bind/etc/named.dlz.zones`:

```
dlz "Mysql zone" {
   database "mysql
   {host=192.168.20.67 port=3306 dbname=bind user=db_bind_app pass=****** ssl=false}
   {select zone from dns_records where zone = '$zone$'}
   {select ttl, type, mx_priority, case when lower(type)='txt' then concat('\"', data, '\"')
        when lower(type) = 'soa' then concat_ws(' ', data, resp_person, serial, refresh, retry, expire, minimum)
        else data end from dns_records where zone = '$zone$' and host = '$record$'}
   {}
   {select ttl, type, host, mx_priority, case when lower(type)='txt' then
        concat('\"', data, '\"') else data end, resp_person, serial, refresh, retry, expire,
        minimum from dns_records where zone = '$zone$'}
   {select zone from xfr_table where zone = '$zone$' and client = '$client$'}";
};
```

在辅助dns上`/usr/local/bind/etc/named.dlz.zones`文件如下，我们将其配置成从主dns同步，需要将配置允许从主dns进行 notify和update。

```
zone "bindtest.example.com" IN {
    type slave;
    file "zone-from-mysql.db";
    masters { 192.168.23.22; };
    allow-update { 192.168.23.22; };
    allow-notify {
	192.168.23.22;
	127.0.0.1;
    };
};
```

编辑启动bind的systemd unit `/lib/systemd/system/named.service`

```shell
[Unit]
Description=Bind DNS Server
After=network.target

[Service]
Type=forking
PIDFile=/run/named/named.pid
ExecStart=/usr/local/bind/sbin/named -c /usr/local/bind/etc/named.conf -u named

[Install]
WantedBy=multi-user.target
```

启动bind

```Shell
systemctl start named
systemctl enable named
```
