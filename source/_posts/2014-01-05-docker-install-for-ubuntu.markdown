---
layout: post
title: "Ubuntu 12.04 安装Docker"
date: 2014-01-05 23:11
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

## 了解Docker 
Docker可谓是2013年的非主流词语，本篇文章为初学者提供入门参考，不涉及介绍，更多的介绍请参见国内资深人士写的文章：

[Docker-Getting-Start](http://tiewei.github.io/cloud/Docker-Getting-Start/)

由于需要在Linux Kernel 3.8及以上才可以很好的工作，官方更是推荐Ubuntu系统，这里有两种选择：Ubuntu 12.04 LTS或最新的Ubuntu 13.10
而本文比较喜欢倾向LTS，幸好有办法解决Kernel版本问题。

提示：以下操作中，绝大部分都是走的goagent代理，关于goagent代理请自行Google。

## 安装Docker
### 更新系统

```
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8087/" update 
sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
sudo reboot
```
### 安装Docker
先解决GFW的问题，所以要设置wget proxy：
```
cat > ~/.wgetrc <EOF
#You can set the default proxies for Wget to use for http, https, and ftp.
# They will override the value in the environment.
https_proxy = http://127.0.0.1:8087/
http_proxy = http://127.0.0.1:8087/
ftp_proxy = http://127.0.0.1:8087/

# If you do not want to use proxy at all, set this to off.
use_proxy = on
EOF
```

同样，docker.io也被墙了，设置下hosts即可；
```
cat >> /etc/hosts <<EOF
54.234.135.251  get.docker.io
54.234.135.251  cdn-registry-1.docker.io
EOF
```

下面才开始安装Docker:
```
sudo sh -c "wget -qO- https://get.docker.io/gpg | apt-key add -"
sudo sh -c "echo deb http://get.docker.io/ubuntu docker main \
 /etc/apt/sources.list.d/docker.list"
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8087/" update 
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8087/" install lxc-docker
```
<!-- more -->
### 测试Docker功能
在Hosts第一次运行Container，时间会比较长，因为它需要去官网下载对应的镜像文件。
```
docker run ubuntu /bin/echo hello
```
运行一个交互式Container:
```
sudo docker run -i -t ubuntu /bin/bash
root@3765e92ae239:/# 
root@3765e92ae239:/# ps
  PID TTY          TIME CMD
    1 ?        00:00:00 bash
    9 ?        00:00:00 ps
```

如果要退出此Container，直接输入exit, 退出之后再次进入Container会发现之前如果有创建新文件或安装新软件都已消失。要解决这个问题，就需要保存刚才的镜像状态：
```
sudo docker run -i -t ubuntu /bin/bash
root@f2d122c6d1b0:/# cd /root/
root@f2d122c6d1b0:/root# touch t.txt
root@f2d122c6d1b0:/root# ll
bash: ll: command not found
root@f2d122c6d1b0:/root# ls -l
total 0
-rw-r--r-- 1 root root 0 Jan  5 15:03 t.txt
root@f2d122c6d1b0:/root# exit
exit
```

退出之后马上提交：
```
docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
f2d122c6d1b0        ubuntu:12.04        /bin/bash           About a minute ago   Exit 0                                  compassionate_heisenberg 
```

ps -l 是打印最后一个Container，包括停止的Container。查找到Container ID 就要对它提交：
```
docker commit f2d122c6d1b0 ubuntu
ac2f8a958c9ec376d371857f3e82858e67da95572823ae545d79f00ef66bf84f
```

再次验证：
```
sudo docker run -i -t ubuntu /bin/bash
root@a1d254851cd4:/# cd /root
root@a1d254851cd4:/root# ls
t.txt
```
从结果可看出刚才所创建的文件依然存在。以上仅是Docker的安装与入门教程，也是偶自己学习的记录过程，后期准备再写4个教程：

* 搭建Docker内部仓库
* 基于Dockerfile快速部署应用
* Docker管理
* Docker Web管制台