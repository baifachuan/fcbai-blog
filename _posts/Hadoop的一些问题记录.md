---
title: Hadoop的一些问题记录
tags: 大数据
categories: 大数据
abbrlink: 82a1eba8
date: 2021-05-31 19:31:16
---

在配置双向SSL的时候需要生成对应的私钥，出现的问题为：

```
bogon:nginx fcbai$ docker run --name some-nginx -p 8088:80 -v /Users/fcbai/software/docker/nginx/nginx.conf:/etc/nginx/nginx.conf -v /Users/fcbai/software/docker/nginx:/usr/share/nginx nginx
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2021/05/16 14:25:44 [warn] 1#1: the "ssl" directive is deprecated, use the "listen ... ssl" directive instead in /etc/nginx/nginx.conf:71
nginx: [warn] the "ssl" directive is deprecated, use the "listen ... ssl" directive instead in /etc/nginx/nginx.conf:71
2021/05/16 14:25:44 [emerg] 1#1: SSL_CTX_use_certificate("/usr/share/nginx/sslKey/server.crt") failed (SSL: error:140AB18F:SSL routines:SSL_CTX_use_certificate:ee key too small)
nginx: [emerg] SSL_CTX_use_certificate("/usr/share/nginx/sslKey/server.crt") failed (SSL: error:140AB18F:SSL routines:SSL_CTX_use_certificate:ee key too small)
```

原因：私钥长度不能设置成1024位，必须2048位，例如：

```
openssl genrsa -out root.key 2048
openssl req -new -out root.csr -key root.key
openssl x509 -req -in root.csr -out root.crt -signkey root.key -CAcreateserial -days 3650

openssl genrsa -out server.key 2048
openssl req -new -out server.csr -key server.key
openssl x509 -req -in server.csr -out server.crt -signkey server.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650



openssl genrsa -out client.key 2048
openssl genrsa -out client2.key 2048

openssl req -new -out client.csr -key client.key
openssl req -new -out client2.csr -key client2.key

openssl x509 -req -in client.csr -out client.crt -signkey client.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650
openssl x509 -req -in client2.csr -out client2.crt -signkey client2.key -CA root.crt -CAkey root.key -CAcreateserial -days 3650


openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
openssl pkcs12 -export -clcerts -in client2.crt -inkey client2.key -out client2.p12
```
如上的长度为2048。

在登陆Knox的时候出现的问题，输入admin/admin-password点击sign in报错，查看gateway.log发现报错日志如下：

```
ERROR service.knoxsso (WebSSOResource.java:getAuthenticationToken(172)) - The original URL: undefined for redirecting back after authentication is not valid according to the configured whitelist: . See documentation for KnoxSSO Whitelisting.
```

在ambari2.6版本中没有quick link的跳转，所以没有cookie带进来，需要在knox的配置项里面修改白名单Whitelisting：

```
^https?:\/\/(sandbox-hdp.hortonworks.com|localhost|127.0.0.1|0:0:0:0:0:0:0:1|::1)(:[0-9])*.*$

^https?:\/\/(localhost|127\.0\.0\.1|0:0:0:0:0:0:0:1|::1):[0-9].*$
```
用上面的第一行去替换第二行的内容。

Hive使用JDBC或者Beeline去连接Hiveserver2的时候，默认连接的是10000的端口，但是这个前提是：`hive.server2.transport.mode`这个的value是`binary`，如果使用类似knox这种代理服务通过http将其代理出去，则此时的value是http，所以默认是开启的10001端口提供的http服务，并不会开启10000端口接受thirft协议的信息。

但是实际工作种基于jdbc的binary和http的代理都会存在，因此建议的做法是开启多个hiveserver2，其中一部分提供binay一部分提供http的服务，让两者共存，通过config group的形式进行区分，针对不同的机器进行单独的配置。


使用Knox代理的形式去消费HDFS文件采用的是WebHDFS协议，使用方式为：

查看文件状态：
```
curl -i -k -u guest:guest-password -X GET \
    'https://localhost:8443/gateway/sandbox/webhdfs/v1/?op=LISTSTATUS'
```

返回值为：

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 760
Server: Jetty(6.1.26)

