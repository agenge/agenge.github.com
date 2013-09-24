---
layout: post
title: Openstack安装与部署(Folsom)
categories:
- 云计算
tags:
- Cinder
- Glance
- Horizon
- Keystone
- Nova
- OpenStack
- 云计算
published: true
comments: true

---
# 前言
近2、3年来，云计算是一个比较热门的话题，无论是国外还是国内，众多计算机厂商开始涉足云计算，未来IaaS、SaaS及PaaS都可能将IT行业又推上到一个新的台阶。云计算谈得最多的IaaS要数 OpenStack，本文主要记录偶试
验的安装步骤，当然，参考了N多网友的博文，具有代表性的可能就是 [stacklab](http://www.stacklab.org/) 的安装教程，以下就是具体安装步骤：

##环境准备
 ### 资源列表
<table width="582" border="0" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td nowrap="nowrap" width="117">
<p align="center">IP
</td>
<td nowrap="nowrap" width="88">
<p align="center">主机名
</td>
<td nowrap="nowrap" width="87">
<p align="center">OS
</td>
<td nowrap="nowrap" width="70">
<p align="center">CPU
</td>
<td nowrap="nowrap" width="52">
<p align="center">内存
</td>
<td nowrap="nowrap" width="168">
<p align="center">备注
</td>
</tr>
<tr>
<td nowrap="nowrap" width="117">
<p align="center">192.168.30.73
</td>
<td nowrap="nowrap" width="88">
<p align="center">control-01
</td>
<td nowrap="nowrap" width="87">
<p align="center">Ubuntu 12.10
</td>
<td nowrap="nowrap" width="70">
<p align="center">i5
</td>
<td nowrap="nowrap" width="52">
<p align="center">16G
</td>
<td nowrap="nowrap" width="168">
<p align="center">keystone、glance、cinder、nova-api、nova-scheduler
</td>
</tr>
<tr>
<td nowrap="nowrap" width="117">
<p align="center">192.168.30.74
</td>
<td nowrap="nowrap" width="88">
<p align="center">compute-01
</td>
<td nowrap="nowrap" width="87">
<p align="center">Ubuntu 12.10
</td>
<td nowrap="nowrap" width="70">
<p align="center">i5
</td>
<td nowrap="nowrap" width="52">
<p align="center">16G
</td>
<td rowspan="3" nowrap="nowrap" width="168">
<p align="center">nova-compute、nova-network、nova-api-metadata
</td>
</tr>
<tr>
<td nowrap="nowrap" width="117">
<p align="center">192.168.30.75
</td>
<td nowrap="nowrap" width="88">
<p align="center">compute-02
</td>
<td nowrap="nowrap" width="87">
<p align="center">Ubuntu 12.10
</td>
<td nowrap="nowrap" width="70">
<p align="center">i5
</td>
<td nowrap="nowrap" width="52">
<p align="center">16G
</td>
</tr>
<tr>
<td nowrap="nowrap" width="117"></td>
<td nowrap="nowrap" width="88"></td>
<td nowrap="nowrap" width="87"></td>
<td nowrap="nowrap" width="70"></td>
<td nowrap="nowrap" width="52"></td>
</tr>
</tbody>
</table>

### 网络配置

1. 在配置之前需要注意，如果你的网卡名称有叫非ethX，例如p1p1，就得按以下步骤修改：

        cd /etc/udev/rules.d/
        cp 70-persistent-net.rules 70-persistent-net.rules.bak
        ip a
 可看到p1p1的MAC地址，如**d4:3d:7e:57:a4:d4**，将文件70-persistent-net.rules的：
        ATTR{address}=="ec:88:8f:ea:c4:eb"
 修改为：
        ATTR{address}=="d4:3d:7e:57:a4:d4"

 NAME="eth1" 修改为:NAME="eth2" 或 NAME="eth0"，只要不和已存在的冲突即可，例如此示例中，本身已经存在一块叫： eth1的网卡，那另外一块就改成eth2。修改之后需要重启服务器。
所有节点配置如下（需要修改每个节点对应的IP）：
        cat /etc/network/interfaces
        # This file describes the network interfaces available on your system
        # and how to activate them. For more information, see interfaces(5).
        # The loopback network interface
        auto lo
        iface lo inet loopback
        
        # The primary network interface
        auto eth1
        iface eth1 inet static
        address 192.168.30.75
        netmask 255.255.255.0
        network 192.168.30.0
        broadcast 192.168.30.255
        gateway 192.168.30.1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers 218.201.4.3
        
        auto eth2
        iface eth2 inet static
        address 10.10.0.75
        netmask 255.255.0.0

 上面2个IP地址需要根据您的具体情况修改。
2. 重启网络

         /etc/init.d/networking restart
3. IP转发

        sudo su -
        sed -i -r 's/^s*#(net.ipv4.ip_forward=1.*)/1/' /etc/sysctl.conf
        echo 1 > /proc/sys/net/ipv4/ip_forward
        sysctl -p
        net.ipv4.ip_forward = 1
4. 更新源

 在所有节点的/etc/apt/source.list 加入以下信息：
        deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-proposed/folsom main
        deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main

 然后执行：

        gpg --keyserver keyserver.ubuntu.com --recv EC4926EA
        gpg --export --armor EC4926EA | sudo apt-key add -
        sudo apt-get update

 如果报错，请参考[这里](http://blog.csdn.net/wche1990/article/details/6759422).
 
###主机名设置
在所有节点的/etc/hosts加入以下信息（需要修改每个节点对应的IP）：

        192.168.30.73  control-01
        192.168.30.74  compute-01
        192.168.30.75  compute-02


###Mysql和rabbitmq安装
在所有节点执行，密码统一设置为”123456”(你也可以设置为其他较安全的密码)：

        sudo apt-get install -y mysql-server python-mysqldb
        sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
        sudo /etc/init.d/mysql restart
        sudo apt-get install -y rabbitmq-server

###时间同步
在所有节点执行：

        sudo apt-get install -y ntp
        sudo sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.comnserver 127.127.1.0nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
        sudo /etc/init.d/ntp restart

---

<!-- more -->
##控制节点安装
###安装OpenStack组件
1. 在控制节点执行：

        os_keystone="keystone python-keystone python-keystoneclient"
        os_glance="glance glance-api python-glanceclient glance-common"
        os_nova="nova-api nova-cert nova-common  nova-scheduler python-nova python-novaclient nova-consoleauth novnc   nova-novncproxy "
        os_horizon="apache2 libapache2-mod-wsgi openstack-dashboard memcached python-memcache"
        os_cinder="cinder-api cinder-scheduler cinder-volume iscsitarget  open-iscsi iscsitarget-dkms python-cinderclient"
        os_swift="python-swift swift swift-proxy swift-account swift-container swift-object python-memcache"
        sudo apt-get install -y $os_keystone $os_glance $os_nova $os_horizon $os_cinder $os_swift

        大概要下载241M的文件。


###初始化数据库
1. 在控制节点执行，创建keystone, nova,cinder,glance数据库：

        mysql -uroot –p
        mysql> CREATE DATABASE keystone;
        mysql> CREATE DATABASE nova;
        mysql> CREATE DATABASE cinder;
        mysql> CREATE DATABASE glance;
        mysql> GRANT ALL ON keystone.* TO openstack@'%' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON nova.* TO openstack@'%' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON cinder.* TO openstack@'%' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON glance.* TO openstack@'%' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON keystone.* TO openstack@'192.168.30.73' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON nova.* TO openstack@'192.168.30.73' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON nova.* TO openstack@'192.168.30.74' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON nova.* TO openstack@'192.168.30.75' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON cinder.* TO openstack@'192.168.30.73' IDENTIFIED BY 'openstack';
        mysql> GRANT ALL ON glance.* TO openstack@'192.168.30.73' IDENTIFIED BY 'openstack';
        mysql> flush privileges;

###Keystone配置
* 在控制节点修改/etc/keystone/keystone.conf配置文件，注释第57行：

        connection = sqlite:////var/lib/keystone/keystone.db
 加入：

        connection = mysql://openstack:openstack@192.168.30.73/keystone
 重启keystone后，初始化数据库：

        sudo /etc/init.d/keystone restart
        sudo keystone-manage db_sync

2. 创建tenant、user、role，脚本keystone_basic.sh内容如下：

            #!/bin/sh
            #
            # Keystone basic configuration
             
            # Mainly inspired by https://github.com/openstack/keystone/blob/master/tools/sample_data.sh
             
            # Modified by Bilel Msekni / Institut Telecom
            #
            # Support: openstack@lists.launchpad.net
            # License: Apache Software License (ASL) 2.0
            #
            #节点的IP地址
            HOST_IP=192.168.30.73
            ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin_pass}
            SERVICE_PASSWORD=${SERVICE_PASSWORD:-service_pass}
            export SERVICE_TOKEN="ADMIN"
            export SERVICE_ENDPOINT="http://${HOST_IP}:35357/v2.0"
            SERVICE_TENANT_NAME=${SERVICE_TENANT_NAME:-service}
             
            get_id () {
                echo `$@ | awk '/ id / { print $4 }'`
            }
             
            # Tenants
            ADMIN_TENANT=$(get_id keystone tenant-create --name=admin)
            SERVICE_TENANT=$(get_id keystone tenant-create --name=$SERVICE_TENANT_NAME)
             
             
            # Users
            ADMIN_USER=$(get_id keystone user-create --name=admin --pass="$ADMIN_PASSWORD" --email=admin@domain.com)
             
             
            # Roles
            ADMIN_ROLE=$(get_id keystone role-create --name=admin)
            KEYSTONEADMIN_ROLE=$(get_id keystone role-create --name=KeystoneAdmin)
            KEYSTONESERVICE_ROLE=$(get_id keystone role-create --name=KeystoneServiceAdmin)
             
            # Add Roles to Users in Tenants
            keystone user-role-add --user-id $ADMIN_USER --role-id $ADMIN_ROLE --tenant-id $ADMIN_TENANT
            keystone user-role-add --user-id $ADMIN_USER --role-id $KEYSTONEADMIN_ROLE --tenant-id $ADMIN_TENANT
            keystone user-role-add --user-id $ADMIN_USER --role-id $KEYSTONESERVICE_ROLE --tenant-id $ADMIN_TENANT
             
            # The Member role is used by Horizon and Swift
            MEMBER_ROLE=$(get_id keystone role-create --name=Member)
             
            # Configure service users/roles
            NOVA_USER=$(get_id keystone user-create --name=nova --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=nova@domain.com)
            keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $NOVA_USER --role-id $ADMIN_ROLE
             
            GLANCE_USER=$(get_id keystone user-create --name=glance --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=glance@domain.com)
            keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $GLANCE_USER --role-id $ADMIN_ROLE
             
            CINDER_USER=$(get_id keystone user-create --name=cinder --pass="$SERVICE_PASSWORD" --tenant-id $SERVICE_TENANT --email=cinder@domain.com)
            keystone user-role-add --tenant-id $SERVICE_TENANT --user-id $CINDER_USER --role-id $ADMIN_ROLE
            
 * 创建endpoint，脚本keystone_endpoints_basic.sh内容如下:

            #!/bin/sh
            #
            # Keystone basic Endpoints
             
            # Mainly inspired by https://github.com/openstack/keystone/blob/master/tools/sample_data.sh
             
            # Modified by Bilel Msekni / Institut Telecom
            #
            # Support: openstack@lists.launchpad.net
            # License: Apache Software License (ASL) 2.0
            #
             
            # Host address
            #节点的manage network的IP地址
            HOST_IP=192.168.30.73
            #节点的public network的IP地址
            EXT_HOST_IP=192.168.30.73
             
            # MySQL definitions
            MYSQL_USER=openstack
            MYSQL_DATABASE=keystone
            MYSQL_HOST=$HOST_IP
            MYSQL_PASSWORD=openstack
             
            # Keystone definitions
            KEYSTONE_REGION=RegionOne
            export SERVICE_TOKEN=ADMIN
            export SERVICE_ENDPOINT="http://${HOST_IP}:35357/v2.0"
             
            while getopts "u:D:p:m:K:R:E:T:vh" opt; do
            case $opt in
                u)
                  MYSQL_USER=$OPTARG
                  ;;
                D)
                  MYSQL_DATABASE=$OPTARG
                  ;;
                p)
                  MYSQL_PASSWORD=$OPTARG
                  ;;
                m)
                  MYSQL_HOST=$OPTARG
                  ;;
                K)
                  MASTER=$OPTARG
                  ;;
                R)
                  KEYSTONE_REGION=$OPTARG
                  ;;
                E)
                  export SERVICE_ENDPOINT=$OPTARG
                  ;;
                T)
                  export SERVICE_TOKEN=$OPTARG
                  ;;
                v)
                  set -x
                  ;;
                h)
                  cat <<EOF
            Usage: $0 [-m mysql_hostname] [-u mysql_username] [-D mysql_database] [-p mysql_password]
            [-K keystone_master ] [ -R keystone_region ] [ -E keystone_endpoint_url ]
            [ -T keystone_token ]
            Add -v for verbose mode, -h to display this message.
            EOF
                  exit 0
                  ;;
                \?)
                  echo "Unknown option -$OPTARG" >&2
                  exit 1
                  ;;
                :)
                  echo "Option -$OPTARG requires an argument" >&2
                  exit 1
                  ;;
              esac
            done
             
            if [ -z "$KEYSTONE_REGION" ]; then
            echo "Keystone region not set. Please set with -R option or set KEYSTONE_REGION variable." >&2
              missing_args="true"
            fi
             
            if [ -z "$SERVICE_TOKEN" ]; then
            echo "Keystone service token not set. Please set with -T option or set SERVICE_TOKEN variable." >&2
              missing_args="true"
            fi
             
            if [ -z "$SERVICE_ENDPOINT" ]; then
            echo "Keystone service endpoint not set. Please set with -E option or set SERVICE_ENDPOINT variable." >&2
              missing_args="true"
            fi
             
            if [ -z "$MYSQL_PASSWORD" ]; then
            echo "MySQL password not set. Please set with -p option or set MYSQL_PASSWORD variable." >&2
              missing_args="true"
            fi
             
            if [ -n "$missing_args" ]; then
            exit 1
            fi
            keystone service-create --name nova --type compute --description 'OpenStack Compute Service'
            keystone service-create --name cinder --type volume --description 'OpenStack Volume Service'
            keystone service-create --name glance --type image --description 'OpenStack Image Service'
            keystone service-create --name keystone --type identity --description 'OpenStack Identity'
            keystone service-create --name ec2 --type ec2 --description 'OpenStack EC2 service'
             
            create_endpoint () {
              case $1 in
                compute)
                keystone endpoint-create --region $KEYSTONE_REGION --service-id $2 --publicurl 'http://'"$EXT_HOST_IP"':8774/v2/$(tenant_id)s' --adminurl 'http://'"$HOST_IP"':8774/v2/$(tenant_id)s' --internalurl 'http://'"$HOST_IP"':8774/v2/$(tenant_id)s'
                ;;
                volume)
                keystone endpoint-create --region $KEYSTONE_REGION --service-id $2 --publicurl 'http://'"$EXT_HOST_IP"':8776/v1/$(tenant_id)s' --adminurl 'http://'"$HOST_IP"':8776/v1/$(tenant_id)s' --internalurl 'http://'"$HOST_IP"':8776/v1/$(tenant_id)s'
                ;;
                image)
                keystone endpoint-create --region $KEYSTONE_REGION --service-id $2 --publicurl 'http://'"$EXT_HOST_IP"':9292/v2' --adminurl 'http://'"$HOST_IP"':9292/v2' --internalurl 'http://'"$HOST_IP"':9292/v2'
                ;;
                identity)
                keystone endpoint-create --region $KEYSTONE_REGION --service-id $2 --publicurl 'http://'"$EXT_HOST_IP"':5000/v2.0' --adminurl 'http://'"$HOST_IP"':35357/v2.0' --internalurl 'http://'"$HOST_IP"':5000/v2.0'
                ;;
                ec2)
                keystone endpoint-create --region $KEYSTONE_REGION --service-id $2 --publicurl 'http://'"$EXT_HOST_IP"':8773/services/Cloud' --adminurl 'http://'"$HOST_IP"':8773/services/Admin' --internalurl 'http://'"$HOST_IP"':8773/services/Cloud'
                ;;
              esac
            }
             
            for i in compute volume image object-store identity ec2; do
            id=`mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" -ss -e "SELECT id FROM service WHERE type='"$i"';"` || exit 1
              create_endpoint $i $id
            done
            
