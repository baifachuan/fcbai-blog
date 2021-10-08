---
title: 使用Knox做http代理代理Tomcat服务器
tags: 大数据
categories: 大数据
abbrlink: 73f60aa1
date: 2021-05-31 19:30:17
---

Knox可以代理任何http服务或者web app,不仅仅局限于代理hadoop的服务或者web应用/界面。
例如，你可以把tomcat装在一台机器上，然后把knox装在另一台机器上或者跟tomcat相同的机器上。knox就可以作为代理，作为通向tomcat的访问点。

* 下载并安装knox 0.6.0或者以上版本。
http://mirror.bit.edu.cn/apache/knox/0.6.0/knox-0.6.0.zip
解压后，先运行knox/bin/knoxcli create-master 来创建密码,再运行knox/bin/ldap start 启动ldap，最后运行knox/bin/gateway来启动knox服务器.
* 下载并安装Tomcat,最新版本或者以前版本都可以.(需要注意的是tomcat9需要java8以上版本)
http://apache.fayea.com/tomcat/tomcat-9/v9.0.0.M1/bin/apache-tomcat-9.0.0.M1.zip
解压后，到tomcat/bin目录运行startup [Windows运行startup.bat ,Linux运行startup.sh]
tomcat默认端口号是8080,如果要修改可到tomcat/conf/server.xml中找到Connector port=”8080”这一行将8080改为其他值.
* 创建一个tomcat.xml放于knox/conf/topologies目录下，并具有以下内容(这样一来tomcat将被视作一个集群的名字，同理如果在knox/conf/topologies目录下放置一个mycluster.xml，那么mycluster也被看做是一个集群的名字,会在knox/data/deployment生成其最新的war包)


```xml
<?xml version="1.0" ?>
<topology>
<gateway>
   <provider>
      <role>authentication</role>
      <name>ShiroProvider</name>
      <enabled>true</enabled>
      <param>
         <name>sessionTimeout</name>
         <value>30</value>
      </param>
     <param>
        <name>main.ldapRealm</name>
        <value>org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm</value>
     </param>
      <param>
        <name>main.ldapRealm.userDnTemplate</name>
        <value>uid={0},ou=people,dc=hadoop,dc=apache,dc=org</value>
      </param>
      <param>
          <name>main.ldapRealm.contextFactory.url</name>
          <value>ldap://localhost:33389</value>
     </param>
     <param>
       <name>main.ldapRealm.contextFactory.authenticationMechanism</name>
       <value>simple</value>
    </param>
    <param>
       <name>urls./**</name>
       <value>authcBasic</value>
    </param>
</provider>
<provider>
   <role>identity-assertion</role>
   <name>Default</name>
   <enabled>true</enabled>
</provider>
</gateway>

  <service> 
     <role>TOMCAT</role> 
     <url>http://localhost:8080</url> 
 </service>
</topology>
```

创建一个tomcat/9.0目录层级放于knox/data/services目录下.

再创建一个service.xml具有以下内容，放于knox/data/services/tomcat/9.0目录下.

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<service role="TOMCAT" name="tomcat" version="9.0">
   <routes>
     <route path="/tomcatui/">
     </route>

     <route path="/tomcatui/**">
     </route>

     <route path="/tomcatui/**?**"> 
     </route>

   </routes>
</service>
```

再创建一个rewrite.xml具有以下内容，放于knox/data/services/tomcat/9.0目录下
rewrite.xml:

```
<rules>
<!-- Inbound  rewrite rules   -->
<rule dir="IN" name="TOMCAT/root/inbound" pattern="*://*:*/**/tomcatui/">
   <rewrite template="{$serviceUrl[TOMCAT]}/"/>
</rule>

<rule dir="IN" name="TOMCAT/path/inbound" pattern="*://*:*/**/tomcatui/{**}">
    <rewrite template="{$serviceUrl[TOMCAT]}/{**}"/>
</rule>

<rule dir="IN" name="TOMCAT/full/inbound" pattern="*://*:*/**/tomcatui/{**}?{**}">
        <rewrite template="{$serviceUrl[TOMCAT]}/{**}?{**}"/>
</rule>
<rules>
```

重启knox服务器（先运行knox/bin/gateway stop 再运行knox/bin/gateway start）
若更改service.xml 或rewrite.xml 则执行 ./knoxcli.sh redeploy –cluster tomcat
或删除data/deployments里所有内容，重启knox服务器

访问https://localhost:8443/gateway/tomcat/tomcatui
用浏览器访问时(比如firefox)，将网址添加到例外（表示这是个受信任的网站），要求用户名和密码时输入guest和guest-password
可以看到它的内容， 等同于访问http://localhost:8080/.

备注： https://{knox-host}:{knox-port}/gateway/tomcat/tomcatui中tomcat会被看做集群名字，/tomcatui会被看做service.xml里的根路径。rewrite.xml中IN类型的rule中的pattern是针对呈献给用户的最终url比如https://{knox-host}:{knox-port}/gateway/tomcat/tomcatui

也可以利用curl用如下命令访问，看到其结果: 
```
curl -i -k -u guest:guest-password -X GET https://localhost:8443/gateway/tomcat/tomcatui
```

至此，Knox代理Tomcat即可成功。
