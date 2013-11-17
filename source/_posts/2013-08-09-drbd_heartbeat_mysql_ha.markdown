---
layout: post
title: DRBD+Heartbeat+Mysql高可用配置
categories:
- 运维
tags:
- drbd
- heartbeat
- mysql
- 集群
- 高可用
published: true
comments: true
---
##介绍
DRBD全称Distributed Replicated Block （分布式的复制块设备），属于Device公司，但是完全开源。它是一款基于块设备的文件复制解决方案，速度比文件级别的软件如NFS，samba快很多，是很多中小企业的共享存储首选解决方案。

DRBD工作需要在两个节点上同时准备一块一模一样的分区组成镜像，这就是为什么它叫做分布式复制块设备，它主要通过复制数据来实现文件同步（备份），主要用于集群文件共享， 我们通过它的工作原理来了解块复制和文件复制的不同。

首先，您需要知道，DRBD是工作在系统内核空间，而不是用户空间，它直接复制的是二进制数据，这是它速度快的根本原因。其次，DRBD至少需要两个节点来工作，一主一次。

DRBD的文件同步过程和普通复制过程的不同：

DRBD在数据进入Buffer Cache时，先经过DRBD这一层，复制一份数据经过TCP/IP协议封装，发送到另一个节点上，另一个节点通过TCP/IP协议来接受复制过来的数据，同步到次节点的DRBD设备上。


##准备工作
1  准备至少2台服务器，且每台服务器有一块磁盘或一个单独未使用的分区，偶的环境如下：

主节点(Primary Node)
次节点(Secondary Node)
主机名
drbd-01.i.12582.com
drbd-02.i.12582.com
操作系统
CentOS 6.3 x86_64位
CentOS 6.3 x86_64位
硬盘分区
/dev/sdb  8G
/dev/sdb  8G
IP地址
192.168.30.234
192.168.30.235

注意：如果有服务器有2块或以上网卡的同学，建议将其中一块网卡专门用来做网络心跳线，甚至接入另外一台单独的交换机，本次环境只有单网卡。
##初始化设置

1  所有节点关闭iptables和SELinux

2  在所有节点/etc/hosts加入以下内容：

    192.168.30.234  drbd-01.i.12582.com drbd-01
    192.168.30.235  drbd-02.i.12582.com drbd-02

<!-- more -->
3  所有节点创建独立分区：

    fdisk /dev/sdbDevice contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabelBuilding a new DOS disklabel with disk identifier 0x46b64833.Changes will remain in memory only, until you decide to write them.
    After that, of course, the previous content won't be recoverable.
    Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)
    WARNING: DOS-compatible mode is deprecated. It's strongly recommended to
    switch off the mode (command 'c') and change display units to
    sectors (command 'u').
    
    Command (m for help): p
    Disk /dev/sdb: 8589 MB, 8589934592 bytes
    255 heads, 63 sectors/track, 1044 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x46b64833

    Device Boot    Start        End        Blocks    Id    System

    Command (m for help): n
    Command action
    e   extended
    p   primary partition (1-4)
    p
    Partition number (1-4): 1
    First cylinder (1-1044, default 1):
    Using default value 1
    Last cylinder, +cylinders or +size{K,M,G} (1-1044, default 1044):
    Using default value 1044

    Command (m for help): w
    The partition table has been altered!
    Calling ioctl() to re-read partition table.
    Syncing disks.

##安装Hearbeat

    wget ftp://mirror.switch.ch/pool/1/mirror/scientificlinux/6rolling/i386/os/Packages/epel-release-6-5.noarch.rpm
    rpm -ivUh epel-release-6-5.noarch.rpm
    yum --enablerepo=epel install heartbeat -y

##安装DRBD
1  安装依赖包

    yum -y install gcc kernel-devel kernel-headers flex

2  安装DRBD

    wget http://oss.linbit.com/drbd/8.4/drbd-8.4.3.tar.gz
    tar zxvf drbd-8.4.3.tar.gz
    cd drbd-8.4.3
    ./configure --prefix=/usr/local/drbd --with-km --with-heartbeat
    make KDIR=/usr/src/kernels/2.6.32-279.el6.x86_64/
    make install
    cp -R /usr/local/drbd/etc/ha.d/resource.d/drbd* /etc/ha.d/resource.d/