* 依次执行：

        sudo chmod +x keystone_*.sh
        sudo ./keystone_basic.sh
        sudo ./keystone_endpoints_basic.sh

* 创建openrc文件（环境变量）

        export OS_TENANT_NAME=admin
        export OS_USERNAME=admin
        export OS_PASSWORD=admin_password
        export OS_AUTH_URL=http://192.168.30.73:5000/v2.0/
        export EC2_URL=$(keystone catalog --service ec2 | awk '/ publicURL / { print $4 }')
        export CREDS=$(keystone ec2-credentials-create)
        export EC2_ACCESS_KEY=$(echo "$CREDS" | awk '/ access / { print $4 }')
        export EC2_SECRET_KEY=$(echo "$CREDS" | awk '/ secret / { print $4 }')
* 使环境变量生效：

        source openrc
* 测试keystone

        keystone user-list

如果没问题，输出内容类似下图：

<a href="http://agenge.com/wp-content/uploads/2013/06/01.png"><img class="alignnone size-full wp-image-702" alt="01" src="http://agenge.com/wp-content/uploads/2013/06/01.png" width="678" height="117" /></a>

---
###Glance配置
####更新配置文件
1. 更新glance配置文件：

        sudo vi /etc/glance/glance-api-paste.in

 将最后三行修改为：

        [filter:authtoken]
        paste.filter_factory = keystone.middleware.auth_token:filter_factory
        auth_host = 192.168.30.73
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = service_password

