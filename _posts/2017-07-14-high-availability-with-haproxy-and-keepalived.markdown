---
layout:     post
title:      "Haproxy + Keepalived高可用配置"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Server
    - Haproxy
    - Keepalived
---

## 0 一篇一句

**完成比完美更重要**

## 1 概述

HAProxy is a free, very fast and reliable solution offering high availability, load balancing, and proxying for TCP and HTTP-based applications.

Keepalived is a routing software written in C. The main goal of this project is to provide simple and robust facilities for loadbalancing and high-availability to Linux system and Linux based infrastructures.

在Ucloud上面，使用Haproxy + Keepalived进行推荐服务器的高可用配置。

## 2 基础环境

#### 2.1 Hosts文件

- 10.19.103.85 blender1
- 10.19.161.195 blender2

#### 2.2 各种地址

- 推荐服务器：blender1:7085，blender2:7085
- HAProxy：blender1:7785，blender2:7785
- Keepalived的VIP：10.19.74.228

#### 2.3 网络拓扑结构图

此处应有图

## 3 HAProxy配置

#### 3.1 下载和编译

1. 下载haproxy-1.7.1.tar.gz
2. 编译安装

```bash
tar zxvf haproxy-1.7.1.tar.gz
ln -s haproxy-1.7.1 haproxy
cd haproxy
# linux2628及以上版本都使用TARGET=linux2628，centos7的内核版本为3.10.0
make TARGET=linux2628 PREFIX=/data/devtools/haproxy
make install PREFIX=/data/devtools/haproxy
```

#### 3.2 配置

```bash
# 用来存pid信息
mkdir $haproxy_dir/var
# 用来存配置文件
mkdir $haproxy_dir/conf
vim $haproxy_dir/conf/haproxy.cfg
```

haproxy.cfg的配置详情如下（haproxy的各个实例的配置都一样）：

```
global
    log 127.0.0.1 local3
    maxconn 8192
    daemon
    nbproc 1
    pidfile /data/devtools/haproxy/var/haproxy.pid


defaults
    mode http
    log global
    option httplog
    option httpclose
    option dontlognull
    option forwardfor
    option redispatch
    retries 2
    balance url_param userid
    stats uri /haproxy-stats
    timeout connect 5000
    timeout client 10000
    timeout server 10000


listen http-in
    hash-type consistent
    bind *:7785
    server web1 blender1:7085 cookie 1 maxconn 4096 check port 7085 inter 1000 rise 3 fall 3
    server web2 blender2:7085 cookie 2 maxconn 4096 check port 7085 inter 1000 rise 3 fall 3
```


#### 3.3 日志