3  其他设置

    mkdir -p /usr/local/drbd/var/run/drbd
    cp /usr/local/drbd/etc/rc.d/init.d/drbd /etc/rc.d/init.d/
    chkconfig --add drbdchkconfig drbd on

##安装DRBD模块

    cd drbdmake 
    clean
    make KDIR=/usr/src/kernels/2.6.32-279.el6.x86_64/
    cp drbd.ko /lib/modules/`uname -r`/kernel/lib/
    depmod

以上操作请在所有节点操作一次。

---
##设置DRBD

**配置drbd.conf**

    cat /usr/local/drbd/etc/drbd.d/global_common.conf
    global {
        minor-count 64;
        usage-count yes;
        # minor-count dialog-refresh disable-ip-verification
    }


    common {
        syncer { rate 1000M; }
        handlers {
            # These are EXAMPLE handlers only.
            # They may have severe implications,
            # like hard resetting the node under certain circumstances.
            # Be careful when chosing your poison.

            pri-on-incon-degr "/usr/lib/drbd/notify-pri-on-incon-degr.sh; /usr/lib/drbd/notify-emergency-reboot.sh; echo b &gt; /proc/sysrq-trigger ; reboot -f";
            pri-lost-after-sb "/usr/lib/drbd/notify-pri-lost-after-sb.sh; /usr/lib/drbd/notify-emergency-reboot.sh; echo b &gt; /proc/sysrq-trigger ; reboot -f";
            local-io-error "/usr/lib/drbd/notify-io-error.sh; /usr/lib/drbd/notify-emergency-shutdown.sh; echo o &gt; /proc/sysrq-trigger ; halt -f";
            fence-peer "/usr/lib/drbd/crm-fence-peer.sh";
            pri-lost "/usr/local/drbd/lib/drbd/notify-pri-lost.sh; /usr/local/drbd/lib/drbd/notify-emergency-reboot.sh; echo b &gt;/proc/sysrq-trigger ; reboot -f";
            split-brain "/usr/lib/drbd/notify-split-brain.sh root";
            out-of-sync "/usr/lib/drbd/notify-out-of-sync.sh root";
            # before-resync-target "/usr/lib/drbd/snapshot-resync-target-lvm.sh -p 15 -- -c 16k";
            # after-resync-target /usr/lib/drbd/unsnapshot-resync-target-lvm.sh;
        }

        startup {

            # wfc-timeout degr-wfc-timeout outdated-wfc-timeout wait-after-sb
            wfc-timeout 60;
            degr-wfc-timeout 120;
            outdated-wfc-timeout 2;
        }

        options {
            # cpu-mask on-no-data-accessible
            }

        disk {
            # size max-bio-bvecs on-io-error fencing disk-barrier disk-flushes
            # disk-drain md-flushes resync-rate resync-after al-extents
            # c-plan-ahead c-delay-target c-fill-target c-max-rate
            # c-min-rate disk-timeout
            on-io-error detach;
            fencing resource-only;
        }
    
        net {
            # protocol timeout max-epoch-size max-buffers unplug-watermark
            # connect-int ping-int sndbuf-size rcvbuf-size ko-count
            # allow-two-primaries cram-hmac-alg shared-secret after-sb-0pri
            # after-sb-1pri after-sb-2pri always-asbp rr-conflict
            # ping-timeout data-integrity-alg tcp-cork on-congestion
            # congestion-fill congestion-extents csums-alg verify-alg
            # use-rle
            protocol C;
        }
    }


设置资源文件

    cat /usr/local/drbd/etc/drbd.d/r0.res
    resource r0 {
        on drbd-01.i.12582.com {
            device          /dev/drbd0;
            disk            /dev/sdb1;
            address         192.168.30.234:7788;
            meta-disk       internal;
        }

        on drbd-02.i.12582.com {
            device          /dev/drbd0;
            disk            /dev/sdb1;
            address         192.168.30.235:7788;
            meta-disk       internal;
        }
    }


---
###设置权限

    chgrp haclient /sbin/drbdsetup
    chmod o-x /sbin/drbdsetup
    chmod u+s /sbin/drbdsetup
    chgrp haclient /sbin/drbdmeta
    chmod o-x /sbin/drbdmeta
    chmod u+s /sbin/drbdmeta


加载DRBD模块与建立resource

    modprobe drbd
    lsmod | grep drbd
    drbd                  328626  0
    libcrc32c               1246  1 drbd