2. 更新glance配置文件：

        sudo vi /etc/glance/glance-registry-paste.ini

 将最后三行修改为：

        [filter:authtoken]
        paste.filter_factory = keystone.middleware.auth_token:filter_factory
        auth_host = 192.168.30.73
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = service_password

3. 更新glance配置文件：

        sudo vim /etc/glance/glance-api.conf

    将49行注释，并加入：

        sql_connection = mysql://openstack:openstack@192.168.30.73/glance

    在[paste_deploy]（一般在最后几行）节点加入：

        flavor = keystone
4. 更新glance配置文件：

        sudo vim /etc/glance/glance-registry.conf

    将28行注释，并加入：

        sql_connection = mysql://openstack:openstack@192.168.30.73/glance
        [paste_deploy]
        flavor = keystone
5. 重启glance服务

        /etc/init.d/glance-api restart
        sudo /etc/init.d/glance-registry restart

6. 同步数据库

        sudo glance-manage db_sync

    并再次重启glance服务：

        sudo /etc/init.d/glance-api restart
        sudo /etc/init.d/glance-registry restart

####上传镜像文件

    wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
    source openrc
    glance image-create --name myFirstVM --is-public true --container-format bare --disk-format qcow2 &lt; cirros-0.3.0-x86_64-disk.img

