---
layout: post
title: "Openstack安装与部署(Havana)"
date: 2013-11-10 13:19
comments: true
categories:
- 云计算
- OpenStack
tags:
- Cinder
- Glance
- Horizon
- Keystone
- Nova
- OpenStack
- 云计算

---
## 前言
首先得说明一下，此教程是基于H版部署，由于时间关系，文章可能会有错误，如果有发现错误之处，请直接回复，感谢。

##环境准备
###资源列表

 {% img /images/2013/11/openstack-host-list.png  %}

###网络配置
控制节点controller网络配置如下：

```
# Internal Network
auto eth0
iface eth0 inet static
address 10.0.0.11
netmask 255.255.255.0

#External Netwok
auto eth1
iface eth1 inet static
address 192.168.30.150
netmask 255.255.255.0
gateway 192.168.30.1
dns-nameservers 218.201.4.3
```

计算节点compute01网络配置如下：

```
# Internal Network
auto eth0
iface eth0 inet static
address 10.0.0.13
netmask 255.255.255.0

#External Netwok
auto eth1
iface eth1 inet static
address 192.168.30.151
netmask 255.255.255.0
gateway 192.168.30.1
dns-nameservers 218.201.4.3
```

设置hostname，在所有节点的/etc/hosts中加入：
```
10.0.0.11       controller
10.0.0.13       compute01
```

重启网络
```
service networking restart
```

IP转发
```
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
sysctl -p
```

###时间同步
安装NTP：

```
apt-get install -y ntp
sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
sudo /etc/init.d/ntp restart
```

其他节点的server都指向控制节点：10.0.0.11

###Mysql和rabbitmq安装
在控制节点执行，安装过程中会提示输入几个密码，此处为了方便，密码统一设置为openstack (是不是应该打马赛克？)：

```
apt-get install -y mysql-server python-mysqldb 
sed -i 's/127.0.0.1/10.0.0.11/g' /etc/mysql/my.cnf
/etc/init.d/mysql restart

apt-get install -y rabbitmq-server
```

###添加Havana 仓库
安装 Keyring
```
apt-get install ubuntu-cloud-keyring
```

添加仓库源
```
echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/havana main" >> /etc/apt/sources.list.d/cloudarchive-havana.list
echo "deb-src http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/havana main" >> /etc/apt/sources.list.d/cloudarchive-havana.list
```

更新系统
```
apt-get update
apt-get dist-upgrade
```
如果遇到无法update，多半是网络运营商缓存问题导致的，这时使用代理搞定吧：
```
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8087/" update
```

<!-- more -->

###初始化数据库
在控制节点执行，创建数据库，并设置：
```
mysql -uroot -p
```

输入密码后执行以下SQL语句
```
CREATE DATABASE keystone;
CREATE DATABASE nova;
CREATE DATABASE cinder;
CREATE DATABASE glance;
CREATE DATABASE neutron;
CREATE DATABASE heat;
GRANT ALL ON neutron.* TO neutron@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON neutron.* TO neutron@'10.0.0.11' IDENTIFIED BY 'openstack';
GRANT ALL ON keystone.* TO neutron@'%' IDENTIFIED BY 'openstack';

GRANT ALL ON keystone.* TO keystone@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON keystone.* TO keystone@'10.0.0.11' IDENTIFIED BY 'openstack';
GRANT ALL ON nova.* TO nova@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON nova.* TO nova@'10.0.0.11' IDENTIFIED BY 'openstack';
GRANT ALL ON cinder.* TO cinder@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON cinder.* TO cinder@'10.0.0.11' IDENTIFIED BY 'openstack';
GRANT ALL ON glance.* TO glance@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON glance.* TO glance@'10.0.0.11' IDENTIFIED BY 'openstack';
GRANT ALL ON heat.* TO heat@'%' IDENTIFIED BY 'openstack';
GRANT ALL ON heat.* TO heat@'10.0.0.11' IDENTIFIED BY 'openstack';

flush privileges;
```
需要注意的是以上的IP需要你自己根据实际情况而修改。

---
##控制节点

