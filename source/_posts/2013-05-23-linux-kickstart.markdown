---
layout: post
title: 批量装机---Linux无人值守之Kickstart
categories:
- 运维
tags:
- Kickstart
- 批量装机
- 无人值守
published: true
comments: true

---

前段时间总是忙于安装系统，对于偶这种懒人来说，一天安装两遍都无法承受，别说更多，基本上有两个原因造出：

 * 时间很宝贵。
 * 偶比较懒惰。
  
一直习惯于 RedHat 家族产品，从最开始接触的RHEL，到后来同时使用的Fedora，及目前使用最多的CentOS，这无疑是偶最为熟悉的Linux，为啥偶要提这些捏？因为以下操作是基于这些Linux操作的啦，费话在此打住，以下为详细操作步骤.

##安装基础包
_注：以下Yum操作是基于偶本地的ftp 仓库源，--enablerepo是告诉yum使用这个指定的安装源，
默认情况下，CentOS自带的仓库源也是可以用滴，优先推荐各位使用哦，有个前提条件：连网！
What？服务器不能上网？公司不许随意连公网？能说脏话吗？Fuck! _
```
yum -y install dhcp-* --enablerepo=centos5.5
yum -y install tftp-* --enablerepo=centos5.5
yum -y install vsftpd-* --enablerepo=centos5.5
cp /usr/share/doc/dhcp-3.0.5/dhcpd.conf.sample /etc/dhcpd.conf
```
##配置dhcp

<!-- more -->


    vi /etc/dhcpd.conf

添加以下信息：
```
filename "pxelinux.0";     # 指定bootloader文件
next-server 192.168.0.20;  # 指定索取pxelinux.0的tftp服务器IP
```
添加的这两行可在大括号外面，也可在里面，next-server选项可写可不写，但建议各位写上啦。
```
service dhcpd start   # 启动服务
cd /tftpboot
cp /mnt/isolinux/* ./
```
实际需要的是vmlinuz，initrd.img  *.msg 这几个文件，但为了操作方便，偶直接把isolinux目录下的文件全cp过来（偶在文章开头就说过偶比较懒惰，换成生产环境千万别这样玩）。

```
mkdir pxelinux.cfg
mv isolinux.cfg pxelinux.cfg/default
cp /usr/lib/syslinux/pxelinux.0 /tftpboot
```
default配置文件的作用是告诉主机从哪里去加载操作系统内核，并将启动加载文件拷到/tftpboot下。修改tftp参数并启动tftp服务
```
vi /etc/xinetd.d/tftp
# *********************************
service tftp
{
    socket_type = dgram
    protocol = udp
    wait = yes
    user = root
    server = /usr/sbin/in.tftpd
    server_args = -s /tftpboot
    disable = <span style="color: #ff0000;"><strong>no</strong></span>
    per_source = 11
    cps = 100 2
    flags = IPv4
}
# *********************************
```
tftpboot 这个参数主要是指定tftp client 客户端从服务器的哪个目录去加载bootloader的pxelinux.0文件。启动服务：
```
service xinetd restart
chkconfig tftp on
vi /tftpboot/pxelinux.cfg/default
```
修改第3行，第12行.
```
default linux
prompt 1
timeout 10 //时间调小点
display boot.msg
F1 boot.msg
F2 options.msg
F3 general.msg
F4 param.msg
F5 rescue.msg
label linux
kernel vmlinuz
append ks=ftp://192.168.0.20/pub/ks.cfg initrd=initrd.img
label text
```

##安装Kickstart
```
yum install -y *kickstart* --enablerepo=centos5.5
system-config-kickstart
```
_提示：所有以system-config开头的命令，都需要图形界面的支持。这不是必须的，前提是对ks的配置文件语法很熟悉啦。 _

###配置ks.cfg
首先将 ks.cfg 保存到 /var/ftp/pub 目录下，将修改相应权限：

    chmod 707 /var/ftp/pub/ks.cfg
以下是偶使用的ks.cfg全文 ，请各位根据自己情况修改：

```
#platform=x86, AMD64, or Intel EM64T
# System authorization information
auth --useshadow --enablemd5
# System bootloader configuration
bootloader --location=mbr
# Partition clearing information
clearpart --all --initlabel
# Use graphical install
#graphical
text
# Firewall configuration
firewall --disabled
# Run the Setup Agent on first boot
firstboot --disable
# System keyboard
keyboard us
# System language
lang en_US
# Installation logging level
logging --level=info
# Use network installation
url --url=<strong>ftp://192.168.30.210/pub/x86_64/centos5.5</strong>
# Network information
network --bootproto=dhcp --device=eth0 --onboot=on
# Reboot after installation
reboot
#Root password
rootpw --iscrypted $1$R79JLo34$.Yi4OUmL5PhpsxzSTL1hX1

# SELinux configuration
selinux --disabled
# System timezone
timezone Asia/Chongqing
# Install OS instead of upgrade
install
# X Window System configuration information
xconfig --defaultdesktop=GNOME --depth=8 --resolution=1026x768 --startxonboot
# Disk partitioning information
part /boot --bytes-per-inode=4096 --fstype="ext3" --size=256
part swap --bytes-per-inode=4096 --fstype="swap" --size=2048
#part / --bytes-per-inode=4096 --fstype="ext3" --grow --size=1
part / --bytes-per-inode=4096 --fstype="ext3" --grow --size=102400

%packages --resolvedeps
#@system-tools
#@gnome-desktop

@base-x

#@sound-and-video
#@chinese-support
#@graphical-internet
#@admin-tools
#@editors
#key --skip

```

所有以井号(#)开头的为注释行.

* url是操作系统的镜像地址
* part / (未注释行)是指根分区分100G, 如果你的磁盘很大，剩余的空间可在安装系统后根据您的具体需求而设。
* @bash-x  安装最基础的系统包
* resolvedeps 将自动解决包之间的依赖关系。

###启动所有服务
```
service dhcpd restart
service xinetd restart
service vsftpd restart
```
##PXE安装系统
当服务器一切工作准备就绪，就开始大规模安装Linux系统吧，由于各个主板对应的BIOS设置不同，此处无法满足所有的需求，通常做法是：

进入BIOS -> PXE boot -> enable -> save and reboot

如果各位有任何疑问，请留言回复。

