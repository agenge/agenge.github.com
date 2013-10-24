---
layout: post
title: "Keystone 源码安装与配置"
date: 2013-10-24 09:02
comments: true
categories: 
- OpenStack
- 云计算
---

## Keystone介绍

什么是Keystone？
Keystone 作为OpenStack中身份认证服务，在OpenStack起到非常关键的作用。它主要负责身份认证，服务规则及令牌的作用，且实现了Identity API,供OpenStack其他各组件之间来进行身份验证。

### 安装依赖包

```
apt-get install -y git python-dev sqlite3 libxml2-dev libxslt1-dev libsasl2-dev libsqlite3-dev libssl-dev libldap2-dev
```

** 下载源码 **

```
git clone https://github.com/openstack/keystone.git 
git clone https://github.com/openstack/python-keystoneclient.git keystone/client
```

安装keystone

```
cd keystone
git checkout origin/stable/grizzly
pip install -r tools/pip-requires
python setup.py install
```

安装客户端

```
cd client
git checkout -b origin/feature/keystone-v3
pip install -r requirements.txt 
python setup.py install

cd ../
mkdir -p /etc/keystone
cp etc/keystone.conf.sample /etc/keystone/keystone.conf
cp etc/logging.conf.sample /etc/keystone/logging.conf
```

** 配置日志存放路径 **

```
mkdir -p /var/log/keystone
touch /var/log/keystone/keystone.log 
```

数据库同步，即创建keystone相关的数据库表

```
keystone-manage db_sync
```

启动服务

```
keystone-all -d &
```

<!-- more -->
## 配置Keytone

截止到现在，我们已经完成Keystone的安装，但现在还无法使用，因为没有租户、用户、密码、服务等。


为使用方便，先设置两个环境变量

```
export OS_SERVICE_TOKEN=ADMIN
export SERVICE_ENDPOINT=http://192.168.30.150:35357/v2.0
```

如果前面安装没问题的话，使用以下命令查看用户列表，默认是没有任何数据返回

```
keystone user-list
```

1. 创建租户

```
keystone tenant-create --name adminTenant --description "Admin Tenant" --enabled true
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |           Admin Tenant           |
|   enabled   |               True               |
|      id     | e36e42a5ed264317a7a5119b1f513e32 |
|     name    |           adminTenant            |
+-------------+----------------------------------+
```
需要记录 tenant id，在创建用户时需要关联，即将用户关联到哪个租户。

2. 创建用户

```
keystone user-create --tenant_id e36e42a5ed264317a7a5119b1f513e32 --name admin --pass openstack --enabled true
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 8cc6ddbeab504e2b9e8bf30eff3d92b7 |
|   name   |              admin               |
| tenantId | e36e42a5ed264317a7a5119b1f513e32 |
+----------+----------------------------------+
```


3. 创建角色

创建一个角色名称为adminRole。请记住该命令生成的Role id，下面的关联用户及租户时需要用到

```
keystone role-create --name adminRole
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|    id    | a00bf7b7fe4e4746bac0fd69778f3dfa |
|   name   |            adminRole             |
+----------+----------------------------------+
```

截止到目前，已经分别创建 Tenant、User、Role，分别是：

Tenant ID: 	e36e42a5ed264317a7a5119b1f513e32
User ID:	8cc6ddbeab504e2b9e8bf30eff3d92b7
Role ID:	a00bf7b7fe4e4746bac0fd69778f3dfa

现在将它们三者关联到一起

```
keystone user-role-add --user-id 8cc6ddbeab504e2b9e8bf30eff3d92b7 \
 --tenant-id e36e42a5ed264317a7a5119b1f513e32 \
 --role-id a00bf7b7fe4e4746bac0fd69778f3dfa
```

## Keytone测试
### 通过 Keystone 获取 Token

访问Keystone 需要4个参数：TenantName Username Password 申请URL，其中URL可以是：

	http://192.168.30.150:35357/v2.0/tokens 或 http://192.168.30.150:5000/v2.0/tokens

```
curl -d '{"auth": {"tenantName": "adminTenant", "passwordCredentials":{"username": "admin", "password": "openstack"}}}' -H "Content-type: application/json" http://192.168.30.150:35357/v2.0/tokens | python -mjson.tool
```
将会返回以下类似信息

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   676  100   567  100   109   6290   1209 --:--:-- --:--:-- --:--:--  6517
{
    "access": {
        "metadata": {
            "is_admin": 0, 
            "roles": [
                "9fe2ff9ee4384b1894a90878d3e92bab", 
                "a00bf7b7fe4e4746bac0fd69778f3dfa"
            ]
        }, 
        "serviceCatalog": [], 
        "token": {
            "expires": "2013-10-16T09:03:21Z", 
            "id": "7c76f8315f6e42d6a05b0ed726bc3441", 
            "issued_at": "2013-10-15T09:03:21.809471", 
            "tenant": {
                "description": "Admin Tenant", 
                "enabled": true, 
                "id": "e36e42a5ed264317a7a5119b1f513e32", 
                "name": "adminTenant"
            }
        }, 
        "user": {
            "id": "8cc6ddbeab504e2b9e8bf30eff3d92b7", 
            "name": "admin", 
            "roles": [
                {
                    "name": "_member_"
                }, 
                {
                    "name": "adminRole"
                }
            ], 
            "roles_links": [], 
            "username": "admin"
        }
    }
}
```

从上述返回结果能看到正常返回 token。更详细的操作请参考[使用Curl操作OpenStack Swift](http://agenge.com/blog/2013/10/17/use-the-curl-operation-swift/)


##错误汇总

```
Traceback (most recent call last):
   File "/usr/bin/easy_install", line 5, in <module>
     from pkg_resources import load_entry_point ImportError: No module named pkg_resources raid:/home/linyoujushi# easy_install genshi Traceback (most recent call last):
```

解决办法：

这是由于我们没有安装setuptools或者没有装好，我们只需要安装这个软件就行了。安装方法：

```
wget http://peak.telecommunity.com/dist/ez_setup.py
python ez_setup.py
```

或者，我们装的setuptools工具太老，我们升级一下：

```
sudo python ./ez_setup.py -U setuptools
```