###Identity Service(Keystone)安装
在控制节点安装认证服务keystone
```
apt-get install -y keystone python-keystone python-keystoneclient
```

####Keystone配置
keystone支持mysql、sqlite及LDAP，以下示例我们将使用Mysql来存储keystone认证信息。

在控制节点修改/etc/keystone/keystone.conf配置文件，注释第78行：

    connection = sqlite:////var/lib/keystone/keystone.db
加入：
    connection = mysql://keystone:openstack@10.0.0.11/keystone

重启keystone后，初始化数据库：

```
keystone-manage db_sync
/etc/init.d/keystone restart
```

配置openssl
在keystone与OpenStack服务之间如果要使用ssl进行token认证，就需要使用到openssl
```
openssl rand -hex 10
8ac14959ce3d1a4fe213
```

修改/etc/keystone/keystone.conf的 [DEFAULT]章节
将 admin_token = admin 修改成：
    
    admin_token = 8ac14959ce3d1a4fe213

再次重启keystone，使其生效

    /etc/init.d/keystone restart

创建User、Tenant、Role、Endpoint、Service
首先创建两个环境变量 
```
export SERVICE_TOKEN=8ac14959ce3d1a4fe213 
export SERVICE_ENDPOINT=http://10.0.0.11:35357/v2.0
```

####创建租户
创建两个租户分别是：admin和service
```
keystone tenant-create --name=admin --description="Admin Tenant"
keystone tenant-create --name=service --description="Service Tenant"
```
上面2个id 暂时需要记录下来，后面将会用到，例如这里是：
    admin	a2ba2a4cd7f54d3b9bed90e16def2b8f
    service	208807d34ff7455e93b39a2820265349

####创建用户

```
keystone user-create --name=admin --pass=openstack \
--email=admin@domain.com --tenant_id=a2ba2a4cd7f54d3b9bed90e16def2b8f \
--enable=True
```
user id同样需要记录下来，例如这里是：
UserId	4fe9772510254b1083962b77d72643c3

####创建角色
创建一个名称为admin的角色

    keystone role-create --name=admin

截止到目前，已经分别创建 Tenant、User、Role，先不管Service这个租户，分别是：

```
Tenant ID: a2ba2a4cd7f54d3b9bed90e16def2b8f
User ID:	133e4aa56bba4400a2ccfd4a3ec08b18
Role ID:	0a788b82f9bb4e849af2b402931f58d2
```
H版现在不需要我们记录具体的id，貌似只需要指定 名称 即可。

#### 关联用户、角色、租户
光创建上面那些身份还不够，最终需要将它们关联起来才能正常使用

```
keystone user-role-add --user admin --role admin --tenant admin
```

####创建Services 及 API endpoints
首先创建一个类型为identity的keystone服务，名称为keystone：

```
keystone service-create --name=keystone --type=identity \
--description="Keystone Identity Service"
```

创建endpoint，同样只需要记住service name即可：

```
keystone endpoint-create --region regionOne \
--service  keystone  \
--publicurl http://10.0.0.11:5000/v2.0 \
--internalurl http://10.0.0.11:5000/v2.0 \
--adminurl http://10.0.0.11:35357/v2.0
```

验证认证服务(Keystone)安装是否成功
先unset之前的环境变量：

    unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

创建keystonerc(环境变量)

```
export OS_REGION_NAME=regionOne
export OS_USERNAME=admin
#export OS_AUTH_STRATEGE=keystone
export OS_PASSWORD=openstack
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://10.0.0.11:35357/v2.0
OS_NO_CACHE=true
```

添加环境变量
为避免我们每次登录都要设置环境变量，现在将上面参数直接写到一个文件中，在每天登录时使其自动生效，这样我们后期就可以直接通过  keystone subcommand 来使用，创建keystonerc文件，并将下列信息写入keystonerc ：

```
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://10.0.0.11:35357/v2.0
```
保存之后并使环境变量立即生效：

    source keystonerc

如果没问题，就能看到token信息。但如果有错误，建议注销当前用户，重新登录后测试一下。


测试是否正常：

    keystone token-get
    