如果一切正常，会输出类似以下内容：

<a href="http://agenge.com/wp-content/uploads/2013/06/02.png"><img class="alignnone size-full wp-image-703" alt="02" src="http://agenge.com/wp-content/uploads/2013/06/02.png" width="536" height="300" /></a>

---
###Cinder配置
####配置iscsi
1. 配置iscsitarget：

        sudo sed -i 's/false/true/g' /etc/default/iscsitarget
        sudo /etc/init.d/iscsitarget restart
        sudo /etc/init.d/open-iscsi restart

2. 更新配置文件/etc/cinder/api-paste.ini（47行）：

        [filter:authtoken]
        paste.filter_factory = keystone.middleware.auth_token:filter_factory
        service_protocol = http
        service_host = 192.168.30.73
        service_port = 5000
        auth_host = 192.168.30.73
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = cinder
        admin_password = service_password

3. 更新配置文件/etc/cinder/cinder.conf：

        [DEFAULT]
        rootwrap_config = /etc/cinder/rootwrap.conf
        sql_connection = mysql://openstack:openstack@192.168.30.73/cinder
        api_paste_confg = /etc/cinder/api-paste.ini
        iscsi_helper = tgtadm
        volume_name_template = volume-%s
        volume_group = cinder-volumes
        verbose = True
        auth_strategy = keystone
        state_path = /var/lib/cinder
        volumes_dir = /var/lib/cinder/volumes

