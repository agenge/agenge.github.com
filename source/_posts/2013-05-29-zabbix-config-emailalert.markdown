---
layout: post
title: Zabbix配置与报警设置
categories:
- 监控
tags:
- zabbix
- 监控系统
- 运维
published: true
comments: true
---

上次偶写了一篇[Zabbix 源码安装与配置](http://agenge.com/blog/2013/05/02/zabbix-source-install-conf) ，安装之后默认是无法监控客户端主机滴，所以偶今天专门写篇文章来介绍Zabbix配置，主要涉及到：添加主机、邮件报警及一些常见的操作流程，希望对初学者有所帮助。
##监控机
通在安装Zabbix之后，还需要在Web管理控制台添加需要监控的主机，下列是以添加一台监控主机为例进行演示。

1. “Configuration” -> “Hosts”，点击添加一台主机“Host”:

 {% img /images/2013/05/01.jpg  %}

2. 需要填写的内容主要为：“主机(Host)”、“模板(Templates)”，详细操作请见下列图中所示：主机的设置如下：

 <a href="http://agenge.com/wp-content/uploads/2013/05/02.jpg"><img class="alignnone size-full wp-image-668" alt="02" src="http://agenge.com/wp-content/uploads/2013/05/02.jpg" width="1004" height="615" /></a>
 模板的设置如下：

 <a href="http://agenge.com/wp-content/uploads/2013/05/03.jpg"><img class="alignnone size-full wp-image-669" alt="03" src="http://agenge.com/wp-content/uploads/2013/05/03.jpg" width="630" height="221" /></a>

 点击“添加(Add)”之后，从弹出的子页面选择需要的模板，Zabbix默认已自带非常多的模板，

 例如Mysql、Zabbix Server、Zabbix Agentd、OS Linux等。

  <a href="http://agenge.com/wp-content/uploads/2013/05/04.jpg"><img class="alignnone size-full wp-image-670" alt="04" src="http://agenge.com/wp-content/uploads/2013/05/04.jpg" width="450" height="450" /></a>

 勾选所需要的监控模板后，点击最下边的“选择(Select)”，回到之前的模板标签页，最后一定要记得保存(Save)。

  <a href="http://agenge.com/wp-content/uploads/2013/05/05.jpg"><img class="alignnone size-full wp-image-671" alt="05" src="http://agenge.com/wp-content/uploads/2013/05/05.jpg" width="560" height="184" /></a>

3. 如果一切正常，默认在1分钟之后就会变成可用状态：
  <a href="http://agenge.com/wp-content/uploads/2013/05/06.jpg"><img class="alignnone size-full wp-image-672" alt="06" src="http://agenge.com/wp-content/uploads/2013/05/06.jpg" width="622" height="78" /></a>

<!-- more -->





##报警设置
在添加监控主机之后，还需要邮件报警设置，否则监控人员无法及时掌握系统状态，报警一般是邮件报警与短信报警，甚至相两者结合。 其中邮件报警又可分为自定义脚本警报和自己搭建邮件服务器进行报警，短信报警暂时不涉及，后面会有专门章节进行介绍，本章只涉及到自定义邮件报警的设置，使用一个Msmtp（兼容SendMail接口的SMTP客户端. 工具，关于它的原理介绍，请自行搜索。下面是详细的安装步骤：
###邮件报警设置
1. 安装
```
tar jxvf msmtp-1.4.31.tar.bz2
cd msmtp-1.4.31
./configure --prefix=/usr/local/msmtp
make && make install
mkdir /usr/local/msmtp/etc
touch /usr/local/msmtp/etc/msmtprc
```
2. 配置
```
vim /usr/local/msmtp/etc/msmtprc
########################################
defaults
account default
host smtp.139.com
port 25
from taurus52@139.com
auth login
tls off
user taurus52@139.com
password 123456
logfile /var/log/mmlog
########################################
```
 host/from/user/password这三个字段，请根据具体情况修改。

 安装Mutt，Mutt是一个邮件用户代理工作，其本身不发送和接收邮件，需要调用邮件传输代码(例如Msmtp或sendmail) 来发送和接收邮件.
 
    ```
    yum -y install mutt
    vim /etc/Muttrc
    set sendmail="/usr/local/msmtp/bin/msmtp"
    set use_from=yes
    set realname="taurus52@163.com"
    set editor="vi"
```
  * /etc/Muttrc为全局设置，如果只是对某个用户设置，可以在~/muttrc中设置。
  * sendmail设置为msmtp的绝对路径
  * realname 设置你的email地址
  * editor 设置为你的编辑器，如果你中emacs忠实用户，也是可以滴。

 测试是否能够发送邮件：
```
echo "this is a test mail" >> /tmp/files
echo "testmail" | mutt -s "test" -a /tmp/files  test_zabbix@gmail.com
```
  * -a 代表添加附件
  * “testmail” 邮件正文
  * -s “test” 邮件标题
  * test_zabbix@gmail.com 为接收的邮件人。

 登录邮件客户端察看确认收到邮件：

 <a href="http://agenge.com/wp-content/uploads/2013/05/07.jpg"><img class="alignnone size-full wp-image-673" alt="07" src="http://agenge.com/wp-content/uploads/2013/05/07.jpg" width="373" height="246" /></a>

 修改zabbix_server.conf以下内容：
```
 mkdir -p /data/zabbix/alertscripts
 vim zabbix_server.conf
 AlertScriptsPath=${datadir}/alertscripts
```
 创建一个监控脚本：
```
 vim /data/zabbix/alertscripts/zabbix_monitor
 #!/bin/bash
 $1 to mail address
 $2 mail subject
 $3 mail content
 echo "$3" | mutt -s "$2" $1
```
 修改权限：
```
 chown zabbix:zabbix /data/zabbix/alertscripts
```
 到目前为止，已经手动测试通过邮件报警的功能，接下来需要将它集成到zabbix监控系统中。

3. 点击“管理(Administration)”->“媒体类型(Media types)”，点击页面最右边的“添加媒体类型(Cretae media type)”:
<a href="http://agenge.com/wp-content/uploads/2013/05/08.jpg"><img class="alignnone size-full wp-image-674" alt="08" src="http://agenge.com/wp-content/uploads/2013/05/08.jpg" width="691" height="318" /></a>

 * Description： 此为描述，可随意填写。
 * Type： 选择 “Script’。
 * Script name：填写刚才创建的shell脚本(当然，也可以是其他脚本)名称，例如zabbix_monitor。
 * Enabled： 启用此脚本。
 
 最后记得“Save”哦。


####添加用户和组
1. 添加用户组

 点击“管理(Administration)”->“用户(Users)”，点击最右边的“创建用户组(Create user group)”:
 
 <a href="http://agenge.com/wp-content/uploads/2013/05/09.jpg"><img class="alignnone size-full wp-image-675" alt="09" src="http://agenge.com/wp-content/uploads/2013/05/09.jpg" width="422" height="196" /></a>

 用户组(User group)只需要填写“Group name”，其他默认即可。

 <a href="http://agenge.com/wp-content/uploads/2013/05/10.jpg"><img class="alignnone size-full wp-image-676" alt="10" src="http://agenge.com/wp-content/uploads/2013/05/10.jpg" width="871" height="355" /></a>

 权限(Permissions)需要添加之前新增的主机，点击“Read-write”下的“添加(Add)”，并在的子页面选择对应的主机组，例如选择“Puppet”：

 <a href="http://agenge.com/wp-content/uploads/2013/05/11.jpg"><img class="alignnone size-full wp-image-677" alt="11" src="http://agenge.com/wp-content/uploads/2013/05/11.jpg" width="621" height="457" /></a>

 点击“Select”确认。最后“保存(Save)”，即添加一个用户组(puppet)。

2. 添加用户

 添加一个用户组之后，此用户组中并没有任何一个用户，必须在此用户组中增加用户才行，在用户组列表页面，点击用户组“Puppet”之后的“Users(0)”：

  <a href="http://agenge.com/wp-content/uploads/2013/05/12.jpg"><img class="alignnone size-full wp-image-678" alt="12" src="http://agenge.com/wp-content/uploads/2013/05/12.jpg" width="611" height="371" /></a>

  在跳转后的“Users”页面最右边，点击“创建用户(Create user)”：

  <a href="http://agenge.com/wp-content/uploads/2013/05/13.jpg"><img class="alignnone size-full wp-image-679" alt="13" src="http://agenge.com/wp-content/uploads/2013/05/13.jpg" width="418" height="185" /></a>

  之后有三个标签需要设置

  * 用户标签：

  <a href="http://agenge.com/wp-content/uploads/2013/05/14.jpg"><img class="alignnone size-full wp-image-680" alt="14" src="http://agenge.com/wp-content/uploads/2013/05/14.jpg" width="679" height="578" /></a>
  
  * 媒体标签：

  <a href="http://agenge.com/wp-content/uploads/2013/05/15.jpg"><img class="alignnone size-full wp-image-681" alt="15" src="http://agenge.com/wp-content/uploads/2013/05/15.jpg" width="885" height="520" /></a>

  * 权限标签(Permissions)：

  <a href="http://agenge.com/wp-content/uploads/2013/05/16.jpg"><img class="alignnone size-full wp-image-682" alt="16" src="http://agenge.com/wp-content/uploads/2013/05/16.jpg" width="595" height="618" /></a>

  此处就能看到之前在创建用户组时看到的权限，最后点击“保存(Save)”。

####触发器设置
添加邮件报警设置和用户设置之后，还需要配置触发器，表示当某个触发器达到设置的值后，就会报警。

Action标签：

<a href="http://agenge.com/wp-content/uploads/2013/05/17.jpg"><img class="alignnone size-full wp-image-683" alt="17" src="http://agenge.com/wp-content/uploads/2013/05/17.jpg" width="631" height="495" /></a>

条件(Conditions)标签：

<a href="http://agenge.com/wp-content/uploads/2013/05/18.jpg"><img class="alignnone size-full wp-image-684" alt="18" src="http://agenge.com/wp-content/uploads/2013/05/18.jpg" width="850" height="310" /></a>

<a href="http://agenge.com/wp-content/uploads/2013/05/19.jpg"><img class="alignnone size-full wp-image-685" alt="19" src="http://agenge.com/wp-content/uploads/2013/05/19.jpg" width="713" height="323" /></a>

操作(Operations)标签：

<a href="http://agenge.com/wp-content/uploads/2013/05/20.jpg"><img class="alignnone size-full wp-image-686" alt="20" src="http://agenge.com/wp-content/uploads/2013/05/20.jpg" width="807" height="651" /></a>

<a href="http://agenge.com/wp-content/uploads/2013/05/21.jpg"><img class="alignnone size-full wp-image-687" alt="21" src="http://agenge.com/wp-content/uploads/2013/05/21.jpg" width="874" height="207" /></a>

以上就是Zabbix的基本操作流程，如果你还有疑问，可回复讨论。