写入一些数据到/dev/sdb1

    dd if=/dev/zero of=/dev/sdb1 bs=1M count=100

建立Resource

    drbdadm create-md r0
    Writing meta data...
    initializing activity log
    NOT initializing bitmap
    New drbd meta data block successfully created.
    success


###启动DRBD

    /etc/init.d/drbd start
    Starting DRBD resources: [
          create res: r0
        prepare disk: r0
         adjust disk: r0
         adjust net: r0
    ]
    degr-wfc-timeout has to be shorter than wfc-timeout
    degr-wfc-timeout implicitly set to wfc-timeout (60s)
    ..........
    ***************************************************************
    DRBD's startup script waits for the peer node(s) to appear.
    - In case this node was already a degraded cluster before the
    reboot the timeout is 120 seconds. [degr-wfc-timeout]
    - If the peer was available before the reboot the timeout will
    expire after 60 seconds. [wfc-timeout]
    (These values are for resource 'r0'; 0 sec -&gt; wait forever)
    To abort waiting enter 'yes' [  49]:yes

    .

节点2按以上操作执行一次。


###DRBD状态查看

1. Secondary/Unknown：若drbd-01服务启动而drbd-02尚未启动,则ro会出现Secondary/Unknown

        /init.d/drbd status
        drbd driver loaded OK; device status:
        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        m:res  cs        ro                 ds                 p   mounted
        fstype
        0:r0   WFConnection  Secondary/Unknown  Inconsistent/DUnknown  C
查看drbd

        cat /proc/drbd
        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        0: cs:WFConnection ro:Secondary/Unknown ds:Inconsistent/DUnknown C r----s
        ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:8385604
    
2. Inconsistent/Inconsistent：ds出現Inconsistent表示兩台node資料尚未同步。

        drbd driver loaded OK; device status:
        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        m:res  cs         ro                   ds                         p  mounted  
        fstype
        0:r0   Connected  Secondary/Secondary  Inconsistent/Inconsistent  C
查看内核

        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        0: cs:Connected ro:Secondary/Secondary ds:Inconsistent/Inconsistent C r-----
        ns:0 nr:0 dw:0 dr:0 al:0 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:8385604
3  SyncSource：cs出現SyncTarget表示正在同步中，可看到目前同步時的進度．

        drbd driver loaded OK; device status:
        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        m:res  cs          ro                 ds                     p  mounted  
        fstype
        ...    sync'ed:    26.1%              (6056/8188)M
        0:r0   SyncSource  Primary/Secondary  UpToDate/Inconsistent  C
4  Secondary/Secondary：表示尚未設定Primary Node。

        drbd driver loaded OK; device status:
        version: 8.4.3 (api:1/proto:86-101)
        GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
        root@drbd-01.i.12582.com, 2013-08-08 14:00:35
        m:res  cs         ro                   ds                         p  mounted  
        fstype
        0:r0   Connected  Secondary/Secondary  Inconsistent/Inconsistent  C

###设置主节点(Primary Node)

在drbd-01进行以下操作:

    drbdadm -- --overwrite-data-of-peer primary r0
再查看下状态:

    /etc/init.d/drbd status
    drbd driver loaded OK; device status:
    version: 8.4.3 (api:1/proto:86-101)
    GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
    root@drbd-01.i.12582.com, 2013-08-08 14:00:35
    m:res  cs          ro                 ds                     p  mounted  fstype
    ...    sync'ed:    13.1%              (7124/8188)M
    0:r0   SyncSource  Primary/Secondary  UpToDate/Inconsistent  C

在drbd-02查看下状态：

    driver loaded OK; device status:
    version: 8.4.3 (api:1/proto:86-101)
    GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by 
    root@drbd-02.i.12582.com, 2013-08-08 14:16:07
    m:res  cs         ro                 ds                 p  mounted  fstype
    0:r0   Connected  Secondary/Primary  UpToDate/UpToDate  C


格式化（drbd-01）

    mkfs.ext4 /dev/drbd0
挂载到文件系统：

    mkdir /datamount /dev/drbd0 /data

##测试DRBD
###测试同步
1. 在主节点（Primary Node）复制一些数据到/data

        cp -r drbd-8.4.3 /data/
2. 在次节点（Secondary Node）执行以下步骤：

        drbdadm down r0
