---
layout: post
title: "OpenStack Swift安装与配置"
date: 2013-10-15 14:00
comments: true
categories: 
- OpenStack
- 云计算

---

## 准备环境

```
192.168.30.150	proxy server
192.168.30.151	storage server 
192.168.30.152	storage server
```

## 网络配置
** Proxy 代理节点网络(单网卡)  **

```
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.30.150
netmask 255.255.255.0
gateway 192.168.30.1
network 192.168.30.0
broadcast 192.168.30.255
dns-nameservers 218.201.4.3
```

** 存储节点网络(单网卡)  **
存储节点1：

```
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.30.151
netmask 255.255.255.0
gateway 192.168.30.1
network 192.168.30.0
broadcast 192.168.30.255
dns-nameservers 218.201.4.3
```

存储节点2：

```
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 192.168.30.152
netmask 255.255.255.0
gateway 192.168.30.1
network 192.168.30.0
broadcast 192.168.30.255
dns-nameservers 218.201.4.3
```

## 安装公共组件
以下操作在所有节点全部安装：
### 添加源

```
cat > /etc/apt/sources.list.d/grizzly.list << _END_
deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main
deb  http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/grizzly main
_END_
```

更新源

```
sudo apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 5EDB1B62EC4926EA
apt-get update
apt-get upgrade
apt-get install ubuntu-cloud-keyring
```	
<!-- more -->
## 安装 Swift

```
os_swift="python-swift swift swift-proxy swift-account swift-container swift-object python-memcache xfsprogs"
os_keystone="python-keystone python-keystoneclient"
apt-get install -y $os_swift $os_keystone
```

安装之后需要手工创建 swift相关配置文件:

```
mkdir /etc/swift
touch /etc/swift/swift.conf
touch /etc/swift/proxy-server.conf
chown -R swift:swift /etc/swift
```

添加 swift.conf内容：

```
cat > /etc/swift/swift.conf << _END_
[swift-hash]
# od -t x8 -N 8 -A n < /dev/random
# The above command can be used to generate random a string.
swift_hash_path_suffix = 50ea1ddb6e88b991
_END_
```

将以下内容添加到 /etc/swift/proxy-server.conf内容：

```
[DEFAULT]
bind_port = 8080
bind_ip = 0.0.0.0
user = swift
swift_dir = /etc/swift

log_facility = LOG_LOCAL0
log_level = DEBUG

[pipeline:main]
pipeline = catch_errors healthcheck cache authtoken keystoneauth container-quotas account-quotas proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true

[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = Member,admin

[filter:authtoken]
paste.filter_factory = keystone.middleware.auth_token:filter_factory
service_protocol = http
service_port = 5000
service_host = 192.168.30.150
auth_port = 35357
auth_host = 192.168.30.150
auth_protocol = http
admin_tenant_name = service
admin_user = swift
admin_password = password
signing_dir = /etc/swift

[filter:cache]
use = egg:swift#memcache
set log_name = cache
memcache_servers = 192.168.30.150:11211

[filter:catch_errors]
use = egg:swift#catch_errors

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:ratelimit]
use = egg:swift#ratelimit
[filter:container-quotas]
use = egg:swift#container_quotas

[filter:account-quotas]
use = egg:swift#account_quotas
```

如果不使用 Keystone 认证，请使用以下的配置文件：

```
[DEFAULT]
bind_port = 8080
bind_ip = 192.168.30.150
user = swift

[pipeline:main]
pipeline = healthcheck cache tempauth proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true

[filter:tempauth]
use = egg:swift#tempauth
user_admin_admin = admin .admin .reseller_admin
user_test_tester = testing .admin
user_test2_tester2 = testing2 .admin
user_test_tester3 = testing3

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:cache]
use = egg:swift#memcache
memcache_servers = 192.168.30.150:11211
```

格式： user_<login1>_<login2> = <password> <privileges>
登录的时候就是：

```
login = admin:admin
password = admin
privileges = .admin .reseller_admin
```


配置rsyslog

```
echo "local0.*    /var/log/swift/proxy-server.log" >> /etc/rsyslog.conf
mkdir /var/log/swift
```

配置 环 Ring

```
cd /etc/swift
sudo swift-ring-builder account.builder create 6 2 1
sudo swift-ring-builder container.builder create 6 2 1
sudo swift-ring-builder object.builder create 6 2 1
```
说明

  - 第一个数字：6表示分区(环)将被处理为2^6th，即使用2的6次方个分区，创建完之后应有 64个分区
  - 第二个数字：每个存储对象保存2份，即创建2个副本；由于偶只有两台storage，故只写2
  - 第三个数字：1表示限制分区数据转移的时间，此处表示1小时，即分区被连续移动两次之间的最小时间间隔

添加设备
添加新设备到Ring上，但add操作不会分配partitions到新的设备上，只有运行“rebalance”命令后才会进行分区的分配，所以这种方式可以有这种优势： 允许一次添加多个设备，只执行一次rebalance就可以了，以下操作步骤：

```
sudo swift-ring-builder account.builder add z1-192.168.30.151:6002/sdb1 100
sudo swift-ring-builder account.builder add z2-192.168.30.152:6002/sdb1 100

sudo swift-ring-builder container.builder add z1-192.168.30.151:6001/sdb1 100
sudo swift-ring-builder container.builder add z2-192.168.30.152:6001/sdb1 100

sudo swift-ring-builder object.builder add z1-192.168.30.151:6000/sdb1 100
sudo swift-ring-builder object.builder add z2-192.168.30.152:6000/sdb1 100
```

