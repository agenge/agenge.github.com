---
layout: post
title: 制作OpenStack CentOS 6.3 镜像
categories:
- 云计算{cloud-computing}
tags:
- CentOS
- OpenStack
- 镜像制作
published: true
comments: true

---
# 前言
距离上次写文章已经N天，即对不住各位，更对不住自己，今天将教大家如何制作
OpenStack CentOS 6.3 镜像，费话不多说，咱就开始干活吧。

1. 准备一个镜像文件，即ISO文件，本示例所使用的为：

        CentOS-6.3-i386-minimal.iso

 *注：无论您是内部测试环境还是生产环境，偶都强烈推荐使用64位操作系统。*

2. 引导并安装系统

        virt-install -n CentOS63 -r 2048 --cpu host \
          -c /data/iso/CentOS-6.3-i386-minimal.iso \
          --disk path=/data/kvm_data/centos6.3-openstack.img,device=disk,bus=virtio,size=10,format=qcow2 \
          --vnc --vncport=5908 --vnclisten=0.0.0.0 -v \
          --network bridge=br0

 这些参数的就请您先自行查找手册，后期有时间偶会补上关于KVM的相关实践手册。

3. 镜像相关设置（此处根据你的需求进行个性化定制，尽量保持简洁）：
 * 分区全部给根分区，防火墙和SELinux禁止.

            chkconfig iptables off

 将/etc/selinux/config 中的 SELINUX=enforcing修改成 SELINUX=disabled
 * 修改分区表：

            vi /etc/fstab
            # 将UUID=这一行注释，加入：
            LABEL=cec-rootfs                           / ext4 defaults<b> 0 0</b>
 * 修改网络配置：

            vi /etc/sysconfig/network-scripts/ifcfg-eth0
            #将HWADDR=这行删除或注释即可，并且设置成BOOTPROTO=dhcp

 * 删除网络规则，因为Centos6之后70-persistent-net.rules这个文件会自动添加除eth0之外的其他网络接口，

            rm -rf /etc/udev/rules.d/70-persistent-net.rules
 * 还要把另外一个文件删除，不然上面这个文件还会自动创建滴：
            mv /lib/udev/write_net_rules{,.bak}
4. 上传镜像

        glance image-create --name centos63Image --is-public true --container-format ovf --disk-format qcow2 &lt; /home/agen/centos63.img

 至此，所有操作已经完成，如果一切正常，你所创建的实例访问正常。如网络还存在问题，可能
和您的OpenStack网络设置有关，不过这已经超出本文的范畴。假如您对OpenStack的环境搭建不是很熟悉的话，建议您再多看看这篇文章：<a title="Openstack安装与部署(Folsom)" href="http://agenge.com/cloud-computing/openstack_install_with_deploy_for_folsom.html">Openstack安装与部署(Folsom)</a>。