如果配置正常，将会显示token expires\id\tenant_id\user_id四行。
下一步，我们加一个 --os_tenant_name参数试下：

    keystone --os_tenant_name admin token-get
    
如果没问题，将会响应和上面类似的输出信息，区别仅是多一行 tenant_id

---
###Image Service(Glance)安装
镜像服务我们安装在keystone同一台机器，首先直接安装 镜像服务：

    apt-get install -y glance

####Glance配置
更新glance配置文件glance-registry.conf：

    vim /etc/glance/glance-registry.conf

修改mysql数据库连接字符串：

    sql_connection = mysql://glance:openstack@10.0.0.11/glance

更新glance配置文件：

    vim /etc/glance/glance-api.conf

在[DEFAULT]章节加入：

    sql_connection = mysql://glance:openstack@10.0.0.11/glance

同步数据库，将会创建镜像服务相关的表：

    glance-manage db_sync

####创建用户
创建名称为glance的认证用户

```
keystone user-create --name=glance --pass=glance \
--email=glance@domain.com \
--tenant service --enable=True
```
注意，此处的 tenant 就是租户service。

配置Glance与 Keystone访问
编辑配置文件/etc/glance/glance-api.conf 和 /etc/glance/glance-registry.conf，在最后加入以下内容：

```
[keystone_authtoken]
auth_host = 10.0.0.11
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = glance
```

编辑配置文件/etc/glance/glance-api-paste.ini和/etc/glance/glance-registry-paste.ini，将最后几行修改成：

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 10.0.0.11
service_port = 5000
auth_host = 10.0.0.11
auth_port = 35357
auth_protocol = http
auth_uri = http://10.0.0.11:5000/
admin_tenant_name = service
admin_user = glance
admin_password = glance
```

####创建service服务和endpoint

    keystone service-create --name=glance --type=image --description="Glance Image Service"

记录service id，下面创建endpoint将会用到。

```
keystone endpoint-create --region regionOne \
--service glance \
--publicurl=http://10.0.0.11:9292 \
--internalurl=http://10.0.0.11:9292 \
--adminurl=http://10.0.0.11:9292
```

####重启glance服务

```
/etc/init.d/glance-registry restart
/etc/init.d/glance-api restart
```

测试镜像服务是否安装成功
镜像文件可以从互联网下载，或自己制作，centos-63.x86_64-cloud.qcow2是偶自己制作的一个镜像文件，即创建一个名称为 centos-63.x86_64-cloud的镜像文件： 

```
glance image-create --name centos-63.x86_64-cloud \
--is-public true --container-format bare \
--disk-format qcow2 < /home/openstack/centos-63.x86_64-cloud.qcow2
```

如果一切正常，会输出类似以下内容：
```
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 5c80ad5a5667308b74f6f024317f9924     |
| container_format | bare                                 |
| created_at       | 2013-10-21T12:21:24                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 79a7fe9f-7f0a-480f-a60d-68f9271ce230 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | centos-63.x86_64-cloud               |
| owner            | None                                 |
| protected        | False                                |
| size             | 462219776                            |
| status           | active                               |
| updated_at       | 2013-10-21T12:21:27                  |
+------------------+--------------------------------------+
```

检查上传是否成功

    glance image-list

如果 Status字段为 active表示上传成功。


###Compute Service(Nova)安装
控制节点如果在管理并控制计算服务，需要依赖nova，首先安装以下nova包：

```
apt-get install -y nova-novncproxy novnc nova-api \
nova-ajax-console-proxy nova-cert nova-conductor \
nova-consoleauth nova-doc nova-scheduler python-novaclient
```

####Nova配置
修改/etc/nova/nova.conf配置文件，在最后加入：

    sql_connection=mysql://nova:openstack@10.0.0.11/nova

为nova服务创建数据库表：

    nova-manage db sync

配置VNC，修改配置文件/etc/nova/nova.conf，在最后加入：

```
# VNC 
novnc_enabled=True
my_ip=192.168.30.150
novncproxy_base_url=http://192.168.30.150:6080/vnc_auto.html
vncserver_listen=0.0.0.0
```

####创建用户与角色
现在创建一个名称为nova的认证用户和角色

```
keystone user-create --name=nova --pass=nova \
--email=nova@domain.com \
--tenant service --enable=True
keystone user-role-add --user=nova --tenant=service --role=admin
```

修改配置文件/etc/nova/nova.conf

```
[DEFAULT]

