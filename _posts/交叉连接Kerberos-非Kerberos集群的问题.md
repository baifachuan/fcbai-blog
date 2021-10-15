---
title: 交叉连接Kerberos/非Kerberos集群的问题
date: 2021-10-15 11:58:20
tags: 编程基础
categories: 编程基础
---

场景：一个Spring的Web服务中提供了一个能力，可以使用Java API去读取多个不同集群的HDFS上的文件，而多个HDFS集群，有的可能开启了Kerberos，有的则可能没有开启。

从写代码来说，可能会见到这样的代码：

```java
public void readFile() {
    Configuration conf = new Configuration();
    conf.set("fs.defaultFS", "hdfs://xxxxx:8020");
    if (isKerberosEnable()) {
        System.setProperty("java.security.krb5.conf", "/etc/krb5.conf");
        conf.set("hadoop.security.authentication", "Kerberos");

        UserGroupInformation.setConfiguration(conf);
        UserGroupInformation.loginUserFromKeytab("username", "user.keytab");
    }

    FileSystem fileSystem = FileSystem.get(conf);
}
```

如果开启了Kerberos则使用用户身份的keytab创建fs对象，如果没有则直接创建fs对象，看起来好像没有什么问题，但是实际会有这样的问题：

* 如果Spring的Web服务启动的时候，连接未开启Kerberos的集群，连接成功，再连接开启Kerberos的集群，出错。
* 如果Spring的Web服务启动的时候，再连接开启Kerberos的集群，连接成功，再连接未开启Kerberos的集群，出错。

也就是第一次肯定会成功，第二次请求如果出现集群切换到不同类型，从开启Kerberos的集群切换到非Kerberos集群就会出错。
```xml
Caused by: java.io.IOException: Server asks us to fall back to SIMPLE auth, but this client is configured to only allow secure connections.
```
大概的错误是客户端配置了kerberos但是服务端没有开启kerberos，看起来很神奇，从现象来看肯定是多次连接过程中，配置交叉了，但是从代码来看，每一次请求都是在方法内全新创建的连接，全新创建的conf对象，全新创建的fs对象。

继续往里面看会发现fs在创建的时候有一个方法：

```java
    public static FileSystem get(URI uri, Configuration conf) throws IOException {
        String scheme = uri.getScheme();
        String authority = uri.getAuthority();
        if (scheme == null && authority == null) {
            return get(conf);
        } else {
            if (scheme != null && authority == null) {
                URI defaultUri = getDefaultUri(conf);
                if (scheme.equals(defaultUri.getScheme()) && defaultUri.getAuthority() != null) {
                    return get(defaultUri, conf);
                }
            }

            String disableCacheName = String.format("fs.%s.impl.disable.cache", scheme);
            if (conf.getBoolean(disableCacheName, false)) {
                LOGGER.debug("Bypassing cache to create filesystem {}", uri);
                return createFileSystem(uri, conf);
            } else {
                return CACHE.get(uri, conf);
            }
        }
    }
```

可以发现hadoop的客户端sdk里面是把fs给缓存了，也就是无论自己的业务代码是否全新创建，fs的对象会在sdk里面被缓存：
```java
static final FileSystem.Cache CACHE = new FileSystem.Cache();
```
知道是缓存引起的问题，解决办法就好解决了，要么在if后面新增else，重新刷新认证的值：

```java
conf.set("hadoop.security.authentication", "Simple");
```

要么关闭缓存，也就是设置：
```xml
fs.hdfs.impl.disable.cache=true
```

都能解决问题。