4. 同步数据库：

        sudo cinder-manage db sync

####创建VG
1. 没有独立的硬盘或分区按以下操作：

    sudo mkdir -p /opt/data/cinder
    cd /opt/data/cinder
    sudo truncate -s 2G vgfile
    sudo losetup -f --show vgfile

 如果正常，会输出以下内容：

        /dev/loop0

 接下来开始创建vg：

        sudo vgcreate cinder-volumes /dev/loop0

 如果正常，会输出以下内容：

        No physical volume label read from /dev/loop0
        Writing physical volume data to disk "/dev/loop0"
        Physical volume "/dev/loop0" successfully created
        Volume group "cinder-volumes" successfully created

2. 有独立的硬盘或分区按以下操作(**以/dev/sda7****为例，以下操作有数据丢失风险，请谨慎操作**)：

        pvcreate /dev/sda7
        vgcreate cinder-volumes /dev/sda7
        lvcreate -L 1T -n instance-lv data-vg
        mkfs.ext4 /dev/data-vg/instance-lv
        echo "/dev/data-vg/instance-lv        /var/lib/nova/instances ext4    defaults        0 0" >> /etc/fstab
        mount -a

 重启cinder所有相关服务：

        sudo /etc/init.d/cinder-api restart
        sudo /etc/init.d/cinder-scheduler restart
        sudo /etc/init.d/cinder-volume restart

---        
###Nova配置
1. 更新配置文件/etc/nova/api-paste.ini（最后几行）：

        [filter:authtoken]
        paste.filter_factory = keystone.middleware.auth_token:filter_factory
        auth_host = 192.168.30.73
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = nova
        admin_password = service_password
        signing_dir = /tmp/keystone-signing-nova

