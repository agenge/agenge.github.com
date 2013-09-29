---
layout: post
title: Puppet用户手册--安装与配置
categories:
- 运维
tags:
- Puppet
- 文件同步
- 资源管理
- 运维
published: true
comments: true

---

##1. 安装Puppet依赖环境
在安装Puppet之前需要保证服务器已经安装以下软件：

1. Ruby: 挂载ISO后，切换到/mnt/Server, 输入： yum  -y install ruby-*
2. Facter
3. Ruby所依赖的OpenSSL 库，运行以下命令来测试是否已经安装所依赖的库文件：
```
 ruby -ropenssl -e "puts :yep"
``` 
 如果输出“yep”表示无问题。

___注意：在安装Puppet之前，如果需要，必须将服务器主机名修改好，否则将会出现很多问题哟！___

##2. 安装Puppet服务器端

由于本文基于源码安装，使用二进制包的安装方式本文不打算介绍，如有需要请自行到网上搜索。好了，首先将facter-2.0.0rc4.tar.gz、puppet-2.7.19.tar.gz上传到 Puppet服务端(192.168.56.2)，具体步骤如下

* 安装facter：
```
# wget http://downloads.puppetlabs.com/facter/facter-2.0.0rc4.tar.gz
# tar zxvf facter-2.0.0rc4.tar.gz
# cd facter-2.0.0rc4
# ruby install.rb
```

 安装后会提示是否有无问题，如下图所示，无任何错误：
 {% img /images/2012/12/1.png %}

* 安装Puppet:
```
# wget http://puppetlabs.com/downloads/puppet/puppet-2.7.20.tar.gz
# tar zxvf puppet-2.7.19.tar.gz
# cd puppet-2.7.19
# ruby install.rb
```
 
 安装后会提示是否有无问题，如下图所示，无任何错误：
 {% img /images/2012/12/2.png %}

 到此，Puppet服务端安装已经结束。

<!-- more -->

##3. 安装Puppet客户端 For Linux

Puppet客户端的安装方式与服务端一样，故不再详细介绍，详细请见第二章。

##4. 安装Puppet客户端 For Windows
暂时不打算写！

##5. 配置Puppet服务端和客户端
在配置之前，要确保Puppet服务端和所有Puppet客户端的本地时间一致，关于时间同步，推荐使用NTP(请参考网上的NTP详细介绍或MAN)。

###5.1 配置Puppet服务端：
创建puppet组和用户：
```
# groupadd puppet
# useradd -g puppet -s /sbin/nologin puppet
```

设置/etc/hosts：
```
# echo "192.168.56.2 puppetmaster.test.com puppetmaster" >> /etc/hosts
# echo "192.168.56.10 client1.test.com client1" >> /etc/hosts
# cp conf/namespaceauth.conf /etc/puppet/
# cp conf/redhat/puppet.conf /etc/puppet/
# cp conf/redhat/server.init /etc/init.d/puppetmaster
# chmod +x /etc/init.d/puppetmaster
```

启动puppetmaster ，若无问题，显示如下：
```
# /etc/init.d/puppetmaster start
Starting puppetmaster:                    [ OK ]
# chkconfig puppetmaster on
```

###5.2  配置Puppet客户端：
除了以下命令不一样，其他全部和服务端一样的配置：
```
# cp conf/redhat/client.init /etc/init.d/puppet
# chmod +x /etc/init.d/puppet
```

服务端对应命令如下：
```
# cp conf/redhat/server.init /etc/init.d/puppetmaster
```

启动puppet，若无问题，显示如下：
```
# /etc/init.d/puppet start
Starting puppet:               [ OK ]
# chkconfig puppet on
```

###5.3 Puppet服务端和客户端测试：
Puppet客户端与服务器端是通过SSL隧道通信的，客户端安装完成后，需要向服务器端申请证书：

* 首次连接服务器端会发起证书申请，在客户端执行命令如下
```
 puppetd --server puppetmaster --test
```

 {% img /images/2012/12/3.png %}

 执行以上命令代表客户端已经成功生成证书，并把证书签名请求发送到Puppet服务端.

