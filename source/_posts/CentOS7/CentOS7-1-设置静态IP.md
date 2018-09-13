---
title: CentOS7-1-设置静态IP
date: 2018-09-13 14:08:21
categories: Linux
tags: 
  - CentOS7
toc: true
list_number: false
---

# 1、找到配置文件

```shell
[root@localhost network-scripts]# cd /etc/sysconfig/network-scripts/
[root@localhost network-scripts]# ls
ifcfg-ens33  ifdown-bnep  ifdown-ipv6  ifdown-ppp     ifdown-Team      ifup          ifup-eth   ifup-isdn   ifup-post    ifup-sit       ifup-tunnel       network-functions
ifcfg-lo     ifdown-eth   ifdown-isdn  ifdown-routes  ifdown-TeamPort  ifup-aliases  ifup-ippp  ifup-plip   ifup-ppp     ifup-Team      ifup-wireless     network-functions-ipv6
ifdown       ifdown-ippp  ifdown-post  ifdown-sit     ifdown-tunnel    ifup-bnep     ifup-ipv6  ifup-plusb  ifup-routes  ifup-TeamPort  init.ipv6-global
[root@localhost network-scripts]# 
```

<!--more-->

# 2、编辑配置文件

第一步查找的配置文件为ifcfg-ens33

```shell
[root@localhost network-scripts]# vim ifcfg-ens33 

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=006320c1-d3ce-4a8e-9e5c-25740047281a
DEVICE=ens33
ONBOOT=no
```

修改后配置文件

```shell
[root@localhost network-scripts]# vim ifcfg-ens33 

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
#BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=006320c1-d3ce-4a8e-9e5c-25740047281a
DEVICE=ens33
#ONBOOT=no

# static assignment
NM_CONTROLLED=no #表示该接口将通过该配置文件进行设置，而不是通过网络管理器进行管理
ONBOOT=yes #开机启动
BOOTPROTO=static #静态IP
IPADDR=10.141.122.52 #本机地址
NETMASK=255.255.255.0 #子网掩码
GATEWAY=10.141.122.1 #默认网关
```

# 3、修改/etc/sysconfig/network

```shell
[root@localhost network-scripts]# vim /etc/sysconfig/network

# Created by anaconda
NETWORKING=yes
GATWAY=10.141.122.1
DNS1=222.172.200.68
```

# 4、重启服务

```java
[root@localhost network-scripts]# service network restart
Restarting network (via systemctl):                        [  OK  ]
[root@localhost network-scripts]# 
```