auth_strategy=keystone
```

修改配置文件/etc/nova/api-paste.ini,并修改以下内容：
```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.0.0.11
auth_host = 10.0.0.11
auth_port = 35357
admin_tenant_name = service
admin_user = nova
admin_password = nova
```

####创建service服务和endpoint

    keystone service-create --name=nova --type=compute --description="Nova Compute Service"

创建endpoint
```
keystone endpoint-create --service nova \
 --publicurl=http://10.0.0.11:8774/v2/%\(tenant_id\)s \
 --internalurl=http://10.0.0.11:8774/v2/%\(tenant_id\)s \
 --adminurl=http://10.0.0.11:8774/v2/%\(tenant_id\)s
```

配置计算服务使用RabbitMQ,修改配置文件/etc/nova/nova.conf，添加以下内容：

```
rpc_backend=nova.rpc.impl_kombu
rabbit_host = 10.0.0.11
```

nova.conf全部内容如下：

```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
iscsi_helper=tgtadm
libvirt_use_virtio_for_bridges=True
libvirt_type=qemu
connection_type=libvirt
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
volumes_path=/var/lib/nova/volumes
enabled_apis=ec2,osapi_compute,metadata
sql_connection=mysql://nova:openstack@10.0.0.11/nova

#Metadata
metadata_host=10.0.0.11
metadata_manager=nova.api.manager.MetadataManager
metadata_listen=10.0.0.11
metadata_listen_port=8775
#service_neutron_metadata_proxy=True
#neutron_metadata_proxy_shared_secret=helloOpenStack

# Cinder 
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900

auth_strategy=keystone
rpc_backend=nova.rpc.impl_kombu
rabbit_host = 10.0.0.11

# VNC 
novnc_enabled=True
my_ip=192.168.30.150
novncproxy_base_url=http://192.168.30.150:6080/vnc_auto.html
vncserver_listen=0.0.0.0
#vncserver_proxyclient_address=192.168.30.150


#Network settings
network_manager=nova.network.manager.FlatDHCPManager
force_dhcp_release=True
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
public_interface=eth2
flat_interface=eth0
floating_range=192.168.30.0/24
flat_network_bridge=br100
fixed_range=10.0.0.0/24   
network_size=256
# disabled ipv6
flat_injected=False
connection_type=libvirt
multi_host=True
```

重启所有nova服务

    cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done

测试nova是否安装正常

    nova image-list
如果正常，输出的信息和glance image-list是一样的。

###Dashborad(Horizon)安装
####安装dashboard
这一步相对容易多了，请看操作：

    apt-get install -y memcached libapache2-mod-wsgi openstack-dashboard

修改/etc/openstack-dashboard/local_settings.py
将OPENSTACK_HOST修改为：

    OPENSTACK_HOST = "10.0.0.11"

登陆dashborad：http://192.168.30.150/horizon

---
##计算节点
以下的示例暂时以一台计算节点来测试安装与部署，即10.0.0.13

###Compute Service(Nova)安装
为计算节点安装相关的包
```
apt-get install -y python-mysqldb 
apt-get install nova-compute-kvm python-novaclient python-guestfs
```

安装过程会提示你，选择”yes”即可。

以下是修复guestfs的一个bug

    chmod 0644 /boot/vmlinuz*

删除SQLite创建的数据库

    rm -f /var/lib/nova/nova.sqlite

配置VNC，编辑/etc/nova/nova.conf在[DEFAULT]章节添加以下信息

```
#VNC
novnc_enabled=True
novncproxy_base_url=http://192.168.30.150:6080/vnc_auto.html
novncproxy_port=6080
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=192.168.30.151
```

编辑/etc/nova/nova.conf，在[DEFAULT]章节添加以下信息

```
glance_host=10.0.0.11
sql_connection=mysql://nova:openstack@10.0.0.11/nova
```

添加keystone认证，从控制节点复制api-paste.conf或编辑/etc/nova/api-paste.conf，并添加以下信息

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.0.0.11
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = nova
```

