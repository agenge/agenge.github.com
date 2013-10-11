---
layout: post
title: 制作OpenStack Windows Server 2008 镜像
categories:
- 云计算
- OpenStack
tags:
- OpenStack
- 镜像制作
published: true
comments: true

---
#前言
对于OpenStack中，制作一个Windows 的镜像让很多新手极为烦恼，偶有幸成功制作，
不敢私藏，故与大家分享小小心得。上次与大家介绍过 [制作OpenStack CentOS 6.3 镜像](http://agenge.com/cloud-computing/make_openstack-centos-image.html)

本次主要是记录如何制作一个Windows Server 2008的镜像，请看操作：

1. 下载Virtio总线驱动

 由于OpenStack只支持Virtio总线的磁盘，所以我们需要在安装之前下载virtio驱动：
    
        wget http://alt.fedoraproject.org/pub/alt/virtio-win/latest/images/virtio-win-0.1-59.iso
2. 准备一个Windows Server 2008的ISO文件。

 除了Virtio总线驱动，您还需要准备一个Windows Server 2008的ISO，为安装操作系统做准备。
3. 创建虚拟磁盘文件：

        qemu-img create -f qcow2 winserver2008.img 20G

 对于虚拟磁盘文件的各种磁盘格式区别与对比，可以参考 [qcow2、raw、vmdk等镜像格式](http://blog.prajnagarden.com/?p=248)

4. 创建虚拟机，使用kvm或virt-install均可，本次安装使用的virt-install

        virt-install --connect qemu:///system -n winserver2008 --vcpus=1 -r 2048 \
          --disk path=/path/to/winserver2008.img,size=60,format=qcow2,bus=virtio,cache=none \
          -c /path/iso/windows_server_2008.iso \
          --vnc --vncport=5909 --vnclisten=0.0.0.0  \
          --os-type windows --os-variant=win2k8 --accelerate \
          --network=bridge:br0,model=virtio  \
          --disk path=/path/to/win_driver/virtio-win-0.1-59.iso,device=cdrom,perms=ro<br />
       
 以上参数有点多，不过这里不一一解释，后期偶会专门写一篇介绍KVM相关知识点，这里只是描述几个重要的参数：
    - -n  虚拟机的名称
	- --disk  虚拟磁盘存放的路径，即第一步qemu-img创建的虚拟磁盘。</span></li>
	- -c  ISO的路径 </span></li>
	- --vncport  VNC连接端口，后面会用到，这里是5909，且必须是未使用的端口。</span></li>
	- --network   这个地方偶使用的是一个叫 br0 的网桥，所以你的系统必须保证有br0这个网桥。</span></li>
<!-- more -->  
 
 使用VNC 客户端连接，例如192.168.30.211:9 
 {% img /images/2013/07/01.png %}

 连接成功之后，和常规安装操作系统没有任何区别，但在分区时会提示找到磁盘文件，如图：

 {% img /images/2013/07/02.png %}

 点击“<b>加载驱动程序</b>”，并按下图选择对应的驱动：

 {% img /images/2013/07/03.png %}

 点击“确定”，如果WIN8驱动找不到磁盘，重新选择WIN7即可。然后再点击“下一步”：

 {% img /images/2013/07/04.png %}

 安装之后并关机，进入下一步操作。

5. 上传镜像

 最后一步就是要将刚才的虚拟磁盘文件上传到云平台中心:
 
        glance image-create --name Win-2008-x86_64-cloud --is-public true --container-format ovf --disk-format qcow2 < /home/agen/winserver2008.img
 通过以上的步骤，希望新手能够成功制作一份完美的镜像文件，如果在制作过程中有任何疑问，请留言，最好是邮件，偶在有时间的情况下肯定会第一时间解答。