haproxy默认情况下不记录日志，需要修改haproxy.conf配置，并同时修改rsyslog的配置来实现日志记录。
最终，haproxy的日志记录在 /data/devtools/haproxy/logs/haproxy.log 下（[参考链接](http://www.ttlsa.com/linux/haproxy-log-configuration-syslog/)）

###### 3.3.1 编辑haproxy.cfg

编辑 $haproxy_dir/conf/haproxy.cfg（见“二”中，haproxy配置的line2和line11）

###### 3.3.2 编辑系统日志配置

```bash
vim /etc/rsyslog.conf

# Provides UDP syslog reception（把这里下面两行的注释去掉）
$ModLoad imudp
$UDPServerRun 514

# Save haproxy log to haproxy.log（添加这部分）
local3.*                                                /data/devtools/haproxy/logs/haproxy.log
```

###### 3.3.3 配置rsyslog的主配置文件，开启远程日志

```bash

vim /etc/sysconfig/rsyslog
vim /etc/default/rsyslog（ubuntu）

SYSLOGD_OPTIONS="-c 2 -r -m 0"
 #-c 2 使用兼容模式，默认是 -c 5
 #-r 开启远程日志
 #-m 0 标记时间戳。单位是分钟，为0时，表示禁用该功能
```
###### 3.3.4 重启rsyslog服务

```bash
sudo service rsyslog restart
```

#### 3.4 启动和停止Haproxy

```bash
# 启动
$haproxy_dir/sbin/haproxy -f $haproxy_dir/conf/haproxy.cfg

# 停止
killall haproxy
```

## 4 Keepalived配置

#### 4.1 下载和编译

1. 下载keepalived-1.3.0.tar.gz
2. 使用sudo安装，因为编译的时候需要往/usr/sbin写东西
```bash
sudo tar zxvf keepalived-1.3.0.tar.gz
sudo ln -s keepalived-1.3.0 keepalived
sudo cd keepalived
sudo ./configure
sudo make
sudo make install
sudo cp bin/keepalived /usr/sbin/keepalived
sudo mkdir -p /etc/keepalived
sudo vim /etc/keepalived/keepalived.conf
```

3. Ucloud的虚拟IP申请，按照图中的步骤，依次选择并申请。得到的VIP地址，填到keepalived的配置文件中
![UcloudVIP](/img/in-post/post-high-availability-with-haproxy-and-keepalived/UcloudVIP.png)

#### 4.2 配置

Keepalived分为主和备，主和备的**相同和不同**如下：
- virtual_router_id相同
- unicast_peer必须为ip地址（不能为域名），blender1的配置中填blender2的ip，blender2的配置中填blender1的ip
- blender1的state为MASTER，blender2的state为BACKUP
- blender1的priority为99，blender2的priority为98
- virtual_ipaddress相同，都为10.19.74.228

###### 4.2.1 blender1的配置（主）

```
! Configuration File for keepalived
global_defs {
   notification_email {
     xiaochutian@foxmail.com
   }
   notification_email_from andrew.xiao@leappmusic.com
   smtp_server smtp.ym.163.com
   smtp_connect_timeout 30
   router_id HAProxy
}

vrrp_script check_haproxy {
    script "/data/devtools/keepalived/haproxy_check/haproxy_check.sh"
    weight 5
    interval 1
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 99
    advert_int 1
    unicast_peer {
       10.19.161.195
    }
    authentication {
       auth_type PASS
       auth_pass 1234
    }
    virtual_ipaddress {
      10.19.74.228 dev eth0
    }
    track_script {
        check_haproxy
    }
}
```

###### 4.2.2 blender2的配置(备）

```
! Configuration File for keepalived
global_defs {
   notification_email {
     xiaochutian@foxmail.com
   }
   notification_email_from andrew.xiao@leappmusic.com
   smtp_server smtp.ym.163.com
   smtp_connect_timeout 30
   router_id HAProxy
}

vrrp_script check_haproxy {
    script "/data/devtools/keepalived/haproxy_check/haproxy_check.sh"
    weight 5
    interval 1
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 99
    advert_int 1
    unicast_peer {
       10.19.161.195
    }
    authentication {
       auth_type PASS
       auth_pass 1234
    }
    virtual_ipaddress {
      10.19.74.228 dev eth0
    }
    track_script {
        check_haproxy
    }
}
```

#### 4.3 日志

- 默认输出到文件  /var/log/messages
- 可以通过类似haproxy的方式，指定日志输出路径 [参考链接](http://www.cnblogs.com/liangdalong/p/6106077.html)

#### 4.4 启动和停止Keepalived

```
sudo service keepalived start/stop
```

## 5 向集群中添加机器

1. 修改haproxy配置文件$haproxy_dir/conf/haproxy.cfg，在listen部分添加机器，配置类似其他机器
2. 重启haproxy

> 注：由于有多个haproxy，可以依次修改并重启，服务不会中断

## 6 其他

Ucloud配置Keepalived文档地址：https://docs.ucloud.cn/compute/uhost/public/keepalived.html

## 5 参考

[完成比完美更重要](http://www.xinli001.com/info/100350992)    
[Haproxy官网](http://www.haproxy.org/)    
[Keepalived官网](http://www.keepalived.org/)    
[Haproxy日志配置](http://www.ttlsa.com/linux/haproxy-log-configuration-syslog/)    
[Keepalived日志配置](http://www.cnblogs.com/liangdalong/p/6106077.html)    
[Ucloud配置Keepalived文档](https://docs.ucloud.cn/compute/uhost/public/keepalived.html)    
