---
layout: post
title: "redis安装与配置"
date: 2013-10-08 20:05
comments: true
categories: 
- NoSQL
- 数据库

---

## 安装环境
* 操作系统：CentOS 6.3 64位
* Redis：最新版本 2.6.14


## Redis安装
1. 下载Redis

 ```
 yum -y install make gcc
 wget http://download.redis.io/redis-stable.tar.gz
 tar zxvf redis-stable.tar.gz
 cd redis-stable
 make
 make install
 ```
 这样即完成redis的安装,整个安装是否超级简单,至少偶认为不复杂。

2. 启动和停止Redis
在Linux系统中，启动与停止服务也是非常简单，Redis主要提供以下几个可执行程序：

 * redis-server     看名称就明白：Redis服务器
 * redis-cli        Redis命令行客户端
 * redis-benchmark  Redis性能测试工具
 * redis-check-aof  AOF文件修复工具
 * redis-check-dump RDB文件检查工具

 ** 启动Redis **
 
 启动redis,直接在命令行输入：
 
 ```
 redis-server
 ```
 
 如果启动正常，就会输出以下类似信息：
 
 ```
  [2935] 08 Oct 20:29:05.156 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
 [2935] 08 Oct 20:29:05.158 * Max number of open files set to 10032
                 _._                                                  
            _.-``__ ''-._                                             
       _.-``    `.  `_.  ''-._           Redis 2.6.14 (00000000/0) 64 bit
   .-`` .-```.  ```\/    _.,_ ''-._                                   
  (    '      ,       .-`  | `,    )     Running in stand alone mode
  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
  |    `-._   `._    /     _.-'    |     PID: 2935
   `-._    `-._  `-./  _.-'    _.-'                                   
  |`-._`-._    `-.__.-'    _.-'_.-'|                                  
  |    `-._`-._        _.-'_.-'    |           http://redis.io        
   `-._    `-._`-.__.-'_.-'    _.-'                                   
  |`-._`-._    `-.__.-'    _.-'_.-'|                                  
  |    `-._`-._        _.-'_.-'    |                                  
   `-._    `-._`-.__.-'_.-'    _.-'                                   
       `-._    `-.__.-'    _.-'                                       
           `-._        _.-'                                           
               `-.__.-'                                               
 
 [2935] 08 Oct 20:29:05.159 # Server started, Redis version 2.6.14
 [2935] 08 Oct 20:29:05.159 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
 [2935] 08 Oct 20:29:05.159 * The server is now ready to accept connections on port 6379
 ```
 
 从输出信息可以看出redis已经启动成功，默认监听的6379端口。不过我们还有更好的办法来启动redis-server，Redis默认已经为我们提供一个系统启动脚本：
 
 ```
 cp utils/redis_init_script /etc/init.d/redis_server_6379
 chmod +x /etc/init.d/redis_server_6379
 mkdir /etc/redis
 cp redis.conf /etc/redis/6379.conf
 echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
 /etc/init.d/redis_server_6379 start
 ```
 
 ** 关闭Redis **
 相对启动来讲，关闭redis就更为简单，只需要输入：
 
 ```
 redis-cli SHUTDOWN
 ```
 此命令输入完成后，首先会断开所有客户端的连接，然后根本配置进行持久化，如果关闭正常，就会输出以下类似信息：
 
 ```
 [2989] 08 Oct 20:46:08.156 # User requested shutdown...
 [2989] 08 Oct 20:46:08.156 * Saving the final RDB snapshot before exiting.
 [2989] 08 Oct 20:46:08.175 * DB saved on disk
 [2989] 08 Oct 20:46:08.175 # Redis is now ready to exit, bye bye...
 ```

---
## Redis 命令行客户端
前面已经介绍过redis-cli这个工具，通过redis-cli向 Redis服务端发送命令有两种方式：

* 将要发送的命令作为 redis-cli 的参数，例如之前介绍过的  redis-cli SHUTDOWN, 此处的SHUTDOWN即是发送命令，也是一个参数。
* 直接进入 redis-cli 交互式命令行执行命令，例如：
 ```
 redis-cli
 redis 127.0.0.1:6379> ping
 PONG
 ```
 其中“redis 127.0.0.1:6379>” 是redis-cli的命令提示符，类似Mysql中的“mysql> ” ，ping是Redis内置的命令，用来测试是否与服务器连接正常。 

## Redis多数据库
和Mysql一样，Redis也是支持在同一台服务器启动多个redis server，之前已经有过介绍，在启动redis时，用到/etc/redis/6379.conf 这个配置文件，下面就来介绍如何在一台服务器启动多个redis数据库

```
cd /etc/redis/
cp 6379.conf 6380.conf 
```
修改6380.conf 以下内容：

```
pidfile /var/run/redis_6380.pid
port 6380
```

保存退出，修改启动服务：

```
cp /etc/init.d/redis_server_6379 /etc/init.d/redis_server_6380
```
将 REDISPORT=6379  修改为 REDISPORT=6380

启动redis server

```
/etc/init.d/redis_server_6379 start
/etc/init.d/redis_server_6380 start
```
上面代表将启动两个redis 数据库，分别监听6379和6380端口。