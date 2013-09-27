---
layout: post
title: 要体现在行动上之Blog恢复
categories:
- 生活
tags:
- blog恢复
published: true
comments: true

---
前几天偶打开自己<a title="agenge" href="http://agenge.com">blog</a>时，突然发现无法打开，具体点说是显示数据库连接错误，第一反应是“被黑”，故而排查的第一步就是查看系统log，然而并没有看到相关错误信息，为尽快恢复，偶先尝试重启mysql，提示没有剩余空间，难道数据库数据文件太多导致？通过du后，发现果然如此，通过将过旧的数据文件（即Mysql bin log）删除即可。以下是整个恢复过程，仅供参考。

** 1. 查看磁盘空间： **
```
[root@centos var]# df -h
Filesystem Size Used Avail Use% Mounted on
/dev/sda1 7.9G 7.9G 0M 100% /
```
** 2. 查看Mysql binlog 大小 **
```
[root@centos var]# pwd
/usr/local/mysql/var
[root@centos var]# ll | wc -l
1582
[root@centos var]# du -sh .
4.1G .
```
VPS的磁盘大小总共才8G，还包括操作系统所占空间，bin log占用足足4.1G。删除吧！

** 3. 删除旧数据 **
```
[root@centos var]# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor. Commands end with ; or \g.
Your MySQL connection id is 45959
Server version: 5.1.54-log Source distribution

Copyright (c) 2000, 2011, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> purge binary logs before '2013-01-01 00:00:00';
Query OK, 0 rows affected (10.39 sec)
mysql> quit
Bye
[root@centos var]# df -h
Filesystem Size Used Avail Use% Mounted on
/dev/sda1 7.9G 4.3G 3.3G 57% /
```
删除2013年1月1日之前的所有数据，剩余可用空间还有3.3G。

** 4. 后记 **
如果以上操作就算完结，那无疑是在等待下一次磁盘被爆，为防止问题再次发生，必须主动监控来达到提前预防，所以偶部署了个小脚本，一旦磁盘达到90%，会mail通知偶。也许是对blog的不重视，之前一直未采取行动，或许惰性控制了我，最后问大家一个问题：如果某个东西对你非常重要，在危机来之前，各位都准备好迎接了吗？理想或想法体现到行动上了吗？

2013/09/27 Update:

如果只想写文章，而不想关心Mysql 及Http Server，甚至担心VPS被黑，偶强烈推荐使用 Octopress + Github，具体介绍及安装请见这里[利用GitHub搭建免费的个人blog](http://agenge.com/blog/2013/09/12/write-blog-octopress-with-github/)