注：次节点在DRBD启动状态下是无法mount /data的，所以必须先手动停止才能mount。

        mkdir /datamount -t ext4 /dev/sdb1 /datals -l
就可以看到数据已经全部同步过来。

##Heartbeat配置
1. 主节点操作：

        vim /etc/ha.d/ha.cf
        debugfile /var/log/ha-debug   # 打开错误日志报告
        
        keepalive 2    # 2秒检测一次心跳线连接
        deadtime 10    # 10秒测试不到 主节点心跳线就认为有问题
        warntime 6    # 警告时间（建议在2－10之间）
        initdead 120   # 初始化启动时 120秒无连接视为正常，或指定heartbeat 在启动时，
        # 需要等待120秒才去启动任何资源
        
        udpport 694     # 用udp的694端口连接，netstat -antulp | grep 694
        ucast eth0 192.168.30.235      # 单播方式连接（主、从都是写对方的IP连接）
        node  drbd-01.i.12582.com  # 声明主节点（uname -n）
        node  drbd-02.i.12582.com  # 声明次节点（uname -n）
        auto_failback on                    # 自动切换（主节点恢复后会自动切换回来）
        respawn hacluster /usr/lib64/heartbeat/ipfail  #监控ipfail进程是否挂掉，否则重启它
2. 编辑认证文件

        vim /etc/ha.d/authkeys
        auth 1
        1 sha1 MySecret&nbsp;
3. 权限

        chmod 600 /etc/ha.d/authkeys
4. 编辑资源文件

        vim /etc/ha.d/haresources
        drbd-01.i.12582.com IPaddr::192.168.30.229/24/eth0 drbddisk::r0 
        Filesystem::/dev/drbd0::/data::ext4
    drbd-01.i.12582.com   主节点的主机名
        
    IPaddr::192.168.30.229/24/eth0    设置虚拟IP
    
    drbddisk::r0                  管理资源r0
    
    Filesystem::/dev/drbd0::/data::ext4   执行umount和mount操作
5. 次节点操作：
    
    将ha.cf中的192.168.30.235改成192.168.30.234


###DRBD主从自动切换测试
1. 首先在 drbd-01启动heartbeat：

        /etc/init.d/heartbeat  start

2. 接着在 drbd-02 启动heartbear:

        /etc/init.d/heartbeat  start

3. 在drbd-01输入：

        ip a

        1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 16436 qdisc noqueue state UNKNOWN
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
            inet6 ::1/128 scope host
                valid_lft forever preferred_lft forever
        2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP qlen 1000
            link/ether 08:00:27:98:a2:6c brd ff:ff:ff:ff:ff:ff
            inet 192.168.30.234/24 brd 192.168.30.255 scope global eth0
            inet <b>192.168.30.229</b>/24 brd 192.168.30.255 scope global secondary eth0:0
            inet6 fe80::a00:27ff:fe98:a26c/64 scope link
                valid_lft forever preferred_lft forever
从结果可以看出，VIP已经出现。

4. 停止drbd-01的heartbeat服务或将网线断掉，同时监控drbd-02的DRBD状态.

5. drbd-02操作：

        watch -n 1 /etc/init.d/drbd status
如果一切正常，可以看到状态在不断变化。

6. 恢复drbd-01的heartbeat服务或将网线接上，同时监控drbd-02的DRBD状态，如果正常drbd-01又变为主节点（auto_failback on 决定）了。

##Mysql+DRBD+Heartbeat配置
###Mysql安装

**安装Mysql依赖包**

    yum -y install gcc gcc-c++ ncurses-devel libtool zlib-devel bison

<b>创建用户与组</b>

    groupadd mysqluseradd -g mysql mysql

<b>设置内核参数</b>

    vi /etc/security/limits.conf
    mysql              soft    nproc   2047
    mysql              hard    nproc   16384
    mysql              soft    nofile  1024
    mysql              hard    nofile  65536


<b>安装Mysql</b>

由于源码安装Mysql5.6需要依赖cmake，必须先安装cmake：

    mkdir ~/software; cd ~/software
    wget ftp://192.168.30.211/pub/Tools/mysql/cmake-2.8.4.tar.gz
    tar zxvf cmake-2.8.4.tar.gz
    cd cmake-2.8.4
    ./configure
    gmake &&  make install