####启用网络
在计算节点启用网络
本节的网络示例均使用的nova-network，如果使用neutron，请见Networking Server安装一章。
首先为计算服务安装nova网络包

    apt-get install -y nova-network

设置网络，编辑/etc/nova/nova.conf，增加以下信息

```
network_manager=nova.network.manager.FlatDHCPManager
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
network_size=254
allow_same_net_traffic=False
multi_host=True
send_arp_for_ha=True
share_dhcp_address=True
force_dhcp_release=True
flat_network_bridge=br100
flat_interface=eth1
public_interface=eth1
```

nova.conf 全部内容如下：

```
[DEFAULT]
#notification_driver=ceilometer.compute.nova_notifier
#notification_driver=nova.openstack.common.notifier.rpc_notifier
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
iscsi_helper=tgtadm
libvirt_use_virtio_for_bridges=True
connection_type=libvirt
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
volumes_path=/var/lib/nova/volumes
enabled_apis=ec2,osapi_compute,metadata
sql_connection=mysql://nova:openstack@10.0.0.11/nova
libvirt_type=qemu

# Cinder
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900
cinder_catalog_info=volume:cinder:internalURL

# Metadata
metadata_host=10.0.0.11
metadata_manager=nova.api.manager.MetadataManager
metadata_listen=10.0.0.11
metadata_listen_port=8775
#service_neutron_metadata_proxy=True
#neutron_metadata_proxy_shared_secret=helloOpenStack

# Compute
compute_driver=libvirt.LibvirtDriver

# RabbitMQ
rpc_backend = nova.rpc.impl_kombu
rabbit_host = 10.0.0.11

# Image Servic3
glance_host=10.0.0.11
image_service=nova.image.glance.GlanceImageService

#VNC
novnc_enabled=True
#my_ip=192.168.30.151
novncproxy_base_url=http://192.168.30.150:6080/vnc_auto.html
novncproxy_port=6080
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=192.168.30.151

# Netwok settings
network_manager=nova.network.manager.FlatDHCPManager
force_dhcp_release=True
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
public_interface=eth2
flat_interface=eth0
floating_range=192.168.30.0/24
flat_network_dhcp_start=10.0.0.40
flat_network_bridge=br100
fixed_range=10.0.0.1/24
#network_size=256
flat_injected=False
connection_type=libvirt
multi_host=True
dhcp_lease_time=604800
iscsi_helper=tgtadm
```

重启nova服务

```
/etc/init.d/nova-api restart
/etc/init.d/nova-compute restart
```
 
---

##Block Storage (Cinder)安装
首先得准备一台机器，以下的示例将是在控制节点来安装，同样，你也可以选择在控制节点安装，以下示例中的磁盘设备是/dev/sdb，100G

###配置块存储控制
安装cinder服务依赖包

    apt-get install -y cinder-api cinder-scheduler

配置块存储服务去使用mysql数据库，编辑/etc/cinder/cinder.conf,添加以下内容

    connection = mysql://cinder:openstack@10.0.0.11/cinder

为块存储服务创建数据库表

    cinder-manage db sync

####创建用户和角色
创建一个 cinder用户，块存储服务使用这个用户去身份认证，使用service租户且给它admin角色

```
keystone user-create --name=cinder --pass=cinder --email=cinder@domain.com \
 --tenant service --enable=True
keystone user-role-add --user=cinder --tenant=service --role=admin
```

编辑/etc/cinder/api-paste.ini，修改以下信息

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.0.0.11
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = cinder
```

####创建service和endpoint
```
keystone service-create --name=cinder --type=volume --description="Cinder Volume Service"
keystone endpoint-create --service cinder \
--publicurl=http://10.0.0.11:8776/v1/%\(tenant_id\)s \
--internalurl=http://10.0.0.11:8776/v1/%\(tenant_id\)s \
--adminurl=http://10.0.0.11:8776/v1/%\(tenant_id\)s
```

再来创建个v2的API
```
keystone service-create --name=cinder --type=volumev2 \
 --description="Cinder Volume Service V2"
