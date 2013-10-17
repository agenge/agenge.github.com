---
layout: post
title: "使用Curl操作OpenStack Swift"
date: 2013-10-17 09:51
comments: true
categories: 
- OpenStack
- 云计算
---

提示：以下操作均是使用的 swift tempauth认证机制。

- 获取 Token

```
	curl -k -v -H 'X-Storage-User: admin:admin' -H 'X-Storage-Pass: admin' http://192.168.30.150:8080/auth/v1.0
```

如果正确，将会返回以下类似信息：

```
* About to connect() to 192.168.30.150 port 8080 (#0)
*   Trying 192.168.30.150... connected
> GET /auth/v1.0 HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: 192.168.30.150:8080
> Accept: */*
> X-Storage-User: admin:admin
> X-Storage-Pass: admin
> 
< HTTP/1.1 200 OK
< X-Storage-Url: http://192.168.30.150:8080/v1/AUTH_admin
< X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5
< Content-Type: text/html; charset=UTF-8
< X-Storage-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5
< Content-Length: 0
< Date: Tue, 15 Oct 2013 01:49:59 GMT
< 
* Connection #0 to host 192.168.30.150 left intact
* Closing connection #0
```	

- Account操作

```
	curl -k -v -X HEAD -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin
```

如果正确，将会返回以下类似信息：

```
* About to connect() to 192.168.30.150 port 8080 (#0)
*   Trying 192.168.30.150... connected
> HEAD /v1/AUTH_admin HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: 192.168.30.150:8080
> Accept: */*
> X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5
> 
< HTTP/1.1 204 No Content
< Content-Length: 0
< Accept-Ranges: bytes
< X-Timestamp: 1381806617.24083
< X-Account-Bytes-Used: 0
< X-Account-Container-Count: 1
< Content-Type: text/plain; charset=utf-8
< X-Account-Object-Count: 0
< Date: Tue, 15 Oct 2013 05:17:23 GMT
< 
* Connection #0 to host 192.168.30.150 left intact
* Closing connection #0
```

- Container操作

  * 列出 Contailner

```
curl -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin
```
 如果正确，将会返回以下类似信息：

```
HTTP/1.1 200 OK
Content-Length: 5
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
X-Account-Bytes-Used: 0
X-Account-Container-Count: 1
Content-Type: text/plain; charset=utf-8
X-Account-Object-Count: 0
Date: Tue, 15 Oct 2013 05:20:11 GMT

test
```
 最后一行的test就是查询出来的内容。

  * 创建 Container

```
curl -i -X PUT -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/myfile
```

   如果正确，将会返回以下类似信息：
  
```
HTTP/1.1 201 Created
Content-Length: 0
Content-Type: text/html; charset=UTF-8
Date: Tue, 15 Oct 2013 05:22:01 GMT
```
  
  我们再来查询一次看是否成功：

  <!-- more -->
  
```
curl -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin
```
  
  如果正确，将会返回以下类似信息：

```
HTTP/1.1 200 OK
Content-Length: 12
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
Content-Type: text/plain; charset=utf-8
X-Account-Object-Count: 0
Date: Tue, 15 Oct 2013 05:23:18 GMT

myfile
test
```
   
   * 只列出部分 Container
 
  很多时候 Container 会有很多个，Swift 默认会列出前10000个，但如果我们只看最前面几个，该怎么办？ 以下示例只显示最前面一个 Container

```
curl -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin?limit=1
```	
  
 结果：

```
HTTP/1.1 200 OK
Content-Length: 7
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
Content-Type: text/plain; charset=utf-8
X-Account-Object-Count: 0
Date: Tue, 15 Oct 2013 05:24:36 GMT
  
myfile
```

  那如果要列出最后几个 Container 又怎么办呢？ 加一个 marker即可，以下示例列出 myfile之后的一个Container

```
curl -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin?marker=myfile\&limit=1
```

 如果正确，将会返回以下类似信息：

```
HTTP/1.1 200 OK
Content-Length: 5
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
Content-Type: text/plain; charset=utf-8
X-Account-Object-Count: 0
Date: Tue, 15 Oct 2013 05:28:53 GMT

test
```

  * 格式化输出 Container
 
```
curl -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin?format=json
```

 如果正确，将会返回以下类似信息：

```
HTTP/1.1 200 OK
Content-Length: 86
Accept-Ranges: bytes
X-Timestamp: 1381806617.24083
X-Account-Bytes-Used: 0
X-Account-Container-Count: 2
Content-Type: application/json; charset=utf-8
X-Account-Object-Count: 0
Date: Tue, 15 Oct 2013 05:29:58 GMT

[{"count": 0, "bytes": 0, "name": "myfile"}, {"count": 0, "bytes": 0, "name": "test"}]
```

  除了JSON格式，还可以格式化XML，只需要将json改成xml 即可。

  * 查看 Container metadata