查看 Ring信息 
可通过以下命令查到到Ring和Ring中的设备信息：
* 查询account信息：

		swift-ring-builder account.builder

* 查询container信息：

		swift-ring-builder container.builder

* 查询object信息
	
		swift-ring-builder object.builder
	
* 生成 Ring
如果确认一切之后，最终还要生成Ring，来进行分区的分配，即之前提到的rebalance：

```
sudo swift-ring-builder account.builder rebalance
sudo swift-ring-builder container.builder rebalance
sudo swift-ring-builder object.builder rebalance
```

设置权限

```
chown -R swift:swift /etc/swift
chown -R swift:swift /var/cache/swift
```

---
## 存储节点安装与配置

添加设备：
先创建分区，另外一定要是 XFS文件系统

```
mkdir -p /srv/node/sdb1
chown -R swift:swift /srv/node/
mkfs.xfs -i size=1024 /dev/sdb1 -f 
echo "/dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab 
mount /srv/node/sdb1  
```

设置 rsyncd.conf

```
uid = swift
gid = swift

log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 192.168.30.151

[account]
max_connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/account.lock

[container]
max_connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/container.lock

[object]
max_connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/object.lock
```

设置 rsync开机自启动

    sudo sed -i 's/RSYNC_ENABLE=false/RSYNC_ENABLE=true/g' /etc/default/rsync

启动 rsync服务

    sudo service rsync start


创建/etc/account-server.conf：

```
[DEFAULT]
bind_ip = 0.0.0.0
bind_port = 6002
workers = 1
user = swift
swift_dir = /etc/swift
devices = /srv/node

[pipeline:main]
pipeline = account-server

[app:account-server]
use = egg:swift#account

[account-replicator]

[account-auditor]

[account-reaper]

```


创建/etc/container-server.conf

```
[DEFAULT]
bind_ip = 0.0.0.0
bind_port = 6001
workers = 1
user = swift
swift_dir = /etc/swift
devices = /srv/node

[pipeline:main]
pipeline = container-server

[app:container-server]
use = egg:swift#container

[container-replicator]

[container-updater]

[container-auditor]

[container-sync]
```

创建/etc/object-server.conf

```
[DEFAULT]
bind_ip = 0.0.0.0
bind_port = 6000
workers = 1
user = swift
swift_dir = /etc/swift
devices = /srv/node

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift

[object-replicator]

[object-updater]

[object-auditor]
```

### 重启服务
在所有存储节点重启以下服务：

```
sudo swift-init object-server start
sudo swift-init object-replicator start
sudo swift-init object-updater start
sudo swift-init object-auditor start
sudo swift-init container-server start
sudo swift-init container-replicator start
sudo swift-init container-updater start
sudo swift-init container-auditor start
sudo swift-init account-server start
sudo swift-init account-replicator start
sudo swift-init account-auditor start
```

在代理节点启动以下服务：

```
sudo swift-init all restart
```


## Swift操作
获得 X-Storage-Url 和 X-Auth-Token:

	curl -k -v -H 'X-Storage-User: admin:admin' -H 'X-Storage-Pass: admin' http://192.168.30.150:8080/auth/v1.0

如果正确，将会返回以下类似信息：

```
* About to connect() to 192.168.30.150 port 8080 (#0)
*   Trying 192.168.30.150... connected
> GET /auth/v1.0 HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: 192.168.30.150:8080
> Accept: */*
> X-Storage-User: admin:admin
> X-Storage-Pass: admin
> 
< HTTP/1.1 200 OK
< X-Storage-Url: http://192.168.30.150:8080/v1/AUTH_admin
< X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5
< Content-Type: text/html; charset=UTF-8
< X-Storage-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5
< Content-Length: 0
< Date: Tue, 15 Oct 2013 01:49:59 GMT
< 
* Connection #0 to host 192.168.30.150 left intact
* Closing connection #0
```	

检查账号

    	curl -k -v -H 'X-Auth-Token: <token-from-x-auth-token-above>' <url-from-xstorage-url-above>  

这里的token-from-x-auth-token-above 就是上面输出的 AUTH_tk8a85916d63b14c568a4633b7920623c5，url-from-xstorage-url-above对应：http://192.168.30.150:8080/v1/AUTH_admin

检测 swift 命令是否工作正常 
	
    	swift -A http://192.168.30.150:8080/auth/v1.0 -U admin:admin -K admin stat

正常输出类似以下信息：

```
   Account: AUTH_admin
Containers: 0
   Objects: 0
     Bytes: 0
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
Content-Type: text/plain; charset=utf-8
```

上传
	swift -A http://192.168.30.150:8080/auth/v1.0 -U admin:admin -K admin upload test apache-tomcat-6.0.36.tar.gz
	
删除
	swift -A http://192.168.30.150:8080/auth/v1.0 -U admin:admin -K admin delete test apache-tomcat-6.0.36.tar.gz

## 排错思路
1. 直接看控制台打印的日志
2. 检查配置文件是否正确
3. 通过观察日志，例如/var/log/syslog
4. 修改配置文件之后，需要重启对应的服务