* 登录到Puppet服务端，查看所有客户端的证书签名请求：

 {% img /images/2012/12/4.png %} 

 从结果可看出，已经看到client1.test.com客户端的证书签名请求，最后对所有的证书请求进行签名：
```
# puppetca -s -a
```
 
 {% img /images/2012/12/5.png %} 

###5.4 示例：同步hosts文件(modules实现)
* 同步之前，先看下未使用module实现的简单示例：

```
# 默认的节点配置
node default {
     file {
             "/tmp/temp1.txt":
             content => "Hello,Puppet!"; }
}

# 同步/root/temp01.txt文件
node "client1.test.com" {
     host { "host1":
     ip => "192.168.1.1",
     target => "/root/temp01.txt",
     ensure => present; }
}
node "feinno-hgg" {
     host { "host2":
     ip => "192.168.1.1",
     target => "/root/temp01.txt",
     ensure => present; }
}
```
以上示例表示配置两个客户端，分别是“client1.test.com”和“feinno-hgg”，同时还有一个默认的node节点，后期如果有新客户端加入，在文件末尾加入一个新的node即可。但请再深入的思考一下，从后期的维护或管理角度来看，必须考虑下面两个重点的问题：

 * 问题1：假设您管理的不是10台，而是100台或者1000台，甚至更多，此种配置方式是否最优？
 * 问题2：假设您管理的客户端是多架构平台的，例如Debian、CentOS、Solaris、AIX或

 Windows，如何解决跨平台的文件同步？

 使用示例1的配置方式，显示无法解决上面这两个问题，所以这就是我们下面要介绍的模块化实现。

* 创建所须目录：
```
#/bin/mkdir -p /etc/puppet/modules/hosts/{files,lib,manifests,templates}
#/bin/mkdir -p /etc/puppet/modules/hosts/files/hosts/etc
```

3. 创建所须文件：
```
# ll /etc/puppet/manifests/
total 8
-rw-r--r-- 1 root root 103 Oct 15 17:08 nodes.pp
-rw-r--r-- 1 root root 52 Oct 15 17:04 site.pp
```

```
# cat site.pp
# 导入nodes.pp
import 'nodes.pp'
$puppetserver="master.test.com"
```

```
# cat nodes.pp
# 匹配所有以字母开头、.test.com结尾的客户端，并包含一个 hosts 的类文件
node /^\w+\.test\.com$/  {    
     include hosts
}
node 'feinno-hgg'  {
     include hosts
}
```

nodes.pp中使用了正则表达式，这使得一个非常 简单的node可以匹配多个客户端，而对于特殊主机来讲，可单独在文件结尾添加node即可，从而解决我们刚才提到的第一个问题。再来看看modules下的2类文件：

```
# ls -l /etc/puppet/modules/hosts/manifests
-rw-r--r-- 1 root root 654 Oct 15 17:16 config.pp
-rw-r--r-- 1 root root 47 Oct 12 11:20 init.pp
```

查看init.pp
```
 #cat init.pp
 # 定义一个hosts的类，并包括它的一个子类：hosts::config
 class hosts {
     include hosts::config
 }
```

查看config.pp
```
# cat config.pp
# 定义一个hosts::config的子类，父类为 hosts
class hosts::config inherits hosts {
    if $operatingsystem in [ "RedHat","CentOS","Ubuntu","Fedora" ] {

            file { "/etc/hosts":
                owner => "root",
                group => "root",
                mode => 644,
                source => "puppet://$puppetserver/hosts/etc/hosts",
            }
    }  elsif $operatingsystem == "Windows" {
            file { "C:/Windows/System32/drivers/etc/hosts":
                owner => "xm_Administrator",
                group => "Administrators",
                source => "puppet://$puppetserver/hosts/etc/win_hosts";
            }
    } else {
        fail("Doesn't support this OS: $operatingsystem")
    }
}
```
从config.pp中可看出，通过一个if/elsif决断语句，结合Puppet内置变量$operatingsystem可同步任何跨平台的操作系统的hosts文件，从而解决我们刚才提到的第二个问题。

