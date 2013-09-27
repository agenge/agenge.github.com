---
layout: post
title: awstats 安装与配置分析IIS日志
categories:
- 日志分析
- 运维
tags:
- awstats
- iis
published: true
comments: true

---
##1、准备环境：

* Windows Server 2003
* IIS 6
* ActivePerl-5.16
* awstats-7.0
 
##2、安装ActivePerl
安装Perl for Windows，我们只需要安装ActivePerl 就行了，下载地址为：

32位：[ActivePerl-5.16.2](http://downloads.activestate.com/ActivePerl/releases/5.16.2.1602/ActivePerl-5.16.2.1602-MSWin32-x86-296513.msi)                    64位：[ActivePerl-5.16.2](http://downloads.activestate.com/ActivePerl/releases/5.16.2.1602/ActivePerl-5.16.2.1602-MSWin32-x64-296513.msi)

ActivePerl 的具体安装步骤就不再写出，只管点击“下一步”，安装完之后确认是否已经安装Perl，例如：


```
C:\Documents and Settings\Administrator>perl -v

This is perl 5, version 16, subversion 2 (v5.16.2) built for MSWin32-x86-multi-t
hread
(with 1 registered patch, see perl -V for more detail)

Copyright 1987-2012, Larry Wall

Binary build 1602 [296513] provided by ActiveState http://www.ActiveState.com
Built Dec 19 2012 12:35:59

Perl may be copied only under the terms of either the Artistic License or the
GNU General Public License, which may be found in the Perl 5 source kit.

Complete documentation for Perl, including FAQ lists, should be found on
this system using "man perl" or "perldoc perl". If you have access to the
Internet, point your browser at http://www.perl.org/, the Perl Home Page.
```
如何没有以上类似信息输出，请确认环境变量是否正确。

<!-- more -->
##3、安装awstats
安装之后就会弹出一个DOS窗口，如下：
```
----- AWStats awstats_configure 1.0 (build 1.9) (c) Laurent Destailleur -----
This tool will help you to configure AWStats to analyze statistics for
one web server. You can try to use it to let it do all that is possible
in AWStats setup, however following the step by step manual setup
documentation (docs/index.html) is often a better idea. Above all if:
- You are not an administrator user,
- You want to analyze downloaded log files without web server,
- You want to analyze mail or ftp log files instead of web log files,
- You need to analyze load balanced servers log files,
- You want to 'understand' all possible ways to use AWStats...
Read the AWStats documentation (docs/index.html).

-----> Running OS detected: Windows

-----> Check for web server install
awstats_configure did not find your Apache web main runtime.

Please, enter full directory path of your Apache web server or
'none' to skip this step if you don't have local web server or
don't have permission to change its setup.
Example: c:\Program files\apache group\apache
Apache Web server path ('none' to skip):
> none

Your web server config file(s) could not be found.
You will need to setup your web server manually to declare AWStats
script as a CGI, if you want to build reports dynamically.
See AWStats setup documentation (file docs/index.html)

-----> Update model config file 'C:/Program Files/AWStats\wwwroot\cgi-bin\awstat
s.model.conf'
File awstats.model.conf updated.

-----> Need to create a new config file ?
Do you want me to build a new AWStats config/profile
file (required if first install) [y/N] ? y

-----> Define config file name to create
What is the name of your web site or profile analysis ?
Example: www.mysite.com
Example: demo
Your web site, virtual server or profile name:
> yfb.10086.cn

-----> Create config file 'C:/Program Files/AWStats\wwwroot\cgi-bin\awstats.yfb.
10086.cn.conf'
Config file C:/Program Files/AWStats\wwwroot\cgi-bin\awstats.yfb.10086.cn.conf
created.

-----> Add update process inside a scheduler
Sorry, for Windows users, if you want to have statistics to be
updated on a regular basis, you have to add the update process
in a scheduler task manually (See AWStats docs/index.html).
Press ENTER to continue...      # 按回车
A SIMPLE config file has been created: C:/Program Files/AWStats\wwwroot\cgi-bin\
awstats.yfb.10086.cn.conf
You should have a look inside to check and change manually main parameters.
You can then manually update your statistics for 'yfb.10086.cn' with command:
> perl awstats.pl -update -config=yfb.10086.cn
You can also build static report pages for 'yfb.10086.cn' with command:
> perl awstats.pl -output=pagetype -config=yfb.10086.cn

Press ENTER to finish...       # 按回车
```
##4、awstats 配置
###4.1 配置日志格式

右击“yfb.10086.cn”->“属性”，在“网站”选项卡选择“W3C 扩展日志文件格式”，点击“属性”->“高级”，确保以下字段勾选，其他全部取消：
```
date
time
c-ip
cs-username
cs-method
cs-uri-stem
cs-uri-query
sc-status
sc-bytes
cs-version
cs(User-Agent)
cs(Referer)
```
###4.2 建立虚拟目录

 * 建立虚拟目录cgi-bin,在网站节点下点击“yfb.10086.cn”->“新建”->“虚拟目录”，下一步，别名填写：cgi-bin，路径为C:\Program Files\AWStats\wwwroot\cgi-bin，权限设置为：读取、运行脚本、执行；

 * 建立虚拟目录icon,在网站节点下点击“yfb.10086.cn”->“新建”->“虚拟目录”，下一步，别名填写：icon，路径为C:\Program Files\AWStats\wwwroot\icon，权限设置为：读取、运行脚本、执行；

### 4.3 修改awstats.domain_name.conf

例如：awstats.yfb.10086.cn.conf，路径为网站根目录下的cgi-bin，打开awstats.yfb.10086.cn.conf并修改以下几个变量值：

 * LogFile修改为：LogFile="C:/WINDOWS/system32/LogFiles/W3SVC1237985490/ex%YY-24%MM-24%DD-24.log"，修改之前确保C:/WINDOWS/system32/LogFiles/W3SVC1237985490 这个路径已经存在。
 * LogType 默认值（W）即可，W表示 web log ，S 为流日志，M为邮件日志，F为FTP日志。
 * LogFormat 修改为LogFormat=2，表示IIS日志，1表示Apache日志。
 
更新：根据IIS在实际情况下来看，此处设置2仍然无法按照我们之前设置的日志格式打印日志，所以设置成：</span>
```
LogFormat="date time cs-method cs-uri-stem cs-username c-ip cs-version cs(User-Agent) cs(Referer) sc-status sc-bytes"
```
 * AllowToUpdateStatsFromBrowser 设置为1，表示允许从浏览器手动更新，但需要修改日志目录的权限为"IUSR_XXX" 读写，设置为0表示只允许从命令行更新；
 * 将LoadPlugin="timezone +2"修改为LoadPlugin="timezone +8"，并将最前面的“#”号删除。
  
###4.4 重启IIS

重启IIS后，并删除C:/WINDOWS/system32/LogFiles/W3SVC1237985490目录下所有的日志。


##5、生成awstats数据

###5.1 手动生成awstats 数据

打开一个DOS窗口，并切换到网站根目录/cgi-bin，执行：
```
>perl awstats.pl -config=yfb.10086.cn -update

Create/Update database for config "./awstats.yfb.10086.cn.conf" by AWStats versi
on 7.0 (build 1.971)
From data in log file "C:/WINDOWS/system32/LogFiles/W3SVC1237985490/ex130130.log
"...
Phase 1 : First bypass old records, searching new record...
Searching new records from beginning of log file...
Phase 2 : Now process new records (Flush history on disk after 20000 hosts)...
Jumped lines in file: 0
Parsed lines in file: 84
Found 0 dropped records,
Found 4 comments,
Found 0 blank records,
Found 0 corrupted records,
Found 0 old records,
Found 80 new qualified records.
```
如果报以下错误：
```
Create/Update database for config "./awstats.yfb.10086.cn.conf" by AWStats versi
on 7.0 (build 1.971)
From data in log file "C:/WINNT/system32/LogFiles/W3SVC1237985490/ex130130.log".
..
Error: Couldn't open server log file "C:/WINNT/system32/LogFiles/W3SVC1237985490
/ex130130.log" : No such file or directory
Setup ('./awstats.yfb.10086.cn.conf' file, web server or permissions) may be wrong.
Check config file, permissions and AWStats documentation (in 'docs' directory).
```
请确保你的日志文件路径是否写正确，日志目录是否有权限等；

由于每次都需要手动生成数据，对于我们这种懒人来讲是不可接受的，那怎么自动生成数据呢？

5.2 自动生成awstats数据

首先，您得在C:\Program Files\AWStats\wwwroot\cgi-bin目录创建一个批处理：awstatsUpdate.bat，内容如下：
```
cd C:/
cd C:\Program Files \AWStats\wwwroot\cgi-bin
perl awstats.pl -config=yfb.10086.cn -update
```
注意：以上部分需要根据您的实际情况修改，yfb.10086.cn是需要偶自己的内网域名，千万不可盲目复制。

创建之后，您最好在DOS窗口单独测试下是否能正常运行，如无问题再添加计划任务。

如何增加一个计划任务的具体步骤由于过于简单此不再写出，相信您肯定会添加计划任务，至于任务的调度时间要根据您的需求来定，5分钟也好、5小时也罢，适合您的需求就好，例如本人就是设置的5分钟。