```
 
记录service id，以下将会使用到
```
keystone endpoint-create \
 --service=98b37a6bd88c44a19bc38586c753cfdd \
 --publicurl=http://10.0.0.11:8776/v2/%\(tenant_id\)s \
 --internalurl=http://10.0.0.11:8776/v2/%\(tenant_id\)s \
 --adminurl=http://10.0.0.11:8776/v2/%\(tenant_id\)s
```

重启cinder服务
```
service cinder-scheduler restart
service cinder-api restart
```

###配置块存储节点
配置块存储服务使用RabbitMQ，编辑/etc/cinder/cinder.conf，增加以下信息
```
rpc_backend = cinder.openstack.common.rpc.impl_kombu
rabbit_host = 10.0.0.11
rabbit_port = 5672
```

现在，我们来创建LVM的PV及LV，所以示例全部是使用的第二块设备/dev/sdb，在操作前请确保你的数据是否备份。

安装lvm2和cinder-volume

    apt-get install cinder-volume lvm2

创建PV 和 VG
```
root@controller:~# pvcreate /dev/sdb 
root@controller:~# vgcreate cinder-volumes /dev/sdb
```

重启cinder服务
```
service cinder-volume restart
service tgt restart
```

最后去 horizon创建卷就可以使用了，步骤大概为：
1.	管理员自定义卷类型
2.	租户创建卷，并选择一个卷类型
3.	附加到某个VM即可。

---

##Orchestration Server(Heat)安装
准备一台机器，以下示例将把Heat安装在控制节点，同样，你也可以选择在控制节点安装。

###安装Orchestration服务

    apt-get install -y heat-api heat-api-cfn 

初始化数据库服务，具体见最开始的 初始化数据库。

###Orchestration配置
编辑/etc/heat/heat.conf，修改[DEFAULT]节：

    sql_connection=mysql://heat:openstack@10.0.0.11/heat

同步数据库，将会自动创建heat服务的数据库表

    heat-manage db_sync

###创建用户与角色
创建一个名称为heat的认证用户和角色
```
keystone user-create --name=heat --pass=heat \
 --email=heat@domain.com \
 --tenant service --enable=True
```
记录user id:  c8b8757e829f461e99a5aa0a72ef78b5

关联租户(service)、用户、角色(admin)
```
keystone user-role-add --user-id 6570003a874d4dcc9fd4e3c7f6f16db6 \
 --tenant-id 208807d34ff7455e93b39a2820265349 \
 --role-id 0a788b82f9bb4e849af2b402931f58d2
```

###创建Heat Service和Endpoint
创建service

```
keystone service-create --name=heat --type=orchestration \
--description="Heat Orchestration API"

keystone endpoint-create --region regionOne \
--service-id a5743b1fa56b452f9608d17fdd4dfa2b \
--publicurl=http://10.0.0.11:8004/v1/%\(tenant_id\)s \
--internalurl=http://10.0.0.11:8004/v1/%\(tenant_id\)s  \
--adminurl=http://10.0.0.11:8004/v1/%\(tenant_id\)s
```

heat service id: 06d78bd1404d48698e3f28c01bb2ad30
heat-cfn service id: 507fa3386d4f4d1caa5dc075621e669a

创建endpoint
```
keystone service-create --name=heat-cfn --type=cloudformation \
--description="Heat CloudFormation API"