开始安装Mysql

    cd ~/software
    wget ftp://192.168.30.211/pub/Tools/mysql/mysql-5.6.5-m8.tar.gz
    tar zxvf mysql-5.6.5-m8.tar.gz
    cd mysql-5.6.5-m8

    cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
      -DDEFAULT_CHARSET=utf8 \
      -DDEFAULT_COLLATION=utf8_general_ci \
      -DENABLED_LOCAL_INFILE=ON \
      -DWITH_INNOBASE_STORAGE_ENGINE=1 \
      -DWITH_FEDERATED_STORAGE_ENGINE=1 \
      -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
      -DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
      -DWITH_PARTITION_STORAGE_ENGINE=1 \
      -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 \
      -DCOMPILATION_COMMENT='agenge for mysql' \
      -DWITH_READLINE=ON \
      -DMYSQL_UNIX_ADDR=/data/mysqldata/3306/mysql.sock \
      -DSYSCONFDIR=/data/mysqldata/3306
      make && make install

###Mysql设置

* 设置权限

        chown -R mysql:mysql /usr/local/mysql

* 设置环境变量

        vi /etc/profileexport PATH=/usr/local/mysql/bin:$PATH

* 使环境变量马上生效

        source /etc/profile

* 设置相关存储路径

        cd /data
        mkdir -p mysqldata/3306/{data,binlog,tmp,innodb_ts,innodb_log}
        cd /data/mysqldata/mkdir script backup
        chown -R mysql:mysql /data/mysqldata/

* 创建my.cnf

        vi /data/mysqldata/3306/my.cnf
        [client]
        port = 3306
        socket = /data/mysqldata/3306/mysql.sock

        # Here follows entries for some specific programs

        # The MySQL server
        [mysqld]
        port     = 3306
        user     = mysql
        socket   = /data/mysqldata/3306/mysql.sock
        pid-file = /tmp/mysql.pid
        basedir  = /usr/local/mysql
        datadir  = /data/mysqldata/3306/data
        tmpdir   = /data/mysqldata/3306/tmp
        open_files_limit    = 10240
        server-id = 333306
        lower_case_table_names = 1
        character-set-server = utf8
        skip-name-resolve
        max_connections = 1000
        max_connect_errors = 100000
        max_allowed_packet = 512M
        max_heap_table_size = 1024M
        max_length_for_sort_data = 4096
        back_log=100
        interactive_timeout = 28800
        wait_timeout = 28800

        default-storage-engine = InnoDB

        net_buffer_length = 8K
        sort_buffer_size = 2M
        join_buffer_size = 4M
        read_buffer_size = 2M
        read_rnd_buffer_size = 16M
        
        query_cache_size = 128M
        query_cache_limit = 2M
        query_cache_min_res_unit = 2k

        thread_cache_size = 300
        table_open_cache = 1024
        tmp_table_size = 256M

        #***********  Logs related settings ***********
        log-bin  = ../binlog/mysql-bin
        relay-log = ../binlog/mysql-relay-bin
        binlog_format=mixed
        binlog_cache_size=32m
        max_binlog_cache_size=512m
        max_binlog_size=512m
        long_query_time = 1
        log_output = FILE
        log-error =  ../mysql-error.log
        slow_query_log = 1
        slow_query_log_file = ../slow_statement.log
        #log_queries_not_using_indexes
        general_log = 0
        general_log_file = ../general_statement.log
        expire-logs-days = 14

        #*********** MyISAM Specific options ***********
        key_buffer_size = 32M
        bulk_insert_buffer_size = 64M
        myisam_sort_buffer_size = 128M
        myisam_max_sort_file_size = 10G
        myisam_repair_threads = 1
        myisam_recover

        #*********** INNODB Specific options ***********
        innodb_file_per_table = 1
        transaction-isolation = READ-COMMITTED

        innodb_additional_mem_pool_size = 16M
        innodb_buffer_pool_size = 1192M
        innodb_data_home_dir = ../innodb_ts
        innodb_data_file_path = ibdata1:2048M:autoextend

        innodb_file_io_threads = 4
        innodb_thread_concurrency = 8
        innodb_log_buffer_size = 128M
        innodb_log_file_size = 256M
        innodb_log_files_in_group = 3

        innodb_log_group_home_dir = ../innodb_log
        innodb_flush_log_at_trx_commit = 2
        innodb_max_dirty_pages_pct = 80
        innodb_lock_wait_timeout = 120
        innodb_flush_method=O_DIRECT
        performance_schema

        [mysqldump]
        quick
        max_allowed_packet = 512M

        [mysql]
        no-auto-rehash
        # Remove the next comment character if you are not familiar with SQL
        #safe-updates

        [myisamchk]
        key_buffer_size = 32M
        sort_buffer_size = 20M
        read_buffer_size = 2M
        write_buffer_size = 2M

        [mysqlhotcopy]
        interactive-timeout

        [mysqld_safe]
        open-files-limit = 8192

        
