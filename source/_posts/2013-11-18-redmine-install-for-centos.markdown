---
layout: post
title: "redmine-install-for-centos"
date: 2013-11-18 00:04
comments: true
categories: 
- 项目管理
tags:
- 项目管理
- redmine

---
Redmine安装与配置

## 介绍
redmine是一个开源的项目管理套件。而且能运行绝大部分平台：Linux、Uinx、Windows等平台。


## 安装Redmine

### 安装依赖包
安装redmine之前，必须安装ruby和gem，下载地址：

- [redmine 2.4.0](http://rubyforge.org/frs/?group_id=1850)
- [ruby 1.8.7](ftp://ftp.ruby-lang.org/pub/ruby/1.8/)
- [rubygems 1.8.24](https://rubyforge.org/frs/?group_id=126&release_id=46730)
- Mysql、Apache httpd

CentOS 6.3 默认安装的ruby已经是1.8.7，所在可以直接yum安装，如果你想源码进行安装，请自行Google：
```
wget https://rubyforge.org/frs/download.php/77242/redmine-2.4.0.tar.gz --no-check-certificate
wget https://rubyforge.org/frs/download.php/76073/rubygems-1.8.24.tgz --no-check-certificate
yum install -y ruby-libs ruby rdoc ruby-devel gcc-c++ curl-devel
```

### 安装rubygems
```
tar zxvf rubygems-1.8.24.tgz 
cd rubygems-1.8.24
ruby setup.rb 
```

为了与apache结合使用，需要创建apache用户与组；
```
groupadd apache
useradd -g apache -s /sbin/nologin apache
```
### 安装 redmine
安装 redmine
```
tar zxvf redmine-2.4.0.tar.gz 
chown -R apache:apache redmine-2.4.0
cd redmine-2.4.0
```

安装依赖包(使用淘宝的gem源)
```
gem sources --remove http://rubygems.org/
gem sources -a http://ruby.taobao.org
gem install bundler
bundle install --without development test postgresql sqlite rmagick    
```
without 表示后面的模块都不需要安装

### 安装Mysql
本示例为了简便，直接使用的yum安装，需要源码安装的见 [这里](http://agenge.com/blog/2013/08/09/drbd_heartbeat_mysql_ha/)
```
yum -y install mysql-server mysql-devel mysql
/etc/init.d/mysqld restart
/usr/bin/mysqladmin -u root password '123456'
```

创建redmine数据库
```
[root@localhost ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.1.69 Source distribution

Copyright (c) 2000, 2013, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database redmine character set utf8;
Query OK, 1 row affected (0.00 sec)

mysql> create user 'redmine'@'localhost' identified by 'redmine';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on redmine.* to 'redmine'@'%';
Query OK, 0 rows affected (0.00 sec)
```

测试是否可以用redmine登录；
```
[root@localhost ~]# mysql -uredmine -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.1.69 Source distribution

Copyright (c) 2000, 2013, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use redmine;
Database changed
```

### 配置redmine的数据库连接
```
cd redmine-2.4.0/config
cp database.yml.example database.yml
vi database.yml
```

修改后的内容如下：

```
production:
  adapter: mysql2
  database: redmine
  host: localhost
  username: redmine
  password: redmine
  encoding: utf8

```

修改redmine的配置文件：
```
cp configuration.yml.example configuration.yml
vi configuration.yml
```
修改default和production的内容如下，其他的值都保持不变或根据你自己的需求而变：

```
default:
  # Outgoing emails configuration (see examples above)
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
      address: 127.0.0.1
      port: 25
      domain: localhost
      authentication: :login
      user_name: "redmine@example.net"
      password: "redmine"
	  
...
production:
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
      address: 127.0.0.1
      port: 25
      domain: localhost
```

### 生成token
```
cd redmine-2.4.0
rake generate_secret_token
```
### 初始化数据库
这步主要是创建所需要的表：
```
RAILS_ENV=production rake db:migrate 
```
### 导入数据
现在开始导入数据到数据库：
```
RAILS_ENV=production rake redmine:load_default_data
Select language: ar, az, bg, bs, ca, cs, da, de, el, en, en-GB, es, et, eu, fa, fi, fr, gl, he, hr, hu, id, it, ja, ko, lt, lv, mk, mn, nl, no, pl, pt, pt-BR, ro, ru, sk, sl, sq, sr, sr-YU, sv, th, tr, uk, vi, zh, zh-TW [en] zh
====================================
Default configuration data loaded.
```
回车之后会让你做一个选择，直接输入zh即可。

### 修改文件系统权限
```
chown -R apache:apache files log tmp public/plugin_assets
chmod -R 755 files log tmp public/plugin_assets
```

## 整合apache
首先安装apache http，如果你已经安装，请忽略此步。
```
yum -y install httpd httpd-devel
```

### 绑定apache
```
gem install passenger
passenger-install-apache2-module 
```
直接按回车。安装完之后创建redmine.conf：
```
touch /etc/httpd/conf.d/redmine.con
```
并添加以下内容：
```
LoadModule passenger_module /usr/lib64/ruby/gems/1.8/gems/passenger-4.0.24/buildout/apache2/mod_passenger.so
PassengerRoot /usr/lib64/ruby/gems/1.8/gems/passenger-4.0.24
PassengerRuby /usr/bin/ruby

Listen 8888
<VirtualHost *:8888>
   ServerName 192.168.1.108
   DocumentRoot /var/www/html/redmine-2.4.0/public
  <Directory /var/www/html/redmine-2.4.0/public >
# This relaxes Apache security settings.
    AllowOverride all
 # MultiViews must be turned off.
    Options -MultiViews
   </Directory>
</VirtualHost>
```
上面有些内容需要你根据实际情况而修改：passenger路径、DocumentRoot

###　启动Apache
```
/etc/init.d/httpd start
```
通过浏览器访问：http://ip:8888，默认的用户名和密码为：admin
以下是登录后的主页：
{% img /images/2013/11/redmine-main.jpg %}

至于redmine的详细使用，期待下次吧。