keystone endpoint-create --region regionOne \
--service-id a7fae41b733f40efb7472b2a17b97794 \
--publicurl=http://10.0.0.11:8000/v1 \
--internalurl=http://10.0.0.11:8000/v1  \
--adminurl=http://10.0.0.11:8000/v1
```

编辑/etc/heat/api-paste.ini，修改[filter:authtoken]节
```
[filter:authtoken]
paste.filter_factory = heat.common.auth_token:filter_factory
auth_host = 10.0.0.11
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = heat
admin_password = heat
```

重启Heat服务

    cd /etc/init.d/; for i in $( ls heat-* ); do sudo service $i restart; done

###创建和管理stacks
从模板创建一个stack,其实官方已经提供很多模板，所以暂时先git下来做测试

    git clone https://github.com/openstack/heat-templates.git


创建stack

```
heat stack-create mystack \
--template-file=/root/heat-templates/cfn/F18/WordPress_Single_Instance.template \
--parameters="InstanceType=m1.large;DBUsername=wordpress;DBPassword=wordpress;KeyName=HEAT_KEY;LinuxDistribution=F18"
```
具体参数请参与官方手册。

---

##Metering/Monitoring Server（Ceilometer）安装

###安装Metering Service

**1. 在控制节点安装依赖包 **

```
apt-get install -y ceilometer-api ceilometer-collector \
ceilometer-agent-central python-ceilometerclient
```

**2. 安装MongoDB **
Orchestration 服务使用数据库来存储信息，此示例使用MongoDB数据库：

```
apt-get install -y mongodb
```

建议将/etc/mongodb.conf中的bind_ip修改为：
```
bind_ip=0.0.0.0
```

**3. 重启mongodb生效 **

```
/etc/init.d/mongodb restart
```

**4. 创建数据库和一个 ceilometer用户 **

```
root@controller:~# mongo
> 
> use ceilometer
switched to db ceilometer
> db.addUser ( { user: "ceilometer",
             pwd: "ceilometer",
             roles: [ "readWrite", "dbAdmin"]
     } )
```

**5. 编辑/etc/ceilometer/ceilometer.conf，替换如下： **

```
[DEFAULT]
pipeline_cfg_file=pipeline.yaml
sample_source=openstack
nova_control_exchange=nova
libvirt_type=qemu
glance_control_exchange=glance
metering_api_port=8777
debug=true
verbose=true
log_file=/var/log/ceilometer/ceilometer.log
log_dir=/var/log/ceilometer
notification_topics=notifications
policy_file=policy.json
rpc_backend=ceilometer.openstack.common.rpc.impl_kombu
allowed_rpc_exception_modules=ceilometer.openstack.common.exception,nova.exception,cinder.exception,exceptions
control_exchange=openstack
rabbit_host=10.0.0.11
rabbit_port=5672
rabbit_userid=guest
rabbit_password=guest
database_connection=mongodb://ceilometer:ceilometer@10.0.0.11:27017/ceilometer
cinder_control_exchange=cinder

[publisher_rpc]
metering_topic=metering
metering_secret=8ac14959ce3d1a4fe213

[ssl]

[database]
database_connection=mongodb://ceilometer:ceilometer@10.0.0.11:27017/ceilometer

[alarm]

[rpc_notifier2]

[api]


[service_credentials]
os_username=ceilometer
os_password=ceilometer
os_tenant_name=service
os_auth_url=http://10.0.0.11:5000/v2.0
os_endpoint_type=publicURL

[dispatcher_file]

[keystone_authtoken]
auth_host=10.0.0.11
auth_port=35357
auth_protocol=http
admin_tenant_name=service
admin_user=ceilometer
admin_password=ceilometer
signing_dir=/var/lib/ceilometer/ceilometer-signing

[collector]

[matchmaker_ring]

[matchmaker_redis]
```

**6. 创建名称为 ceilometer 的用户，以便使用认证服务。使用service 租户并赋予这个用户admin角色: **

```
keystone user-create \
  --name=ceilometer --pass=ceilometer \
  --email=ceilometer@domain.com
  
keystone user-role-add --user=ceilometer --tenant=service --role=admin  
```

**7. 添加服务和endpoint **

```
keystone service-create --name=ceilometer --type=metering \
 --description="Ceilometer Metering Service"
```

记录service id，在下面创建endpoint要使用。

```
keystone endpoint-create \
 --service-id=94ae66fbf5d44c63b7257d0f3c3a48d6 \
 --publicurl=http://10.0.0.11:8777/ \
 --internalurl=http://10.0.0.11:8777/ \
 --adminurl=http://10.0.0.11:8777/