2. 更新配置文件/etc/nova/nova.conf

        [DEFAULT]
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
        
        # AUTHENTICATION
        auth_strategy=keystone
        
        # SCHEDULER
        scheduler_driver=nova.scheduler.multi.MultiScheduler
        compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
        
        # CINDER
        volume_api_class=nova.volume.cinder.API
        
        # DATABASE
        sql_connection=mysql://openstack:openstack@192.168.30.73/nova
        
        # COMPUTE
        # kvm or QUME
        libvirt_type=kvm  
        libvirt_use_virtio_for_bridges=True
        start_guests_on_host_boot=True
        resume_guests_state_on_host_boot=True
        api_paste_config=/etc/nova/api-paste.ini
        allow_admin_api=True
        use_deprecated_auth=False
        nova_url=http://192.168.30.73:8774/v1.1/
        root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
        
        # APIS
        ec2_host=192.168.30.73
        ec2_url=http://192.168.30.73:8773/services/Cloud
        keystone_ec2_url=http://192.168.30.73:5000/v2.0/ec2tokens
        s3_host=192.168.30.73
        cc_host=192.168.30.73
        metadata_host=192.168.30.73
        enabled_apis=ec2,osapi_compute,metadata
        
        # RABBITMQ
        rabbit_host=192.168.30.73
        
        # GLANCE
        image_service=nova.image.glance.GlanceImageService
        glance_api_servers=192.168.30.73:9292
        
        # NETWORK
        network_manager=nova.network.manager.FlatDHCPManager
        force_dhcp_release=True
        dhcpbridge_flagfile=/etc/nova/nova.conf
        dhcpbridge=/usr/bin/nova-dhcpbridge
        firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
        public_interface=eth0    
        flat_interface=eth1     
        flat_network_bridge=br100
        fixed_range=10.10.0.0/24   
        network_size=256
        flat_injected=False
        connection_type=libvirt
        multi_host=True
        
        # NOVNC CONSOLE
        novnc_enabled=True
        novncproxy_base_url=http://192.168.30.73:6080/vnc_auto.html
        vncserver_proxyclient_address=192.168.30.73
        vncserver_listen=192.168.30.73

3. 修改配置文件/etc/sudoers，在最后一行追加：

        nova ALL=(ALL) NOPASSWD:ALL

4. 同步数据库

        sudo nova-manage db sync

5. 重启nova相关服务

        sudo /etc/init.d/nova-api restart
        sudo /etc/init.d/nova-cert restart
        sudo /etc/init.d/nova-consoleauth restart
        sudo /etc/init.d/nova-novncproxy restart
        sudo /etc/init.d/nova-scheduler restart

6. 检查服务状态

        sudo nova-manage service list

 如果正常，会输出类似以下信息：

 <a href="http://agenge.com/wp-content/uploads/2013/06/03.png"><img class="alignnone size-full wp-image-704" alt="03" src="http://agenge.com/wp-content/uploads/2013/06/03.png" width="974" height="62" /></a>

7. 创建fixed_ip（内网虚拟机IP）

        sudo nova-manage network create private --fixed_range_v4=10.10.0.0/24 --num_networks=1 --bridge=br100 --bridge_interface=eth2 --network_size=256 --multi_host=T

8. 创建floating_ip（可以理解为floating_rang，并非一个具体的IP地址），它和公网一个段的IP:

        nova-manage floating create 192.168.30.1/24

9. 设置安全策略

        nova secgroup-add-rule default tcp 1 65535 0.0.0.0/0
        nova secgroup-add-rule default udp 1 65535 0.0.0.0/0
        nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
        
---
##计算节点安装
###安装OpenStack组件

    os_nova="nova-common python-nova python-novaclient nova-compute nova-network nova-api-metadata"
    os_other="kvm libvirt-bin pm-utils bridge-utils"
    sudo apt-get install -y $os_nova $os_other
    
大约71M左右。

