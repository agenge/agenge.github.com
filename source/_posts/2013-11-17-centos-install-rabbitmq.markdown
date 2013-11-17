---
layout: post
title: "CentOS 6 安装RabbitMQ"
date: 2013-11-17 21:18
comments: true
categories: 
- OpenStack
- 运维
tags:
- 消息队列
- RabbitMQ

---

## 系统信息

- 操作系统:  CentOS 6.3 64位
- Erlang:    5.8.5 64位
- RabbitMQ:  3.1.5


## 安装依赖包

###启用并安装EPEL(Extra Packeages for Enterprise Linux)
```
wget http://mirror.neu.edu.cn/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
rpm -ivh epel-release-6-8.noarch.rpm
```

添加 erlang 仓库源
由于 RabbitMQ 是erlang编写，所以要依赖它

```
wget -O /etc/yum.repos.d/epel-erlang.repo http://repos.fedorapeople.org/repos/peter/erlang/epel-erlang.repo
```

###安装erlang
```
yum clear all
yum install -y erlang
```
然后测试下erlang是否成功：
```
erl -v
Erlang R14B04 (erts-5.8.5) [source] [64-bit] [rq:1] [async-threads:0] [kernel-poll:false]

Eshell V5.8.5  (abort with ^G)
```
从输出结果来看，已经安装成功，版本信息为：V5.8.5

---
##安装 RabbitMQ
把依赖包安装好，就可以进行rabbitmq的安装，包名为：rabbitmq-server:
```
yum -y install rabbitmq-server
```
<!-- more -->
###启动与设置rabbitmq
安装之后，我们就可以来启动rabbitmq server，启动方式有两种：
- 通过系统服务启动
```
service rabbitmq-server start
Starting rabbitmq-server: SUCCESS
rabbitmq-server.
```
- 直接运行启动脚本(推荐使用)
```
/etc/init.d/rabbitmq-server start
Starting rabbitmq-server: SUCCESS
rabbitmq-server.
```

除了在终端输出的信息外，还可以通过日志来查看更详细的信息，默认路径为：/var/log/rabbitmq/rabbit@localhost.log
```
=INFO REPORT==== 17-Nov-2013::20:44:29 ===
Starting RabbitMQ 3.1.5 on Erlang R14B04
Copyright (C) 2007-2013 GoPivotal, Inc.
Licensed under the MPL.  See http://www.rabbitmq.com/

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
node           : rabbit@localhost
home dir       : /var/lib/rabbitmq
config file(s) : (none)
cookie hash    : GA+C2KHxoRs776SK+Ki+qg==
log            : /var/log/rabbitmq/rabbit@localhost.log
sasl log       : /var/log/rabbitmq/rabbit@localhost-sasl.log
database dir   : /var/lib/rabbitmq/mnesia/rabbit@localhost

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
Limiting to approx 924 file handles (829 sockets)

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
Memory limit set to 398MB of 996MB total.

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
Disk free limit set to 1000MB

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
msg_store_transient: using rabbit_msg_store_ets_index to provide index

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
msg_store_persistent: using rabbit_msg_store_ets_index to provide index

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
started TCP Listener on [::]:5672

=INFO REPORT==== 17-Nov-2013::20:44:29 ===
Server startup complete; 0 plugins started.
```

除了这个日志，平常在维护及故障排除时，还可以查看其他更多的日志：
```
tree /var/log/rabbitmq/
/var/log/rabbitmq/
├── rabbit@localhost.log
├── rabbit@localhost-sasl.log
├── shutdown_err
├── shutdown_log
├── startup_err
└── startup_log

0 directories, 6 files
```

### 设置开机启动
RabbitMQ安装的默认配置是没有开机自启动的，如：
```
chkconfig --list rabbitmq-server
rabbitmq-server 0:off   1:off   2:off   3:off   4:off   5:off   6:off
```

设置下rabbitmq为on即可：
```
chkconfig --list rabbitmq-server
rabbitmq-server 0:off   1:off   2:on    3:on    4:on    5:on    6:off
```
或者直接指定启动级别：
```
chkconfig --level 35 rabbitmq-server on
chkconfig --list rabbitmq-server    
rabbitmq-server 0:off   1:off   2:off   3:on    4:off   5:on    6:off
```

另外，RabbitMQ默认还提供一个强大的工具：rabbitmqctl，例如查看RabbitMQ状态：
```
rabbitmqctl status
```
输出结果如下：
```
Status of node rabbit@localhost ...
[{pid,1961},
 {running_applications,[{rabbit,"RabbitMQ","3.1.5"},
                        {os_mon,"CPO  CXC 138 46","2.2.7"},
                        {xmerl,"XML parser","1.2.10"},
                        {mnesia,"MNESIA  CXC 138 12","4.5"},
                        {sasl,"SASL  CXC 138 11","2.1.10"},
                        {stdlib,"ERTS  CXC 138 10","1.17.5"},
                        {kernel,"ERTS  CXC 138 10","2.14.5"}]},
 {os,{unix,linux}},
 {erlang_version,"Erlang R14B04 (erts-5.8.5) [source] [64-bit] [rq:1] [async-threads:30] [kernel-poll:true]\n"},
 {memory,[{total,26976648},
          {connection_procs,2648},
          {queue_procs,5296},
          {plugins,0},
          {other_proc,9052880},
          {mnesia,56656},
          {mgmt_db,0},
          {msg_index,33664},
          {other_ets,761160},
          {binary,1872},
          {code,14419185},
          {atom,1354457},
          {other_system,1288830}]},
 {vm_memory_high_watermark,0.4},
 {vm_memory_limit,418077081},
 {disk_free_limit,1000000000},
 {disk_free,49057206272},
 {file_descriptors,[{total_limit,924},
                    {total_used,3},
                    {sockets_limit,829},
                    {sockets_used,1}]},
 {processes,[{limit,1048576},{used,122}]},
 {run_queue,0},
 {uptime,836}]
...done.
```



### 安装管理界面插件
除了命令行管理工具，还提供了一个WEB管理界面，该管理界面是以插件的方式存在，不过默认没有启用，要启用它也很简单：
```
/usr/lib/rabbitmq/bin/rabbitmq-plugins enable rabbitmq_management
```
如果启用成功，将会提示如下：
```
The following plugins have been enabled:
  mochiweb
  webmachine
  rabbitmq_web_dispatch
  amqp_client
  rabbitmq_management_agent
  rabbitmq_management
Plugin configuration has changed. Restart RabbitMQ for changes to take effect.
```
提示重启rabbitmq server:
```
/etc/init.d/rabbitmq-server restart
Restarting rabbitmq-server: SUCCESS
rabbitmq-server.
```

通过浏览器打开: http://192.168.1.108:55672/mgmt/ ，默认的用户名和密码都是guest.

RabbitMQ管理界面的截图：
{% img /images/2013/11/rabbitmq-mg.jpg %}

如果无法打开，可能会是以下几个问题造成的：

* RabbitMQ Server可能启动失败，进一步的排错可通过查看详细的日志。
* 防火墙阻挡，包括iptables和企业内部防火墙。
* SELinux开启，禁用它即可。