{"FileStatuses":{"FileStatus":[
{"accessTime":0,"blockSize":0,"group":"hdfs","length":0,"modificationTime":1350595859762,"owner":"hdfs","pathSuffix":"apps","permission":"755","replication":0,"type":"DIRECTORY"},
{"accessTime":0,"blockSize":0,"group":"mapred","length":0,"modificationTime":1350595874024,"owner":"mapred","pathSuffix":"mapred","permission":"755","replication":0,"type":"DIRECTORY"},
{"accessTime":0,"blockSize":0,"group":"hdfs","length":0,"modificationTime":1350596040075,"owner":"hdfs","pathSuffix":"tmp","permission":"777","replication":0,"type":"DIRECTORY"},
{"accessTime":0,"blockSize":0,"group":"hdfs","length":0,"modificationTime":1350595857178,"owner":"hdfs","pathSuffix":"user","permission":"755","replication":0,"type":"DIRECTORY"}
]}}
```

上传文件：

```
curl -i -k -u guest:guest-password -X PUT \
    'https://localhost:8443/gateway/sandbox/webhdfs/v1/tmp/LICENSE?op=CREATE'
```
首先通过PUT的形式获取对应的文件上传的位置，该请求会返回如下内容：

```
fcbai:~ fcbai$ curl -i -k -u guest:guest-password -X PUT \
>     'https://localhost:8443/gateway/sandbox/webhdfs/v1/tmp/LICENSE?op=CREATE'
HTTP/1.1 307 Temporary Redirect
Date: Mon, 24 May 2021 12:16:13 GMT
Set-Cookie: KNOXSESSIONID=node05cyhbral09wqvtpo2rbl796f9.node0; Path=/gateway/sandbox; Secure; HttpOnly
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: rememberMe=deleteMe; Path=/gateway/sandbox; Max-Age=0; Expires=Sun, 23-May-2021 12:16:13 GMT; SameSite=lax
Server: nginx/1.15.7
Date: Mon, 24 May 2021 12:16:13 GMT
Content-Type: application/octet-stream
Connection: keep-alive
Cache-Control: no-cache
Expires: Mon, 24 May 2021 12:16:13 GMT
Pragma: no-cache
X-FRAME-OPTIONS: SAMEORIGIN
Location: https://localhost:8443/gateway/sandbox/webhdfs/data/v1/webhdfs/v1/tmp/LICENSE?_=AAAACAAAABAAAADA5RkdHiYYGz5Ozx694RDxUOdys9CBe-CcVbEaz-4VpSNfajBHD-MyyjeIsh-_OVM6IoykCFhWD8lZO_dKgBAfBNs0N9siPI2DpDJJkRl53TWCWD1883dpG5LJ6UcP_fIx5YwDZ60j93ul4QSaRF0MDVgooGyZZKpr4xtNpBF5a-sY9bttRv-7gK3WJqeN9x3-ne1isWUuNYWXv7e8tU8Jzve0b20287g5ylSkiLCWZH0NN4ZDxaYRa_0rHIfY-Wa3hAgpKN5oQwO1Sd94TXMXUA8k7_hmBJA6
Content-Length: 0
```

其中的Location字段包含了返回的文件上传的URL地址，通过如下请求上传文件：

```
fcbai:~ fcbai$ curl -i -k -u guest:guest-password -T LICENSE -X PUT https://localhost:8443/gateway/sandbox/webhdfs/data/v1/webhdfs/v1/tmp/LICENSE?_=AAAACAAAABAAAADA5RkdHiYYGz5Ozx694RDxUOdys9CBe-CcVbEaz-4VpSNfajBHD-MyyjeIsh-_OVM6IoykCFhWD8lZO_dKgBAfBNs0N9siPI2DpDJJkRl53TWCWD1883dpG5LJ6UcP_fIx5YwDZ60j93ul4QSaRF0MDVgooGyZZKpr4xtNpBF5a-sY9bttRv-7gK3WJqeN9x3-ne1isWUuNYWXv7e8tU8Jzve0b20287g5ylSkiLCWZH0NN4ZDxaYRa_0rHIfY-Wa3hAgpKN5oQwO1Sd94TXMXUA8k7_hmBJA6
HTTP/1.1 100 Continue

HTTP/1.1 201 Created
Date: Mon, 24 May 2021 12:16:32 GMT
Set-Cookie: KNOXSESSIONID=node04gc0b5khddcuzc5qhn5bzxu010.node0; Path=/gateway/sandbox; Secure; HttpOnly
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: rememberMe=deleteMe; Path=/gateway/sandbox; Max-Age=0; Expires=Sun, 23-May-2021 12:16:32 GMT; SameSite=lax
Location: https://localhost:8443/gateway/sandbox/webhdfs/v1/tmp/LICENSE
Access-Control-Allow-Origin: *
Connection: close
```

可以在HDFS文件系统中查询到对应的文件内容：

```
[root@sandbox-hdp ~]# hadoop  fs -cat /tmp/LICENSE          
iddididididi             
```

