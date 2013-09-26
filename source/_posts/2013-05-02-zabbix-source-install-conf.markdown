---
layout: post
title: Zabbix 源码安装与配置
categories:
- 运维
- 监控
tags:
- zabbix
- 监控系统
published: true
comments: true
---
##1、安装环境
* 操作系统：:CentOS 5.5 64位
* Mysql：5.5.27
* Apache httpd：2.4.4
* Zabbix：2.0.6

##2、源码安装
###安装Zabbix
####创建用户
```
groupadd zabbix
useradd -g zabbix -M -s /sbin/nologin zabbix
```
###源码安装Zabbix
下载页面请点击 [这里](http://www.zabbix.com/download.php)。下载之后按照以下步骤执行：

```
yum install wget curl-devel net-snmp-devel php-bcmath
tar zxvf zabbix-2.0.6.tar.gz
cd zabbix-2.0.6
./configure --prefix=/data/zabbix \
  --enable-server --enable-agent \
  --with-mysql --with-net-snmp \
  --with-libcurl --enable-proxy

make && make install
```
如果一切顺利，安装完成。


###创建Zabbix 数据库
这里假设你已经安装好Mysql数据库，具体的安装方法请自己到网上搜索解决。
```
mysql -uroot -p
Enter password:

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 63
Server version: 5.0.95 Source distribution

Copyright (c) 2000, 2011, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database zabbix character set utf8;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
Query OK, 1 row affected (0.00 sec)

mysql> quit
mysql -uzabbix -p zabbix &lt; database/mysql/schema.sql
mysql -uzabbix -p zabbix &lt; database/mysql/images.sql
mysql -uzabbix -p zabbix &lt; database/mysql/data.sql
```
###配置Zabbix
设置服务自启动：
```
cp misc/init.d/fedora/core/zabbix_server /etc/init.d/
chmod +x /etc/init.d/zabbix_server
```
修改zabbix_server中的BASEDIR=/data/zabbix
```
chkconfig --add zabbix_server
chkconfig zabbix_server on
```
设置zabbix 配置文件：
```
vim /data/zabbix/etc/zabbix/zabbix_server.conf
DBName=zabbix
DBPassword=zabbix
```

只需要修改密码即可，其他都保持默认值。当然如果你的Mysql不是安装在本地，肯定也要修改相应的IP啦。


####启动Zabbix Server
```
/etc/init.d/zabbix_server start
```
确认zabbix server是否已经启动：
```
netstat -antulp | grep zabbix

tcp        0      0 0.0.0.0:10051               0.0.0.0:*                   LISTEN      19087/zabbix_server
```
<!-- more -->

##3、安装Zabbix Web接口
Zabbix前端使用PHP编写，故Web Server必须支持PHP。
####安装Apr
```
wget http://mirrors.cnnic.cn/apache//apr/apr-1.4.6.tar.gz
tar zxvf apr-1.4.6.tar.gz
cd apr-1.4.6
./configure --prefix=/usr/local/apr
make && make install
```
####安装Apr-utils
```
wget http://mirrors.cnnic.cn/apache//apr/apr-util-1.5.2.tar.gz
tar zxvf apr-util-1.5.2.tar.gz
cd apr-util-1.5.2
./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/
make && make install
```
####安装pcre
```
tar zxvf pcre-8.32.tar.gz
cd pcre-8.32
./configure
make && make install
```
###安装Apache httpd
```
wget http://labs.mop.com/apache-mirror//httpd/httpd-2.4.4.tar.bz2
tar jxvf httpd-2.4.4.tar.bz2
cp -rf apr-1.4.6 /data/software/httpd-2.4.4/srclib/apr
cp -rf apr-util-1.5.2 /data/software/httpd-2.4.4/srclib/apr-util
cd httpd-2.4.4
./configure --prefix=/usr/local/httpd \
  --enable-modules --enable-ssl --enable-module=so \
  --with-apr=/usr/local/apr \
  --with-apr-util=/usr/local/apr-util --with-included-apr \
  --enable-mods-shared=most --with-included-apr

make && make install
/usr/local/httpd/bin/apachectl start
```
####安装 PHP
安装gettext（国际化支持）：
```
wget http://ftp.gnu.org/pub/gnu/gettext/gettext-0.18.2.tar.gz
tar zxvf gettext-0.18.2.tar.gz
cd gettext-0.18.2
./configure
make && make install
```
安装libpng：
```
tar zxvf libpng-1.6.1.tar.gz
cd libpng-1.6.1
./configure --prefix=/usr/local/libpng --enable-shared
make && make install
```
安装freetype：
```
tar jxvf freetype-2.4.11.tar.bz2
cd freetype-2.4.11
./configure --prefix=/usr/local/freetype<b></b>
make && make install
```
安装JPEG：
```
wget http://www.ijg.org/files/jpegsrc.v9.tar.gz
tar zxvf jpegsrc.v9.tar.gz
cd jpeg-9/
./configure --prefix=/usr/local/jpeg9 --enable-shared --enable-static
make && make install
```
安装GD库：
```
wget https://bitbucket.org/libgd/gd-libgd/get/GD_2_0_33.tar.gz
tar zxvf GD_2_0_33.tar.gz
cd libgd-gd-libgd-486e81dea984/src
./configure --with-png  --with-freetype  --with-jpeg
make install
```
安装PHP：
```
wget ftp://192.168.30.211:/pub/Tools/php-5.3.19.tar.bz2
tar jxvf php-5.3.19.tar.bz2

./configure --prefix=/usr/local/php  \
  --with-config-file-path=/usr/local/php/etc \
  --disable-debug --disable-rpath --with-gettext  \
  --with-mcrypt --with-mysql=/usr/local/mysql \
  --with-mysql-sock=/data/mysqldata/3306/mysql.sock \
  --with-mysqli=/usr/local/mysql/bin/mysql_config \
  --enable-mbstring --enable-pdo --with-curl \
  --enable-inline-optimization --with-bz2 \
  --with-zlib --enable-sockets --enable-bcmath \
  --enable-sysvsem --enable-sysvshm --enable-pcntl \
  --enable-mbregex --with-mhash --enable-xml \
  --enable-zip --with-pcre-regex --with-gettext \
  --with-apxs2=/usr/local/httpd/bin/apxs \
  --with-gd --enable-gd-native-ttf --with-jpeg-dir=/usr/local/include \
  --with-png-dir=/usr/local/include --with-freetype-dir=/usr/include/freetype2
  
make ZEND_EXTRA_LIBS='-liconv'
make install
cp php.ini-production /usr/local/php/etc/php.ini
cp sapi/fpm/init.d.php-fpm.in /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm
chkconfig php-fpm on
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
```
修改时区：
```
vim /usr/local/php/etc/php.ini
```
在 [Date] 之后增加一行：
```
date.timezone =  Asia/Chongqing
```
将 max_execution_time 的值修改为 300或更大值，将max_input_time的值修改为300或更大值。



修改httpd.conf：
```
vim /usr/local/httpd/conf/httpd.conf
```
在最后增加：
```
AddType application/x-httpd-php  .php
AddType application/x-httpd-php-source  .phps
```
并将：
```
DirectoryIndex index.html
```
修改为：
```
DirectoryIndex index.php index.html
```
####配置Zabbix Web接口
```
mkdir /usr/local/httpd/htdocs/zabbix
cd frontends/php/
cp -a . /usr/local/httpd/htdocs/zabbix/
cd /usr/local/httpd/htdocs/zabbix/
cp conf/zabbix.conf.php.example conf/zabbix.conf.php
vim conf/zabbix.conf.php
```
修改好连接Myql数据库的相关信息。


####安装Zabbix前台

使用浏览器打开Zabbix URL: http://server_ip/zabbix

用户名：admin

密码：  zabbix


##安装Zabbix Agent
###Zabbix Agent For Linux
添加用户：
```
groupadd zabbix
useradd -g zabbix -M -s /sbin/nologin zabbix
```
下载 Zabbix Agent
```
wget http://www.zabbix.com/downloads/2.0.6/zabbix_agents_2.0.6.linux2_6.amd64.tar.gz
tar zxvf zabbix_agents_2.0.6.linux2_6.amd64.tar.gz
mv sbin/zabbix_agent* /usr/sbin/
mv bin/zabbix_* /usr/bin/
mkdir -p /etc/zabbix
mv conf/* /etc/zabbix
echo "zabbix_agent    10050/tcp" >> /etc/services
echo "zabbix_agent    10050/udp" >> /etc/services
sed -i 's/Server=127.0.0.1/Server=192.168.30.226/' /etc/zabbix/zabbix_agentd.conf
sed -i 's/Hostname=Zabbix server/Hostname=192.168.30.226/' /etc/zabbix/zabbix_agentd.conf
```
启动 Zabbix Agent
```
zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf
ps aux | grep zabbix
echo "zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf" >> /etc/rc.local
```
测试是否正确
```
zabbix_get -s 192.168.30.226 -p 10050 -k agent.ping
1
```
如果返回1表示正常。