* 配置modules

 细心的读者可能会问：Puppet如何识别的module？ 问的相当好，Puppet默认是无法识别自定义modules的，需要我们在puppet.conf的 [main]中配置一个参数：modulepath，例如：
```
# cat puppet.conf | grep module
modulepath = /etc/puppet/modules
```
 通过简单的参加配置，Puppet就可识别自定义模块。

* 配置 Fileserver：
```
# /etc/puppet/fileserver.conf
[hosts]
        path /etc/puppet/modules/hosts/files
        allow *
```
此处配置一个hosts的文件服务，及指定文件存放位置，同时允许所有客户端同步，生产环境不建议直接对所有客户端开放。而我们之前在config.pp中的source后面的路径格式为：

```
puppet://$puppetserver/mount_point/path
```

例如：

```
puppet://$puppetserver/hosts/etc/hosts
```
对应的绝对路径为：/etc/puppet/modules/hosts/files/etc/hosts

6. Puppet客户端 for Windows 8客户端测试：

```
D:\Program Files (x86)\Puppet Labs\Puppet\bin>puppet.bat agent --test --server master.test.com
info: Retrieving plugin
info: Caching catalog for feinno-hgg
info: Applying configuration version '1350292129'
notice: /Stage[main]/Hosts::Config/File[C:/Windows/System32/drivers/etc/hosts]/content:
 
info: FileBucket got a duplicate file {md5}2c0dd3682bc4dbab317365a88af6177a
info: /Stage[main]/Hosts::Config/File[C:/Windows/System32/drivers/etc/hosts]: Filebucketed C:/Windows/System32/drivers/etc/hosts to puppet with sum 2c0dd3682bc4dbab317365a88af6177a
notice: /Stage[main]/Hosts::Config/File[C:/Windows/System32/drivers/etc/hosts]/content: content changed '{md5}2c0dd3682bc4dbab317365a88af6177a' to '{md5}09636a06eea3999e8b02ca831923f3d6'
notice: this OS: windows.  Sync complete.
notice: /Stage[main]/Hosts::Config/Notify[this OS: windows.  Sync complete.]/message: defined 'message' as 'this OS: windows.  Sync complete.'
notice: Finished catalog run in 0.85 seconds
```
通过日志可看出，已经成功同步win_host到C:/Windows/System32/drivers/etc/hosts 。

___提示：在MS 2008或Win 7、Win 8客户端同步时，注意C:/Windows/System32/drivers/etc的目录需要有写权限。___

##6. 管理Puppet服务
###6.1 Puppet服务端启动、停止、重启过程如下：

```
# /etc/init.d/puppetmaster start # 启动
# /etc/init.d/puppetmaster stop # 停止
# /etc/init.d/puppetmaster restart # 重启
```

###6.2 Puppet客户端启动、停止、重启过程如下：

```
# /etc/init.d/puppet start
# /etc/init.d/puppet stop
# /etc/init.d/puppet restart
```
###6.3 Puppet For Windows客户端设置：

在Windows下安装Puppet客户端后，会自动在系统服务中添加一个Puppet Agent服务，并已经设置为开机自启动，如果不是，请自行修改。

###6.4 设置Puppet服务端开机自启动：
```
# chkconfig puppetmaster on
# chkconfig --list puppetmaster
```

