---
layout: post
title: "Centos 6.X 安装KVM与安装实例"
date: 2013-11-15 21:58
comments: true
categories: 
- 云计算
- OpenStack
- 虚拟化
tags:
- OpenStack
- 虚拟化
- KVM

---

## 前言
虚拟化已经不是一个新鲜话题，不过仍然看到很多新手连KVM都没有入门，甚至是先接触OpenStack才遇到KVM，所以今天给各位新手提供一个安装教程。

## 安装KVM依赖包

目前绝大部分的CPU都支持Intel VT或AMD-V ，此处检查步骤略过。

```
yum install -y bridge-utils kvm kmod-kvm qemu kvm-qemu-img virt-viewer virt-manager libvirt libvirt-python python-virtinst
```

创建一个叫 br0 的网桥：
```
cd /etc/sysconfig/network-scripts/
cp ifcfg-eth0 ifcfg-br0
vi ifcfg-eth0 
```
增加：
```
BRIDGE=br0
```

将ifcfg-br0 修改为：
```
DEVICE="br0"
TYPE="Bridge"
```

然后重启网络：
```
/etc/init.d/network restart
ifconfig br0
/etc/init.d/libvirtd restart
```

经过上面的简单安装与设置之后，KVM就已经安装完成，但怎么通过命令行创建虚拟机呢？

## 创建虚拟机
<!-- more -->
创建一个磁盘文件
```
qemu-img create -f qcow2 /data/kvm_data/centos5.img 8g
```

创建一个新的虚拟机
```
# -n 虚拟机名称  
# --vcpus 分配虚拟机CPU个数
# -r 分配内存大小 默认为MB
# -f 指定虚拟机的磁盘文件路径
# -s 磁盘文件固定大小，例如-s 10 立即扩展到10G
# -c 镜像文件位置
# --vnc --vncport=590X --vnclisten=0.0.0.0  通过VNC连接安装操作系统
# --os-type linux 安装一个linux虚拟机
# --network=bridge:br0  网络连接方式：使用一个叫br0的网桥 
virt-install --connect qemu:///system -n test01 --vcpus=1 \
  -r 1024 --virt-type=kvm -f /data/kvm_data/centos5.img -s 10 \
  -c /data/iso/CentOS-5.5-x86_64-bin-DVD-1of2.iso \
  --vnc --vncport=5903 --vnclisten=0.0.0.0 --os-type linux --accelerate \
  --network=bridge:br0 
```

自动安装CentOS 6.3
由于我在公司内部搭建了个无人值守服务，所幸可以直接安装使用：
```
virt-install --connect qemu:///system -n test03 --vcpus=1 \
  -r 2048 --virt-type=kvm -f /data/kvm_data/test03.img -s 20 \
  --vnc --vncport=5902 --vnclisten=0.0.0.0 \
  --os-type linux --accelerate \
  --network=bridge:br0 \
  -x ks=ftp://192.168.30.33/pub/ks/64/centos6.3/ks.cfg \
  --location ftp://192.168.30.33/pub/centos/6.3/x86_64/
```  

查看所有虚拟机(包括已经关闭的虚拟机)
```
virsh list --all
virsh destroy ID/NAME  关闭虚拟机
virsh start NAME   启动虚拟机
virsh console ID/NAME  控制台连接虚拟机
```

安装虚拟机之后如何连接呢？ 有两种选择：VNC和Console，由于偶工作时不太喜欢图形界面，还是命令行吧
实现控制台连接需要在虚拟机做一些配置，如下：

- 修改etc/grub.conf 添加“console=ttyS0
```
title CentOS (2.6.18-128.1.10.el5)
    root (hd0,0)
    kernel /boot/vmlinuz-2.6.18-128.1.10.el5 ro root=LABEL=/ console=ttyS0
    initrd /boot/initrd-2.6.18-128.1.10.el5.img
```
添加ttyS0到/etc/securetty

```
echo ttyS0 >> /etc/securetty
```

- 修改/etc/inittab添加下面这行：
```
S0:12345:respawn:/sbin/agetty ttyS0 115200
```

重启系统之后，使用语法为：
```
virsh console vm_name
```

你还可以任何时刻退出控制台，快捷键为：ctrl+]

今天就暂时分享这些内容，下次有机会将分享一个关于Xen相关的教程。