###Nova配置
1. 更新配置文件/etc/nova/nova.conf

        [DEFAULT]
        logdir=/var/log/nova
        state_path=/var/lib/nova
        lock_path=/run/lock/nova
        verbose=False
        api_paste_config=/etc/nova/api-paste.ini
        scheduler_driver=nova.scheduler.simple.SimpleScheduler
        s3_host=192.168.30.73
        ec2_host=192.168.30.73
        ec2_dmz_host=192.168.30.73
        rabbit_host=192.168.30.73
        cc_host=192.168.30.73
        metadata_host=10.0.0.7
        nova_url=http://192.168.30.73:8774/v1.1/
        sql_connection=mysql://openstack:openstack@192.168.30.73/nova
        ec2_url=http://192.168.30.73:8773/services/Cloud
        root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
        
        # Auth
        use_deprecated_auth=false
        auth_strategy=keystone
        keystone_ec2_url=http://192.168.30.73:5000/v2.0/ec2tokens
        # Imaging service
        glance_api_servers=192.168.30.73:9292
        image_service=nova.image.glance.GlanceImageService
        
        # NOVNC CONSOLE
        novnc_enabled=True
        novncproxy_base_url=http://192.168.30.73:6080/vnc_auto.html
        vncserver_proxyclient_address=192.168.30.74
        vncserver_listen=192.168.30.74
        
        # Network settings
        network_manager=nova.network.manager.FlatDHCPManager
        dhcpbridge_flagfile=/etc/nova/nova.conf
        dhcpbridge=/usr/bin/nova-dhcpbridge
        firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
        public_interface=eth0
        flat_interface=eth1
        flat_network_bridge=br100
        fixed_range=10.10.0.1/24
        floating_range=192.168.30.128/24
        network_size=250
        flat_network_dhcp_start=10.0.0.40
        flat_injected=false
        force_dhcp_release=true
        dhcp_lease_time=604800
        multi_host=true
        iscsi_helper=tgtadm
        connection_type=libvirt
        # Compute #
        compute_driver=libvirt.LibvirtDriver
        
        # Cinder #
        volume_api_class=nova.volume.cinder.API
        osapi_volume_listen_port=5900
            
2. 修改配置文件/etc/nova/nova-compute.conf

        [DEFAULT]
        libvirt_type=kvm

 除了KVM，你还可以修改成其他的，例如QEMU/XEN等。
 
3. 禁用默认的网桥

        virsh net-autostart default --disable
        virsh net-destroy default

4. 重启nova相关服务

        sudo /etc/init.d/nova-api-metadata restart
        sudo /etc/init.d/nova-compute restart
        sudo /etc/init.d/nova-network restart

---    
##访问OpenStack DashBoard
在控制节点重启httpd和memcached：

    sudo /etc/init.d/apache2 restart
    sudo /etc/init.d/memcached restart