```
curl -i -X HEAD -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/test
```
  
  如果正确，将会返回以下类似信息：

```
HTTP/1.1 204 No Content
Content-Length: 0
X-Container-Object-Count: 0
Accept-Ranges: bytes
X-Timestamp: 1381806903.70007
X-Container-Bytes-Used: 0
Content-Type: text/plain; charset=utf-8
Date: Tue, 15 Oct 2013 05:32:06 GMT
```


  * 删除 Container 
  
```
curl -i -X DELETE -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/mytest
```

- Object操作
 
  * 创建 Object

```
curl -k -i -X PUT -T "apache-tomcat-6.0.36.tgz" -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/myfile/apache-tomcat-6.0.36.tgz
```	

 如果正确，将会返回以下类似信息：

```
HTTP/1.1 100 Continue

HTTP/1.1 201 Created
Last-Modified: Tue, 15 Oct 2013 05:39:07 GMT
Content-Length: 0
Etag: 3dde098fd0b3a08d3f2867e4a95591ba
Content-Type: text/html; charset=UTF-8
Date: Tue, 15 Oct 2013 05:39:08 GMT
```

  * 列出  Object

```
	curl -k -i -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/myfile
```

 如果正确，将会返回以下类似信息：

```
HTTP/1.1 200 OK
Content-Length: 25
X-Container-Object-Count: 1
Accept-Ranges: bytes
X-Timestamp: 1381814521.71796
X-Container-Bytes-Used: 6780936
Content-Type: text/plain; charset=utf-8
Date: Tue, 15 Oct 2013 05:40:43 GMT

apache-tomcat-6.0.36.tgz
```

另外 Object和 Container一样可以通过加参数来限制查询，具体示例可参考 Container操作。

  * 下载 Object

```
	curl -k -X GET -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/myfile/apache-tomcat-6.0.36.tgz > apache-tomcat-6.0.36.tgz
```

如果正确，将会返回以下类似信息：

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 6622k  100 6622k    0     0  32.8M      0 --:--:-- --:--:-- --:--:-- 33.1M
```

  * Copy Object
 
```
curl -k -i -X PUT -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' \
 -H 'X-Copy-From: /myfile/apache-tomcat-6.0.36.tgz' \
 -H 'Content-Length:0' http://192.168.30.150:8080/v1/AUTH_admin/test/apache-tomcat-6.0.36.tgz 
```

将/myfile/apache-tomcat-6.0.36.tgz 拷贝到 /test/apache-tomcat-6.0.36.tgz,如果正确，将会返回以下类似信息：

```
HTTP/1.1 201 Created
Content-Length: 0
X-Copied-From-Last-Modified: Tue, 15 Oct 2013 05:39:07 GMT
X-Copied-From: myfile/apache-tomcat-6.0.36.tgz
Last-Modified: Tue, 15 Oct 2013 05:47:52 GMT
Etag: 3dde098fd0b3a08d3f2867e4a95591ba
Content-Type: text/html; charset=UTF-8
Date: Tue, 15 Oct 2013 05:47:52 GMT
```

  * 删除 Object
 
```
	curl -k -i -X DELETE -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' http://192.168.30.150:8080/v1/AUTH_admin/test/apache-tomcat-6.0.36.tgz
```

如果正确，将会返回以下类似信息：

```
HTTP/1.1 204 No Content
Content-Length: 0
Content-Type: text/html; charset=UTF-8
Date: Tue, 15 Oct 2013 05:50:49 GMT
```
 通过之前的GET就能验证是否成功删除，此处略过。

  * 设置 Object Metadata
 
```
curl -k -i -X POST -H 'X-Auth-Token: AUTH_tk8a85916d63b14c568a4633b7920623c5' \
-H 'X-Object-Meta-Breed: apache tomcat 6.0.36' \
 http://192.168.30.150:8080/v1/AUTH_admin/myfile/apache-tomcat-6.0.36.tgz
```

如果正确，将会返回以下类似信息：

```
HTTP/1.1 202 Accepted
Content-Length: 76
Content-Type: text/html; charset=UTF-8
Date: Tue, 15 Oct 2013 05:54:09 GMT
```

通过之前的HEAD，就能查看到刚才添加的元数据

```
HTTP/1.1 200 OK
Content-Length: 6780936
X-Object-Meta-Breed: apache tomcat 6.0.36
Accept-Ranges: bytes
Last-Modified: Tue, 15 Oct 2013 05:54:08 GMT
Etag: 3dde098fd0b3a08d3f2867e4a95591ba
X-Timestamp: 1381816448.74507
Content-Type: application/x-tar
Date: Tue, 15 Oct 2013 05:55:53 GMT
```
