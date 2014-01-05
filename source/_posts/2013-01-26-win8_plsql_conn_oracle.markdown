---
layout: post
title: Win8通过PLSQL连接Oracle
categories:
- 数据库
tags:
- PLSQL
- Oracle
published: true
comments: true

---
偶一直比较喜欢新产品并很愿意去尝试，虽说特喜欢Linux，但Win8正式版发布后，立即将公司的笔记本第一时间从Ubuntu 10.04迁移到Win8 64位企业版，本文不介绍Win8、不介绍安装Oracle11g，而是记录偶在Win8系统通过PL/SQL连接Oracle的经过，以便于各位看客少走弯路，此问题折腾偶整整一天，费话就此打住，开始：

1. 首先您得安装一个Oracle 11g 64位，建议只安装数据库软件，不创建数据库实例，偶安装后的路径：E:\app\agen\product\11.2.0\dbhome_1Oracle database 11g For Windows 共有2安装包，下载地址：[iso1](http://download.oracle.com/otn/nt/oracle11g/112010/win64_11gR2_database_1of2.zip "iso1")  | [iso2](http://download.oracle.com/otn/nt/oracle11g/112010/win64_11gR2_database_2of2.zip "iso2")

2. 必须再安装一个Oracle 11g 32位的客户端，下载地址：[instantclient-basic-win32-11.2.0.1.0](http://download.oracle.com/otn/nt/instantclient/112010/instantclient-basic-win32-11.2.0.1.0.zip "instantclient-basic")将 instantclient_11_2 解压到：E:\app\agen\product，并把 tnsnames.ora 复制到instantclient_11_2下；

3. 安装PL/SQL DEV，安装包、注册码及汉化包，请见：[PL/SQL Dev + 注册机 + 官方简体中文](http://www.oracle.com.cn/viewthread.php?tid=158668 "PLSQL DEV +注册机 + 汉")，安装后，在登录时选择取消，弹出PLSQL DEV的界面，选择：“工具”-&gt;“首选项”，分别设置如下：

 * Oracle 主目录名：E:\app\agen\product\instantclient_11_2

 * OCI库：E:\app\agen\product\instantclient_11_2\oci.dll

 点击“应用”->“确定”，并关闭PL/SQL。

4. 设置环境变量：
 * 在系统环境变量，选择“Path”，点击“编辑”，在最前面加入以下路径：E:\app\agen\product\instantclient_11_2，并确认。
 * 新建变量名称：TNS_ADMIN， 变量值：E:\app\agen\product\instantclient_11_2(因为我们的tnsnames.ora在这里啦)。
 * 新建变量名称：NLS_LANG， 变量值：SIMPLIFIED CHINESE_CHINA.ZHS16GBK

 请注意,此变量值通过以下SQL语句在数据库服务器得到：
```
SELECT userenv(‘language’) nls_lang FROM dual;
```
最后启动PLSQL，如果您所有步骤顺利通过，就可连接到数据库。
