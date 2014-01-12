---
layout: post
title: "Ubuntu12.04部署Docker内部仓库"
date: 2014-01-12 19:54
comments: true
categories: 
- 云计算
- OpenStack
tags:
- LXC
- Docker
- OpenStack
- PasS
- 虚拟化
---

## 前言
从上篇文章中了解到，push/pull 都是针对的index.docker.io的官方仓库，而实际情况告诉我们，企业大部分服务器是不对外开放权限的，Docker.Inc 其实已经考虑到我们这种实际需求，通过在内网直接搭建私有仓库来解决此问题。

## 安装Docker-registry
```
git clone https://github.com/dotcloud/docker-registry
cd docker-registry/
cp config/config_sample.yml config/config.yml
```
编辑配置文件，主要是修改镜像的存储路径：
```
vim config/config.yml

dev:
    storage: local
    storage_path: /home/taurus/registry
    loglevel: debug
```	
保存退出。并创建存储镜像目录：
```
mkdir -p /you_path/to/registry
```
安装依赖包：
```
sudo apt-get install build-essential python-dev libevent-dev python-pip libssl-dev
sudo pip install -r requirements.txt
```

启动服务：
```
cd docker-registry/
sudo gunicorn --access-logfile - --debug -k gevent -b 0.0.0.0:80 -w 1 wsgi:application & 
```
打开浏览器：http://IP   如果正常会显示："docker-registry server (dev)"

<!-- more -->
## 镜像管理
查看本地系统有哪些镜像文件：
```
sudo docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu              12.04               8dbd9e392a96        8 months ago        128 MB
ubuntu              latest              8dbd9e392a96        8 months ago        128 MB
ubuntu              precise             8dbd9e392a96        8 months ago        128 MB
ubuntu              12.10               b750fe79269d        9 months ago        175.3 MB
ubuntu              quantal             b750fe79269d        9 months ago        175.3 MB
base                latest              b750fe79269d        9 months ago        175.3 MB
```


###上传镜像到 本地仓库：
1. 先打个标签
```
sudo docker tag 8dbd9e392a96 192.168.30.239/ubuntu1204
```
记录上面输出的ID（你要上传哪个镜像就记录哪个ID），然后开始上传：
```
docker push 192.168.30.239/ubuntu1204
```
测试本地仓库是否可用：
```
docker pull 192.168.30.239/ubuntu1204
```
如果没问题，会提示下载完成。

运行一个Hello
```
docker run 192.168.30.239/ubuntu1204 /bin/echo hello
```
这个……用docker来运行打印hello world，真是大材小用。我们来运行一个sshd + tomcat：
```
docker pull 192.168.30.239/ubuntu1204
docker run -i -t 192.168.30.239/ubuntu1204 /bin/bash
```
此时就会进入一个终端：
```
root@d04d4fdde922:/# 
```
现在开始安装ssh
```
apt-get update
apt-get -y install openssh-server
```
安装后还需要设置ssh，方便其他客户端连接：
```
mkdir /var/run/sshd
root@d04d4fdde922:/# passwd
Enter new UNIX password: 
Retype new UNIX password: 
passwd: password updated successfully

exit
```
设置之后还要对这个docker  commit，否则此docker重启后，刚才的设置会消失。
```
docker ps -l
```
获取容器ID
```
root@hgg-pc:~# docker commit d04d4fdde922 192.168.30.239/ubuntu1204
5acd9899febecd686bd6b8ceb66be487495f30a39657dcc36835fd534057b064
```
以后台的方式长期运行ssh
```
docker run -d -p 22 -p 8080:8080 192.168.30.239/ubuntu1204 /usr/sbin/sshd -D
```

* -D  后台运行
* -p  ssh port
* -p  也是端口，后面我们在部署 tomcat 时使用,  8081:8080 映射端口，此容器中的8080端口会被映射到外部的8081端口，即外部使用8081来访问此窗口的8080端口。

查看是否运行成功：
```
docker ps
CONTAINER ID        IMAGE                              COMMAND             CREATED             STATUS              PORTS                                           NAMES
36b805df0f4d        192.168.30.239/ubuntu1204:latest   /usr/sbin/sshd -D   3 minutes ago       Up 3 minutes        0.0.0.0:49154->22/tcp, 0.0.0.0:8080->8080/tcp   trusting_galileo  
```

可以看到我们本地的 49154端口映射到容器的22端口，我们再来登录测试一下：
```
root@hgg-pc:~# ssh root@127.0.0.1 -p 49154
The authenticity of host '[127.0.0.1]:49154 ([127.0.0.1]:49154)' can't be established.
ECDSA key fingerprint is 42:17:df:cb:99:7a:5d:a9:a2:36:f9:0a:64:a9:d8:41.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:49154' (ECDSA) to the list of known hosts.
root@127.0.0.1's password: 
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.8.0-34-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

root@36b805df0f4d:~# 
```
完美！还差一个tomcat。开始操作：
```
apt-get install python-software-properties
add-apt-repository ppa:webupd8team/java
apt-get update
apt-get install -y wget
apt-get install oracle-java7-installer
javac -version

wget http://mirror.bit.edu.cn/apache/tomcat/tomcat-7/v7.0.47/bin/apache-tomcat-7.0.47.tar.gz
tar xvf apache-tomcat-7.0.47.tar.gz
cd apache-tomcat-7.0.47
bin/startup.sh
```

打开浏览器：http://IP  就能看到tomcat首页。
如果不想每次都这样搭建，还是再commit一下吧：
```
docker commit d04d4fdde922 192.168.30.239/ubuntu1204
```
