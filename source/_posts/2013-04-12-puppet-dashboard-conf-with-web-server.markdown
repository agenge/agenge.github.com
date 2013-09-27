---
layout: post
title: Puppet Dashboard配置及apache和Nginx整合
categories:
- 运维
tags:
- Dashboard
- Puppet
- 资源管理
published: true
comments: true

---
Dashboard一个基于Ruby on Rails设计的Web控制台，便于管理Puppet Master和Agent，整个安装过程比较简单，以下是具体的安装步骤。
##安装前提条件
1. ruby版本>1.8.6，如果你是CentOS 5或是RHEL 5，可能你需要升级Ruby，官方下载地址请

 点击[这里](ftp://ftp.ruby-lang.org/pub/ruby)。具体的升级步骤由于下载的安装包已经包含，故不再介绍，本文示例基于Ruby 1.8.7。

 ```
 # ruby -v
 ruby 1.8.7 (2011-06-30 patchlevel 352) [x86_64-linux]
 ```

2. 安装相关依赖包
```
 # yum install -y mysql mysql-devel mysql-server \
     ruby ruby-devel ruby-irb ruby-mysql \
     ruby-rdoc ruby-ri
```
 如果你已经有Mysql，此处不需要安装mysql-server包。安装后加入开机自启动并启动mysql 。
```
# chkconfig mysqld on
# /etc/init.d/mysqld start
Starting mysqld:                                           [  OK  ]
```

3. 升级GEM软件包安装器

 首先检查gem版本，如果已经是1.3.5，就不需要升级：
 ```
    # gem -v
    1.3.5
 ```
如果是1.3.1，意味着你要升级gem：
```
# wget http://production.cf.rubygems.org/rubygems/rubygems-1.3.5.tgz
# tar zxvf rubygems-1.3.5.tgz
# cd rubygems-1.3.5
# ruby setup.rb
```

4. 安装rake
```
# gem install rake
```
<!--more-->


##数据库配置
```
# mysql -uroot -p
Enter password:

mysql> use dashboard
Database changed
mysql> grant all on dashboard.* to dashboard@localhost identified by 'dashboard';
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

```

##安装Puppet Dashboard
* 下载Dashboard 
```
# cd /usr/local/
# wget http://downloads.puppetlabs.com/dashboard/puppet-dashboard-1.2.20.tar.gz
# tar -zxvf puppet-dashboard-1.2.20.tar.gz
# cd puppet-dashboard-1.2.20
```
创建数据库用户,此处我们使用源码方式进行安装,你可以通过yum或apt-get安装。
```
# cp config/database.yml.example config/database.yml
```
* 修改数据库配置：
```
# vim config/database.yml
production:
database: dashboard
username: dashboard
password: dashboard
encoding: utf8
adapter: mysql
```
production代表生产环境，如果你是开发环境(development)或测试环境(test)，请修改相应的配置。

* 初始化数据库
```
# rake RAILS_ENV=production db:create
# gem install mysql
# rake RAILS_ENV=production db:migrate
```
* 修改时区
默认是UTC时间，如果你在控制台看到的时区不对，那需要修改时区：
```
# vim config/environment.rb
```
将config.time_zone = 'UTC'  修改为 config.time_zone = 'Chongqing'

* 启动Webrick Web服务器
```
# cd /usr/local/puppet-dashboard-1.2.20
# script/server -p 3000 -e production  -d
```
上面即代表启动dashboard，端口3000，以及后台(-d)运行,通过浏览器输入： http://dashboard_server:3000   即可访问。

###Puppet Dashboard配置
####服务器配置
在Puppet 服务器端/etc/puppet/puppet.conf中的[main]节加入：
```
reports = http, store
reporturl = http://192.168.30.226:3000/reports/upload
```
部署lib文件：
```
# cp /usr/local/puppet-dashboard/ext/puppet/puppet_dashboard.rb /usr/lib/ruby/site_ruby/1.8/puppet/reports/
```
保存之后，重启puppetmaster即可。
```
# /etc/init.d/puppetmaster restart
```
####客户端配置
在Puppet Agent（客户端）/etc/puppet/puppet.conf中的[main]节加入：
```
report = true
```
保存之后，重启puppet即可。
```
# /etc/init.d/puppet restart
```
####控制台配置
登录Dashboard后，需要自己添加： group和node


##Puppet Dashboard常用操作
** 导入日志 **
```
# cd /usr/local/puppet-dashboard
# rake RAILS_ENV=production reports:import
```
** 优化数据库 **
当数据量过大时，需要优化mysql数据库：
```
# cd /usr/local/puppet-dashboard
# rake RAILS_ENV=production db:raw:optimize
```

** 删除日志 **
删除1个月之前的数据：
```
# cd /usr/local/puppet-dashboard
# rake RAILS_ENV=production reports:prune upto=1 unit=mon
```
删除15天前的日志：
```
# rake RAILS_ENV=production reports:prune upto=15 unit=day
```

** 备份数据库 **
```
# rake RAILS_ENV=production db:raw:dump
mysqldump --add-locks --create-options --disable-keys --extended-insert --quick --set-charset --user=dashboard --password=dashboard dashboard > production.sql.tmp
mv production.sql.tmp production.sql

```
**恢复数据库 **
```
# rake RAILS_ENV=production FILE=production.sql db:raw:restore
```

** Dashboard 启停脚本 **
```
#!/bin/bash
# Description: Puppet Dashboard init.d script

# Get function from functions library
. /etc/init.d/functions

# Start the service Puppet Dashboard
start() {
    echo -n "Starting Puppet Dashboard: "
    /usr/bin/ruby /usr/local/puppet-dashboard/script/server -e production -b 192.168.30.226 -d
    ### Create the lock file ###
    touch /var/lock/subsys/puppetdb
    success $"Puppet Dashboard startup"
    echo
}

# Restart the service Puppet Dashboard
stop() {
    echo -n "Stopping Puppet Dashboard: "
    kill -9 `ps ax | grep "/usr/bin/ruby /usr/local/puppet-dashboard/script/server" | grep -v grep | awk '{ print $1 }' `
    ### Now, delete the lock file ###
    rm -f /var/lock/subsys/puppetdb
    success $"Puppet Dashboard shutdown"
    echo
}

### main logic ###
case "$1" in
    start)
        start
    ;;
    stop)
        stop
    ;;
    status)
        status Puppet DB
    ;;
    restart|reload|condrestart)
        stop
        start
    ;;
    *)
    echo $"Usage: $0 {start|stop|restart|reload|status}"
    exit 1
esac

exit 0

```

##使用Passenger运行Puppet Dashboard
相信用过Dashboard的人都知道在访问时控制台时比较缓慢，这正是由于内置的Webrick服务器导致，不过Puppet支持与其他第三方Web Server整合，以下分别介绍整合过程。

###Apache整合Puppet Dashboard
1. 安装依赖包
```
# yum install ruby ruby-libs ruby-devel  http httpd
```
2. 安装Passenger
```
# gem install passenger
```
3. 安装Apache的passenger模块
```
# passenger-install-apache2-module
```
如果安装没问题的话，就会有以下类似信息输出：

 ```
 The Apache 2 module was successfully installed.

 Please edit your Apache configuration file, and add these lines:

    LoadModule passenger_module /usr/lib64/ruby/gems/1.8/gems/passenger-3.0.19/ext/apache2/mod_passenger.so
    PassengerRoot /usr/lib64/ruby/gems/1.8/gems/passenger-3.0.19
    PassengerRuby /usr/bin/ruby

 After you restart Apache, you are ready to deploy any number of Ruby on Rails
 applications on Apache, without any further Ruby on Rails-specific
 configuration!

 Press ENTER to continue.

 ```
直接输入回车即可。

4. 配置Apache虚拟主机

 ```
 # cd /usr/local/puppet-dashboard
 # cp ext/passenger/dashboard-vhost.conf /etc/httpd/conf.d/
 # vi /etc/httpd/conf.d/dashboard-vhost.conf
 # UPDATE THESE PATHS TO SUIT YOUR ENVIRONMENT
 LoadModule passenger_module /usr/lib64/ruby/gems/1.8/gems/passenger-3.0.19/ext/apache2/mod_passenger.so
 PassengerRoot /usr/lib64/ruby/gems/1.8/gems/passenger-3.0.19
 PassengerRuby /usr/bin/ruby
 
 # you may want to tune these settings
 PassengerHighPerformance on
 PassengerMaxPoolSize 12
 PassengerPoolIdleTime 1500
 # PassengerMaxRequests 1000
 PassengerStatThrottleRate 120
 RailsAutoDetect On
 
 <VirtualHost *:80>
         ServerName pmaster.i.12582.com
         DocumentRoot /usr/local/puppet-dashboard/public/
         <Directory /usr/local/puppet-dashboard/public/>
                 Options None
                 Order allow,deny
                 allow from all
         </Directory>
   ErrorLog /var/log/httpd/dashboard.example.com_error.log
   LogLevel warn
   CustomLog /var/log/httpd/dashboard.example.com_access.log combined
   ServerSignature On
 </VirtualHost>
```
5. 重启httpd
```
# /etc/init.d/httpd restart
```
 此时再访问 http://pmaster.i.12582.com 发现速度较之前快太多。


###Nginx整合Puppet Dashboard
1. 安装依赖包
```
# yum install ruby ruby-libs ruby-devel  http httpd
```
2. 安装Passenger
```
# gem install passenger
```
3. 安装Nginx及passenger模块
```
# passenger-install-nginx-module
```
回车后，会有两个选择：
 * 表示下载nginx，编译且安装，推荐这种方式安装.
 * 表示自定义安装Nginx,安装之后，nginx默认存放在/opt目录.

 修改/opt/nginx/conf/nginx.conf，在最后一个}之前添加以下内容：

 ```
    server {
      listen 80;
      server_name pmaster.i.12582.com;
      root /usr/local/puppet-dashboard/public;
      passenger_enabled on;
    }
 ```

4. 重启Nginx
```
# killall nginx
# /opt/nginx/sbin/nginx
```

最后，你要是有任何疑问，请留言。