* 安装mysql database，并启动Myql：

        /usr/local/mysql/scripts/mysql_install_db \
          --datadir=/data/mysqldata/3306/data \
          --defaults-file=/data/mysqldata/3306/my.cnf \
          --basedir=/usr/local/mysql --user=mysql
        mysqld_safe --defaults-file=/data/mysqldata/3306/my.cnf &amp;
        mysqladmin -uroot password 'new_password' -S /data/mysqldata/3306/mysql.sock

        mysql -uroot -p123456
        mysql> select user,host from mysql.user;
        mysql> delete from mysql.user where (user,host) not in (select 'root','localhost');
        mysql> delete from mysql.proxies_priv where host='localhost.localdomain';
        mysql> delete from mysql.db;
        mysql> flush privileges;
        mysql> quit
        mysqladmin -uroot -p123456 shutdownrm -f /data/mysqldata/3306/mysql-error.logmysqld_safe --defaults-file=/data/mysqldata/3306/my.cnf &amp;

* 启动Mysql脚本

        cp support-files/mysql.server  /etc/init.d/mysqld
        chmod +x /etc/init.d/mysqld

* 修改ha.cf

 由于现在是管理Mysql，故要将mysqld由heartbeat管理（2个节点都执行）

        cat /etc/ha.d/haresources
        drbd-01.i.12582.com IPaddr::192.168.30.229/24/eth0:0  drbddisk::r0 
        Filesystem::/dev/drbd0::/data::ext4 mysqld

* 
测试

    1. 准备Mysql数据（节点1操作）

            mysql -uroot -p
            Enter password:
            Welcome to the MySQL monitor.  Commands end with ; or \g.
            Your MySQL connection id is 2
            Server version: 5.6.5-m8-log agenge for mysql

            Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.

            Oracle is a registered trademark of Oracle Corporation and/or its
            affiliates. Other names may be trademarks of their respective
            owners.

            Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

            mysql> create database mydb;
            Query OK, 1 row affected (0.02 sec)

            mysql> use mydb;
            Database changed
            mysql> create table t_1( id  int not null, name varchar(10));
            Query OK, 0 rows affected (0.14 sec)
            mysql> insert into t_1 values(1,'aaa');
            Query OK, 1 row affected (0.00 sec)

            mysql> commit;
            Query OK, 0 rows affected (0.00 sec)
            mysql> quit;
            Bye

    2. 切换主、次节点，同时监控drbd-02服务器的日志

            watch -n 1 /etc/init.d/drbd status

    如果一切正常，可以看到ro的状态从“Secondary/Primary”变成“Primary/Secondary”。

    例如偶切换成的状态如下：

            drbd driver loaded OK; device status:
            version: 8.4.3 (api:1/proto:86-101)
            GIT-hash: 89a294209144b68adb3ee85a73221f964d3ee515 build by root@<b>drbd-02.i.12582.com</b>, 2013-08-08 16:46:23
            m:res  cs         ro                 ds                 p  mounted  fstype
            0:r0   Connected  <b>Primary/Secondary</b>  UpToDate/UpToDate  C  /data    ext4

    3. 查询数据是否丢失（drbd-02操作）

            mysql -uroot -p
            Enter password:
            Welcome to the MySQL monitor.  Commands end with ; or \g.
            Your MySQL connection id is 2
            Server version: 5.6.5-m8-log agenge for mysql

            Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.

            Oracle is a registered trademark of Oracle Corporation and/or its
            affiliates. Other names may be trademarks of their respective
            owners.

            Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

            mysql> use mydb;
            Database changed
            mysql> select * from t_1;
            +----+------+
            | id | name |
            +----+------+
            |  1 | aaa  |
            +----+------+
            1 row in set (0.00 sec)

    以看到在drbd-02上数据已经有了。