##7. 常见错误
###7.1 语法错误 puppetd -s puppetmaster.test.com –t
```
/usr/lib/ruby/site_ruby/1.8/puppet/application/agent.rb:54:in `handle_serve': uninitialized constant Puppet::Network::Handler (NameError)
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:363:in `send'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:363:in `parse_options'
    from /usr/lib/ruby/1.8/optparse.rb:1247:in `call'
    from /usr/lib/ruby/1.8/optparse.rb:1247:in `order!'
    from /usr/lib/ruby/1.8/optparse.rb:1205:in `catch'
    from /usr/lib/ruby/1.8/optparse.rb:1205:in `order!'
    from /usr/lib/ruby/1.8/optparse.rb:1279:in `permute!'
    from /usr/lib/ruby/1.8/optparse.rb:1300:in `parse!'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:370:in `parse_options'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:305:in `run'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:416:in `hook'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:305:in `run'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:407:in `exit_on_fail'
    from /usr/lib/ruby/site_ruby/1.8/puppet/application.rb:305:in `run'
    from /usr/sbin/puppetd:4
```
答：此问题由于版本太低导致，或修改参数形式，例如将-t  -s 修改成 --test --server。


###7.2  Could not load openssl Ruby library; cannot install

```
# tar xvf openssl-0.9.8p.tar.gz
# cd openssl-0.9.8p
# cp ssl/ssl.h /root/ruby-1.8.7/ext/openssl/
# ./config -fPIC --prefix=/usr/local --openssldir=/usr/local/openssl enable-shared
# make && make install
```

进入到ruby源码目录下的ext/openssl/：

```
# cd /root/ruby-1.8.7/ext/openssl/
# ruby extconf.rb --with-openssl-include=/usr/local/openssl/include --with-openssl-lib=/usr/local/openssl/lib 
# make && make install
```
###7.3   Error 400 on SERVER: Cannot find file: Invalid mount

```
[root@client1 puppet]# puppetd --test
info: Caching catalog for client1.test.com
info: Applying configuration version '1349933729'
err: /Stage[main]//Node[client1.test.com]/File[/etc/puppet/files/temp01.txt]: Could not evaluate: Error 400 on SERVER: Cannot find file: Invalid mount 'temp01.txt' Could not retrieve file metadata for puppet:///temp01.txt: Error 400 on SERVER: Cannot find file: Invalid mount 'temp01.txt' at /etc/puppet/manifests/site.pp:20
notice: Finished catalog run in 0.45 seconds
```

###7.4 err: Could not retrieve catalog from remote server: Connection refused

```
[root@client1 puppet]# puppetd --test --server master.test.com
err: Could not retrieve catalog from remote server: Connection refused - connect(2)
warning: Not using cache on failed catalog
err: Could not retrieve catalog; skipping run
err: Could not send report: Connection refused - connect(2)
```
该错误表示无法连接到Master，即Puppet服务端，由以下两个原因造成：

1. Puppet服务端拒绝连接，通常是由于防火墙或selinux造成；

2. Puppet客户端本地防火墙导致

本人在测试时遇到此问题，但经检查上述原因后，仍然报拒绝连接错误，经过约2小时的仔细排查，终于找到问题，原因却让我差点吐血， 以下是排查过程：

  * 第一步：查看Puppet服务端防火墙，发现是开启的，但无任何规则，关闭防火墙后，到客户端尝试，无果! 再查看Linux，问题仍在。
  * 第二步：查看Puppet客户端防火墙，已是关闭状态，SELinux同样关闭状态。
  * 第三步：尝试修改Puppet服务端的auth.conf文件，并重启puppetmaster后，问题仍在。
  * 第四步：查看Puppet客户端所连接的服务端是否正确：
  
```
# puppet agent --configprint server
```

打印的域名和hosts文件一致。

  * 第五步：查看Puppet服务端和Puppet客户端的hosts文件是否正确

 在第一次查看这两个文件的时候，发现没问题，后来再从其他地方折腾回来看hosts文件的时候，

 发现Puppet客户端的hosts文件存在一个让本人都不好意思写出来的错误：
 
```
 # Do not remove the following line, or various programs
 # that require network functionality will fail.
 127.0.0.1 master.test.com master localhost.localdomain localhost
 #::1 localhost6.localdomain6 localhost6
  
 192.168.56.2 master.test.com
 192.168.56.10 client1.test.com
 192.168.10.188 feinno-hgg
