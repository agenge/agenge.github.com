---
layout: post
title: 安全漏洞整改解决方案
categories:
- 运维
tags:
- OpenSSH
- SSH
- 安全漏洞
published: true
comments: true

---

注意：以下所有操作須确认后再实施：

* OpenSSH 相关漏洞  

解决方案

升级OpenSSH为最新版本,目前为5.9,首先到官网下载:[openssh-5.9p1.tar.gz](http://www.openssh.com/portable.html#http)

把OpenSSH 上传到服务器，检查升级前版本(以下所有操作均在root下完成)：

```
shell> ssh -V # 此命令会显示OpenSSL、OpenSSH的詳細版本号
```

首先安装OpenSSH:

```
shell> tar xvf openssh-5.9p1.tar.gz
shell> cd openssh-5.9p1
shell> sed -i -e &#8216;s/_5.9//&#8217; version.h
```

查询信赖包是否安装：

```
shell> rpm -qa | grep zlib #如果能看到zlib、zlib-devel，请继续，否则安装后再继续。
shell> ./configure &#8211;sysconfdir=/etc/ssh
shell> make && make install
```

开始配置：

```
shell> /bin/cp /usr/local/sbin/sshd /etc/init.d/sshd
shell> mkdir /root/ssh_bak  #创建备份目录
shell> mv /etc/ssh/* /root/ssh_bak/  #移动到备份目录
shell> /bin/cp /usr/local/etc/* /etc/ssh/
shell> sed -e &#8216;s@/usr/bin/ssh-keygen.*@#@&#8217; /etc/init.d/sshd
shell> /etc/init.d/sshd restart
shell> ssh -V 检查是否是 OpenSSHp1 开头，如果是，则OpenSSH升级成功。
```

*  远端WEB服务器上存在/robots.txt文件 
  
解决方案： 可直接删除（可参考：[http://zh.wikipedia.org/wiki/Robots.txt](http://zh.wikipedia.org/wiki/Robots.txt "robots.txt")

* ICMP timestamp请求响应漏洞 
 
解决方案：

```
shell> echo 0 > /proc/sys/net/ipv4/icmp_echo_ignore_all
shell> echo 0  >> /etc/rc.local
```

Windows Server 2008 参考: [http://hi.baidu.com/%BA%D3%C4%CF%CD%F8%C2%B7/blog/item/91076a62831cdb4aebf8f807.html](http://hi.baidu.com/%BA%D3%C4%CF%CD%F8%C2%B7/blog/item/91076a62831cdb4aebf8f807.html)
Windows Server 2003参考: [http://zhidao.baidu.com/question/41992099](http://zhidao.baidu.com/question/41992099)

* Apache Tomcat 相关漏洞 

解决方案：

根据安全厂商给的解决方案链接：[http://www.ocert.org/advisories/ocert-2011-003.html](http://www.ocert.org/advisories/ocert-2011-003.html) 从此页面可得知，有问题的Tomcat版本如下：<= 5.5.34, <= 6.0.34, <= 7.0.22，而无安全漏洞的 Tomcat版本如下： 5.5.35, >= 6.0.35, >= 7.0.23

访问：[Tomcat](http://tomcat.apache.org/index.html "Apache Tomcat"), 下载对应的Tomcat版本，例如经分在使用 Tomcat 5.5.34，下载对应的Tomcat 5.5.35,如在使用Tomcat 6.0.34，下载对应的 Tomcat 6.0.35，依此类推。

* Apache Tomcat sendfile请求安全限制绕过和拒绝服务漏洞：此漏洞也是通过上面的版本升级方式解决。具体请参考官方解释：
 
[http://tomcat.apache.org/security-5.html#Fixed_in_Apache_Tomcat_5.5.35](http://tomcat.apache.org/security-5.html#Fixed_in_Apache_Tomcat_5.5.35) 和 [http://secunia.com/advisories/45232/](http://secunia.com/advisories/45232/)

<!-- more -->


* SNMP服务存在可读口令

解决方案：

可按照漏洞扫描結果中的过程操作，如问困难，可向系统组同事请教。

* rpc相关漏洞 


解决方案：

(和项目组确认没有使用NFS后再操作)

```
shell> /etc/init.d/portmap stop && chkconfig portmap off
shell> /etc/init.d/rpcidmapd stop  &&  chkconfig rpcidmapd off
shell> /etc/init.d/nfslock stop && chkconfig nfslock off
```

* 利用SMTP/EXPN命令可猜测目标主机上的用户名 <span style="color: #ff0000;">解决方案（待确认）：

* Oracle数据库服务器CREATE ANY DIRECTORY权限提升漏洞  <span style="color: #ff0000;">暂时没有解决方案。

* SNMP服务可以通过SNMPV1访问  <span style="color: #ff0000;"> 解决方案（待确认）

* Apache HTTP Server相关漏洞 

解决方案：

通过下载使用Apache HTTP Server 2.2.22或以上可解决，详情参考：[http://mail-archives.apache.org/mod_mbox/httpd-announce/201201.mbox/browser](http://mail-archives.apache.org/mod_mbox/httpd-announce/201201.mbox/browser)

此页面显示了2.2.22或以上这个版本修复了哪些安全漏洞。官方下载地址：[http://www.apache.org/dyn/closer.cgi](http://www.apache.org/dyn/closer.cgi)

10.1 Apache Apache::Status和Apache2::Status模块跨站脚本漏洞

[http://mail-archives.apache.org/mod_mbox/perl-advocacy/200904.mbox/%3Cad28918e0904011458h273a71d4x408f1ed286c9dfbc@mail.gmail.com%3E](http://mail-archives.apache.org/mod_mbox/perl-advocacy/200904.mbox/%3Cad28918e0904011458h273a71d4x408f1ed286c9dfbc@mail.gmail.com%3E)

10.2  Apache服务器不完整HTTP请求拒绝服务漏洞[精确扫描]:
更改httpd.conf中 TimeOut 的值为30秒

11.远端WWW服务支持TRACE请求 解决方案 请参考第10点。

12.Oracle tnslsnr没有设置口令  解决方案：
扫描报告已经明确写出详细步骤，或DBA自己完成。

13. 猜测出远程FTP服务存在可登录的用户名口令  解决方案：
确认账号的密码复杂性，例如检查是否存在123456 类似这样的密码，如有简单密码，确认后再修改。

14.目标主机showmount -e信息泄露 解决方案：
确认是否有NFS服务在运行，如在运行确认是否关闭，如因有业务影响请在整改报告中明确写出原因（但绝不能外网访问NFS）。

15.检测到远端rlogin服务正在运行中  解决方案：
如果是AIX系统，请在整改报告中明确写出原因（但绝不能外网登录）。

16.远端运行着IDENT服务  解决方案：
漏洞扫描报告中已经有详细的操作步骤。

17.检测到远端rsh服务正在运行中   解决方案：
如果是AIX系统，请在整改报告中明确写出原因（但绝不能外网登录）。

18.检测到远端rexec服务正在运行中  解决方案：
漏洞扫描报告中已经有详细的操作步骤。

19.存在一个可用的远程代理服务器  <span style="color: #ff0000;">解决方案待确认。

20.远程web主机存在目录遍历漏洞  <span style="color: #ff0000;"> 解决方案待确认。

21. 远程主机允许匿名FTP登录  解决方案：
修改配置文件，不许出现匿名登录，由于FTP的类型较多，具体的操作步骤可咨询系统组同事。

22.FTP服务器版本信息可被获取  <span style="color: #00ff00;">无需整改（因要修改源码重新编译）。

23.远端SSH Server允许使用低版本SSH协议  解决方案：
参考漏洞扫描报告中的操作步骤，或参考第1点直接升级OpenSSH版本（强烈建议）。

24.检测到远端XDMCP服务正在运行中  解决方案：
关闭XDMCP服务

25. PHP相关漏洞  解决方案：
根据[http://www.venustech.com.cn/NewsInfo/124/6459.Html](http://www.venustech.com.cn/NewsInfo/124/6459.Html) 可得知，受影响版本是：

 * PHP 5.2 <= 5.2.13
 * PHP 5.3 <= 5.3.2
 
目前最好的办法是升级PHP版本。官方最新稳定是：PHP 5.3.10。 而5.2.X 最高版本是：PHP 5.2.17