```

**8. 配置认证信息，编辑/etc/ceilometer/ceilometer.conf,修改[keystone_authtoken]节 **

```
[keystone_authtoken]
auth_host=10.0.0.11
auth_port=35357
auth_protocol=http
auth_uri=http://10.0.0.11:35357/v2.0
admin_tenant_name=service
admin_user=ceilometer
admin_password=ceilometer
```

**9. 重启所有ceilometer服务 **
```
cd /etc/init.d/; for i in $( ls ceilometer-* ); do sudo service $i restart; done
```

###添加Agent: 计算
为收集数据，需要在**计算节点**安装一个Agent。

**1. 首先在控制节点安装Metering agent服务 **

            apt-get install ceilometer-agent-compute

**2. 	编辑/etc/nova/nova.conf，在[DEFAULT]节添加以下信息 **

```
# Ceilometer
instance_usage_audit=True
instance_usage_period=hour
notify_on_state_change=vm_and_task_state
notification_driver=nova.openstack.common.notifier.rpc_notifier
notification_driver=ceilometer.compute.nova_notifier
```

**3. 编辑/etc/ceilometer/ceilometer.conf，替换如下： **

```
[DEFAULT]
pipeline_cfg_file=pipeline.yaml
sample_source=openstack
nova_control_exchange=nova
libvirt_type=qemu
glance_control_exchange=glance
metering_api_port=8777
debug=true
verbose=true
log_file=/var/log/ceilometer/ceilometer.log
log_dir=/var/log/ceilometer
notification_topics=notifications
policy_file=policy.json
rpc_backend=ceilometer.openstack.common.rpc.impl_kombu
allowed_rpc_exception_modules=ceilometer.openstack.common.exception,nova.exception,cinder.exception,exceptions
control_exchange=openstack
rabbit_host=10.0.0.11
rabbit_port=5672
rabbit_userid=guest
rabbit_password=guest
database_connection=mongodb://ceilometer:ceilometer@10.0.0.11:27017/ceilometer
cinder_control_exchange=cinder

[publisher_rpc]
metering_topic=metering
metering_secret=8ac14959ce3d1a4fe213

[ssl]

[database]
database_connection=mongodb://ceilometer:ceilometer@10.0.0.11:27017/ceilometer

[alarm]

[rpc_notifier2]

[api]


[service_credentials]
os_username=ceilometer
os_password=ceilometer
os_tenant_name=service
os_auth_url=http://10.0.0.11:5000/v2.0
os_endpoint_type=publicURL

[dispatcher_file]

[keystone_authtoken]
auth_host=10.0.0.11
auth_port=35357
auth_protocol=http
admin_tenant_name=service
admin_user=ceilometer
admin_password=ceilometer
signing_dir=/var/lib/ceilometer/ceilometer-signing

[collector]

[matchmaker_ring]

[matchmaker_redis]

```

**4. 重启compute agent服务 **

```
/etc/init.d/ceilometer-agent-compute restart
```

###添加Agent: 镜像服务
要测量镜像服务(Image Service)，只需要在glance-api.conf中修改notifier_strategy = noop为
notifier_strategy = rabbit 或者 notifier_strategy = qpid，以下示例使用的rabbit。

修改完成后重启镜像服务：

```
/etc/init.d/glance-registry restart
/etc/init.d/glance-api restart
```

###添加Agent: 块存储

**1. 要测量块存储服务(Block Storage)，只需要在cinder.conf中加入以下信息： **

```
# Ceilometer
notification_driver = cinder.openstack.common.notifier.rabbit_notifier
control_exchange = cinder
```


**2. 重启cinder相关服务：**

```
/etc/init.d/cinder-volume restart
/etc/init.d/cinder-api restart
```

###添加Agent: 对象存储

**1. 要访问对象存储(Object Storage)服务就需要ResellerAdmin角色： **

```
keystone role-create --name=ResellerAdmin

keystone user-role-add --tenant_id 208807d34ff7455e93b39a2820265349 \
--user_id c538a8fec188450588b71f9d6b6f1ab5 \
--role_id 56cb34407d5e4ce596e36bd03475be01
```

###测试结果
设置完之后，我们通过Dashboard来查看结果，来张漂亮的图先

{% img /images/2013/11/openstack-resouce-use.png %}
 

