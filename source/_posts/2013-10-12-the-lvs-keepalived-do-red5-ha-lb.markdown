---
layout: post
title: "LVS+Keepalived做Red5高可用和负载均衡"
date: 2013-10-12 14:33
comments: true
categories: 
- 流媒体

---

## 前言
此篇文章纯实践篇，只有部分关键配置会作解决，理论知识偶会专门写一篇来介绍。


1. 准备环境

```
192.168.30.136     lvs-dr_01	# LVS Director Master
192.168.30.167     lvs-dr_02	# LVS Director Backup
192.168.30.172     lvs-vip		# VIP  虚拟IP
192.168.30.171     realserver-01	# 后端Red5 服务器
192.168.30.150     realserver-02	# 后端Red5 服务器
```

2. 安装依赖包

* 检查是否支持IPVS

```
modprobe -l | grep ipvs 
```
如果支持的话，会就输出以下类似信息：

```
kernel/net/netfilter/ipvs/ip_vs.ko
kernel/net/netfilter/ipvs/ip_vs_rr.ko
kernel/net/netfilter/ipvs/ip_vs_wrr.ko
kernel/net/netfilter/ipvs/ip_vs_lc.ko
kernel/net/netfilter/ipvs/ip_vs_wlc.ko
kernel/net/netfilter/ipvs/ip_vs_lblc.ko
kernel/net/netfilter/ipvs/ip_vs_lblcr.ko
kernel/net/netfilter/ipvs/ip_vs_dh.ko
kernel/net/netfilter/ipvs/ip_vs_sh.ko
kernel/net/netfilter/ipvs/ip_vs_sed.ko
kernel/net/netfilter/ipvs/ip_vs_nq.ko
kernel/net/netfilter/ipvs/ip_vs_ftp.ko
```

3. 安装ipvsadm
SSH到主节点 lvs-dr_01 和 lvs-dr_02

```
yum -y install ipvsadm
```

4. 安装 Keepalived
以下分别是安装依赖包、下载keepalived-1.2.8、以配置文件：

```
yum install -y gcc kernel-devel openssl-devel popt-devel  make # 安装依赖包
wget http://www.keepalived.org/software/keepalived-1.2.8.tar.gz
tar zxvf keepalived-1.2.8.tar.gz 
cd keepalived-1.2.8
./configure --with-kernel-dir=/usr/src/kernels/2.6.32-358.18.1.el6.x86_64/
make
make install DIR=/usr/local/
cp /usr/local/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
mkdir -p /etc/keepalived
cp /usr/local/sbin/keepalived /usr/sbin/
```

在 /usr/local/etc/keepalived/keepalived.conf 中加入以下内容：


```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc   # 全局联系人
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER		# keepalived MASTER，另外一台 lvs-dr_02 修改为：BACKUP
    interface eth0		# 使用哪个网络接口来通信
    virtual_router_id 52	# 默认为51，两个DR都修改成非51，这里修改成 52
    priority 100		# 优先级，数字越大表示 优先级越高，MASTER一定要比BACKUP高
    advert_int 1
    authentication {
        auth_type PASS		# 密码验证类型
        auth_pass 1111		# MASTER与BACKUP之间的验证密码，两端必须保持一致。
    }
    virtual_ipaddress {
        192.168.30.172   # 虚拟IP，即 VIP
    }
}
# 虚拟服务器  需要填写自己 IP 和 端口
virtual_server 192.168.30.172 1935 {
    delay_loop 6    		# 每隔 6 秒去检查 RealServer 的健康状态
    lb_algo rr				#  LVS 的 rr(轮循)算法，LVS共 10种算法，如果需要修改就在此处。
    lb_kind DR				# LVS DR模式(直接路由)，LVS共三种工作模式：VS/NAT、VS/TUN、VS/DR
    persistence_timeout 50		# 同一IP的连接在60秒内分配到同一台 RealServer
    protocol TCP			# 使用TCP协议来检查后台服务器(realserver)状态

    real_server 192.168.30.171 1935 {		# 后端red5 IP地址与端口
        weight 1						# 权重
        TCP_CHECK {
            connect_timeout 3			# 3秒内无响应超时
            nb_get_retry 3
            delay_before_retry 3
            connect_port 1935
        }
    }

    real_server 192.168.30.150 1935 {		# 后端red5 IP地址与端口
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 1935
        }
    }

}
```
同样，在两台 Director 都需要安装与配置，注意的是两台keepalived.conf的配置所有不同。

5. 设置Real Server
以下操作如果没有特别说明，将针对 **所有Real Server**
将以下内容添加到 /root/realserver.sh

```
#!/bin/bash

SNS_VIP=192.168.30.172
/etc/rc.d/init.d/functions
case "$1" in
start)
       /sbin/ifconfig lo:0 $SNS_VIP netmask 255.255.255.255 broadcast $SNS_VIP
       /sbin/route add -host $SNS_VIP dev lo:0
       echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
       sysctl -p >/dev/null 2>&1
       echo "RealServer Start OK"
       ;;
stop)
       /sbin/ifconfig lo:0 down
       /sbin/route del $SNS_VIP >/dev/null 2>&1
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
       echo "RealServer Stoped"
       ;;
*)
       echo "Usage: $0 {start|stop}"
       exit 1
esac
exit 0
```

添加执行权限
chmod +x /root/realserver.sh

启动及添加到开机自启动

```
/root/realserver.sh start
echo "/root/realserver.sh start" >> /etc/rc.local
```

6. 测试
首先依次启动 lvs-dr_01 和 lvs-dr_02 的keepalived服务：

```
/etc/init.d/keepalived restart
```

检查LVS路由和连接

```
ipvsadm -Ln

IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.30.172:1935 rr persistent 50
  -> 192.168.30.150:1935          Route   1      2          0         
  -> 192.168.30.171:1935          Route   1      0          0         
```
可以看到 realserver-01:1935 和 realserver-02:1935 已经加入到LVS。


通过RTMP客户端不断访问，可以看到2台 RealServer 都有活动的链接：

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.30.172:1935 rr persistent 50
  -> 192.168.30.150:1935          Route   1      1          0         
  -> 192.168.30.171:1935          Route   1      1          0         
```

7. 常见错误
ip address associated with VRID not present in received packet
这个错误主要原因是 在同一网段内virtual_router_id 值不能相同，如果相同会在messages中收到VRRP错误包，所以需要更改 virual_router_id，但如果只改一个，就等于是2个相对独立的集群，所以virual_router_id改成非51的相同值即可，例如都改成 52.