```

终于发现问题所在，客户端的hosts写成这样，也确实不易，保存退出后，经测试问题已解决。这种低级错误都是平常不够严谨才造成的，且还花大量时间，所以当出现问题后，一定要非常仔细检查每个操作或步骤，只有这样，才能更高效的找到根源并解决。

###7.5 Could not evaluate: Error 400 on SERVER: Not authorized to call find
```
info: Caching catalog for client1.test.com
info: Applying configuration version '1350282880'
notice: this OS: RedHat.  Sync complete.
notice: /Stage[main]/Hosts::Config/Notify[this OS: RedHat.  Sync complete.]/message: defined 'message' as 'this OS: RedHat.  Sync complete.'
err: /Stage[main]/Hosts::Config/File[/etc/hosts]: Could not evaluate: Error 400 on SERVER: Not authorized to call find on /file_metadata/hosts/etc/hosts Could not retrieve file metadata for puppet://master.test.com/hosts/etc/hosts: Error 400 on SERVER: Not authorized to call find on /file_metadata/hosts/etc/hosts at /etc/puppet/modules/hosts/manifests/config.pp:10
```
该问题由于没有通过Puppet服务端fileserver的认证造成，由于要同步文件资源，故需要在Puppet服务端Fileserver对应的mount point加上认证，例如：

```
# cat /etc/puppet/fileserver.conf
[hosts]
        path /etc/puppet/modules/hosts/files
        allow client1.test.com
```

###7.6 Failed to generate additional resources using 'eval_generate: SSL_connect
```
D:\Program Files (x86)\Puppet Labs\Puppet\bin>puppet.bat agent --test --server master.test.com
info: Retrieving plugin
err: /File[C:/ProgramData/PuppetLabs/puppet/var/lib]: Failed to generate additional resources using 'eval_generate: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=master.test.com]
err: /File[C:/ProgramData/PuppetLabs/puppet/var/lib]: Could not evaluate: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=master.test.com] Could not retrieve file metadata for puppet://master.test.com/plugins: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=master.test.com]