访问 [http://192.168.30.73/horizon](http://192.168.30.73/horizon)，用户名和密码分别是admin/admin_password

---
##日常管理
###控制节点管理
####创建实例
#####创建虚拟机密钥对
在控制节点执行以下语句：

        root@control-01:~# ssh-keygen 
        Generating public/private rsa key pair.
        Enter file in which to save the key (/root/.ssh/id_rsa): 
        Created directory '/root/.ssh'.
        Enter passphrase (empty for no passphrase): 
        Enter same passphrase again: 
        Your identification has been saved in /root/.ssh/id_rsa.
        Your public key has been saved in /root/.ssh/id_rsa.pub.
        The key fingerprint is:
        6b:6e:2f:b5:b5:73:a6:8e:96:bf:c0:a5:84:7c:c5:92 root@control-01
        The key's randomart image is:
        +--[ RSA 2048]----+
        |                 |
        |           o     |
        |          E o    |
        |       . . o     |
        |        S o .    |
        |         =.o.    |
        |        o.++ .   |
        |       oo +oo o  |
        |       ..+oo=*   |
        +-----------------+
        
#####导入密钥

        nova keypair-add --pub_key .ssh/id_rsa.pub key2
        nova keypair-list
        +------+-------------------------------------------------+
        | Name | Fingerprint                                     |
        +------+-------------------------------------------------+
        | key2 | 6b:6e:2f:b5:b5:73:a6:8e:96:bf:c0:a5:84:7c:c5:92 |
        +------+-------------------------------------------------+

#####查看镜像

        nova image-list
        +--------------------------------------+--------------+--------+--------+
        | ID                                   | Name         | Status | Server |
        +--------------------------------------+--------------+--------+--------+
        | 533457ef-a1fe-41e0-a351-908975d3550b   | myFirstImage   | ACTIVE |        |
        +--------------------------------------+--------------+--------+--------+

镜像的创建方法请见上传镜像文件。

#####查看虚拟机配置模板

        nova flavor-list

#####创建实例(虚拟机)

        nova boot --flavor 1 --image 533457ef-a1fe-41e0-a351-908975d3550b --key_name key2 myFirstvm

创建一个名叫：myFirstvm的实例。

* --image             表示使用的哪个镜像文件
* --flavor             使用哪个配置模板，具体见查看虚拟机配置模板
* --key_name       指定密钥文件



在创建之后会输出以下类似信息：

<a href="http://agenge.com/wp-content/uploads/2013/06/04.png"><img class="alignnone size-full wp-image-705" alt="04" src="http://agenge.com/wp-content/uploads/2013/06/04.png" width="996" height="454" /></a>

从图中可看出正在创建一个实例。


####配置网络
#####创建与绑定公网
1. 创建一个公网IP，在控制节点执行以下语句：

        nova  floating-ip-create

2. 创建指定的公网IP

 floating-ip-create只是创建一个随机可用（IP池）的可用公网IP，那如何要创建一个指定的IP如何操作？没关系，nova可以做到，请看操作：

        nova add-floating-ip test01 192.168.30.6

3. 绑定公网IP地址到虚拟机

        nova add-floating-ip vm01 192.168.30.1

 表示绑定到vm01这个虚拟机。可通过查看虚拟机状态看到：

        nova show vm01

#####防火墙设置
1. 允许SSH登录

        nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

2. 允许ICMP Ping

        nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0

3. 查看防火墙设置

        nova secgroup-list-rules default

#####删除或解绑floating-ip
如果某个公网IP不再需要，你可以通过删除或解绑它：

        nova remove-floating-ip test01 192.168.30.6
或

        nova floating-ip-delete test01 192.168.30.6
删除之后就会从IP池释放出来，以供其他Tenant使用。

---
###Cinder管理
1. 列出当前用户所有资源

        cinder absolute-limits

2. 创建一个Volume

        cinder create --display_name cin01 10

 * --display_name为volume名称，后面的数字表示为10G。

3. 查看Volume

        cinder list

 或者

        nova volume-list

---
###计算节点管理
####KVM驱动
要启用KVM，只需要在/etc/nova/nova.conf加入：

        libvirt_type=kvm
        compute_driver=libvirt.LibvirtDriver

KVM支持的虚拟机镜像格式：RAW、QCOW2、VMDK


####QEMU驱动
要启用QEMU，只需要在/etc/nova/nova.conf加入：

        libvirt_type=qemu
        compute_driver=libvirt.LibvirtDriver

QEMU支持的虚拟机镜像格式：RAW、QCOW2、VMDK



####Xen驱动
要启用Xen，包括Xen, XenAPI, XenServer and XCP，只需要在/etc/nova/nova.conf加入：

        compute_driver=xenapi.XenAPIDriver
        xenapi_connection_url=http://your_xenapi_management_ip_address
        xenapi_connection_username=root
        xenapi_connection_password=your_password

XenAPI支持的虚拟机镜像格式：RAW、VHD

如果您的Xen使用的libvirt管理，只需要在/etc/nova/nova.conf加入：

        libvirt_type=xen
        compute_driver=libvirt.LibvirtDriver

---
###Glance管理
####磁盘格式
* raw 此格式支持兼容转换其他格式文件，类似于中间格式，最大的缺点是**不支持虚拟机的快照功能**，使用空间类似于物理磁盘，使用多少空间实际上就是多少空间，可以通过du -h file.raw 来查看它真正的大小，而ls -lh看到的就是整个raw文件大小（包括剩余空间）

* qcow2 更小的存储空间，du -h 和ls -h看到的大小一样。且支持虚拟机的快照功能，性能接近于 raw 格式的文件。支持AES加密、zlib压缩。


* vhd 微软的虚拟磁盘文件，不过也兼容其他虚拟软件。

* vmdk VMware的专有格式，其在自身的虚拟环境中，无论从性能还是功能，都是最强大、最稳定的，其他虚拟化软件较少用到。

* iso 经典的ISO文件，不多介绍。

* vdi VirtualBox的专有格式，一般服务器虚拟化也很少用到。

* aki Amazon内核镜像文件。

* ari Amazon 内存（ramdisk）镜像文件。

* ami Amazon机器镜像文件。


####容器格式
Glance的container format主要包括以下格式：

* bare                  表示镜像没有包括container或元数据，如果不知道选哪个格式，就选bare吧。

* ovf                   开放式虚拟磁盘文件，最初由VMware发起，支持多种虚拟化。

* aki                    Amazon内核镜像文件，在 磁盘格式设置为 aki 时使用。

* ari                    Amazon 内存（ramdisk）镜像文件，在 磁盘格式设置为 ari 时使用。

* ami                   Amazon机器镜像文件，在 磁盘格式设置为 ami 时使用。


关于安装与部署，偶暂时写到这里，这只是很简单的一个入门例子而已，下次偶会介绍
如何制作CentOS5.5、CentOS 6.3及Windows Server 2008的镜像，很多同学因镜像而烦恼。