err: Could not retrieve catalog from remote server: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=master.test.com]
warning: Not using cache on failed catalog
err: Could not retrieve catalog; skipping run
err: Could not send report: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=master.test.com]
```

此问题由于Puppet服务端和Puppet客户端的时间不一致导致，将服务端和客户端的时间修改成一致即可解决，如果确保时间一致，问题仍在，请重新生成证书，重新生成证书过程如下：

1. 在Puppet服务端清除证书：
```
# puppetca -c feinno-hgg
```
2. 在Puppet客户端删除证书：

 Linux客户端证书路径：

  * MS2003：%ALLUSERSPROFILE%\Application Data\PuppetLabs\puppet\etc\ssl

  * MS2008：%PROGRAMDATA%\PuppetLabs\puppet\etc\ssl\
  
 例如删除Linux客户端下的证书：

```
# rm -fr /var/lib/puppet/ssl/*
```

3. 在Puppet客户端执行：

```
# puppetca --test --server master.test.com
```

4. 在Puppet客户端执行：

```
# puppetca –l
```
如果有显示客户端的认证请求签名，则输入：

```
# puppetca -s feinno-hgg    #意思为客户端feinno-hgg的证书请求进行签名
```

或：

```
# puppetca -s -a # 对所有的客户端进行证书签名
```

###7.7 Could not retrieve information from environment production source(s)

```
D:\Program Files (x8 6)\Puppet Labs\Puppet\bin>puppet.bat agent --test --server master.test.com
info: Caching certificate for feinno-hgg
info: Retrieving plugin
info: Caching certificate_revocation_list for ca
err: /File[C:/ProgramData/PuppetLabs/puppet/var/lib]: Could not evaluate: Could
not retrieve information from environment production source(s) puppet://master.t
est.com/plugins
info: Caching catalog for feinno-hgg
info: Applying configuration version '1350284808'
info: Creating state file C:/ProgramData/PuppetLabs/puppet/var/state/state.yaml
notice: Finished catalog run in 0.18 seconds
```
此问题是属于2.7.x 的一个Bug导致，即在Windows平台下Puppet客户端向Puppet服务端同步文件时，默认情况下会同步Puppet服务端的plug，那要怎么解决呢？操作如下：

在modules_name的目录下创建一个 lib的空目录即可。例如：

```
# mkdir /etc/puppet/modules/mymodule/lib
```

也可参考以下官方地址：[https://projects.puppetlabs.com/issues/2244](https://projects.puppetlabs.com/issues/2244)

###7.8 notice: Run of Puppet configuration client already in progress; skipping

解决方法： 部分情况下puppet服务会无法启动，且会提示puppet已经启动，这个时候需要删除一个文件puppetdlock：

* Linux客户端绝对路径： /var/lib/puppet/state/puppetdlock
* Windows 2003客户端绝对路径：C:\Documents and Settings\All Users\Application Data\PuppetLabs\puppet\var\state\ puppetdlock
* Windows 2008客户端绝对路径：C:\ ProgramData\PuppetLabs\puppet\var\state\ puppetdlock

###7.9 certificate verify failed: [CRL is not yet valid for /CN

```
err: /File[C:/ProgramData/PuppetLabs/puppet/var/lib]: Failed to generate additional resources using 'eval_generate: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=pmaster.i.12582.com]
err: /File[C:/ProgramData/PuppetLabs/puppet/var/lib]: Could not evaluate: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=pmaster.i.12582.com] Could not retrieve file metadata for puppet://pmaster.i.12582.com/plugins: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=pmaster.i.12582.com]
err: Could not retrieve catalog from remote server: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=pmaster.i.12582.com]
warning: Not using cache on failed catalog
err: Could not retrieve catalog; skipping run
err: Could not send report: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed: [CRL is not yet valid for /CN=pmaster.i.12582.com]
```
原因：

在配置证书时可能由于CA证书不匹配或被删除导致。

解决办法：

Master执行：

```
# puppet cert clean c02.i.12582.com
```

客户端执行：

```
# rm -f C:/ProgramData/PuppetLabs/puppet/etc/ssl/certs/c02.i.12582.com.pem
# puppetca –test
```

此处需要注意一定别删除整个SSL目录，只需要删除指定的pem文件即可，另上面请根据实际情况修改。

###7.10 charset: utf8, collation: utf8_unicode_ci

详细错误信息：

```
Couldn't create database for {"encoding"=>"utf8", "adapter"=>"mysql", "username"=>"dashboard", "database"=>"dashboard", "password"=>"dashboard"}, charset: utf8, collation: utf8_unicode_ci (if you set the charset manually, make sure you have a matching collation)
```

解决方法：

```
[root@pmaster puppet-dashboard-1.2.20]# ln -s /var/lib/mysql/mysql.sock /tmp/mysql.sock
```

###7.11 undefined method `more_results' for #<Mysql>

解决方法：注释mysql_adapter.rb中的318和       642

```
@connection.more_results && @connection.next_result    # invoking stored procedures with CLIENT_MULTI_RESULTS requires this
 to tidy up else connection will be dropped
```

###7.12 undefined method `requirement' for #<Rails: 

解决方法：修改gem_dependency.rb
注释掉81行 

```
#gem self.name, self.requirement # <  1.8 unhappy way 
```

修改为：

```
gem self.name, :version => (self.respond_to?(:requirement) ? self.requirement : self.version_requirements)
```

##8. 排错思路
* 服务器端、客户端之间和本身的防火墙确认无问题；

* 服务器端的SELinux确认禁用；

* 证书确认是否正确配置正确，重新配置的过程如下：

 Puppet服务端：
 ```
 # puppetd -r -c client01.domain.com
 ```
 
Puppet客户端：

* Windows 2003：C:\Documents and Settings\All Users\Application Data\PuppetLabs\puppet\etc\ssl
* Windows 2008：C:\ProgramData\PuppetLabs\puppet\etc\ssl
* Linux 客户端: /var/lib/puppet/ssl
 