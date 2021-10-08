---
title: Ambari源码编译记录
tags: 编程基础
categories: 编程基础
abbrlink: 1877d8de
date: 2021-05-08 08:29:33
---

Ambari是Hadoop集群的一个管理服务，很多时候如果需要定制自己的Hadoop发行版或者开发Hadoop云服务的时候，就需要涉及到一些自定义的开发，这个时候可能就会使用到Ambari。

Ambari开发语言以Java和Py为主，同时还有大量的shell脚本，因为Ambari最终生成的是对应的机器的安装包，所以需要考虑一下自己的编译环境，工程默认提供了一个基于centos的docker编译环境，可以通过`start-build-env.sh`脚本进入环境。该脚本是将本地的环境，例如mvn，java以及源码工程目录挂载到docker容器中。

### 编译准备

源码地址：https://github.com/apache/ambari.git 

在编译之前，需要先设置参数：
```
mvn versions:set-property -Dproperty=revision -DnewVersion=2.7.5
```
因为我本次是基于master代码，最新是2.7.5的正式版，所以使用了这个版本，确定版本后，通过如下代码进行编译：
```
mvn -B clean install package rpm:rpm -DnewVersion=2.7.5 -DskipTests -Drat.skip -Dpython.ver="python >= 2.6" -Preplaceurl
```

-Drat.skip参数是为了跳过licensing 检查,防止报错Too many files with unapproved license

### 编译问题


#### Node版本问题
```
[INFO] --- exec-maven-plugin:1.2.1:exec (Bower install) @ ambari-admin ---
bower                            error Unexpected token {Stack trace:
SyntaxError: Unexpected token {
    at exports.runInThisContext (vm.js:53:16)
    at Module._compile (module.js:373:25)
    at Object.Module._extensions..js (module.js:416:10)
    at Module.load (module.js:343:32)
    at Function.Module._load (module.js:300:12)
    at Module.require (module.js:353:17)
    at require (internal/module.js:12:17)
    at Object.<anonymous> (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/bower-registry-client/node_modules/request/lib/cookies.js:3:13)
    at Module._compile (module.js:409:26)
    at Object.Module._extensions..js (module.js:416:10)Console trace:
Trace
    at StandardRenderer.error (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/lib/renderers/StandardRenderer.js:72:17)
    at Logger.<anonymous> (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/bin/bower:111:22)
    at emitOne (events.js:77:13)
    at Logger.emit (events.js:169:7)
    at Logger.emit (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/bower-logger/lib/Logger.js:29:39)
    at /usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/lib/commands/index.js:40:20
    at _rejected (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/q/q.js:797:24)
    at /usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/q/q.js:823:30
    at Promise.when (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/q/q.js:1035:31)
    at Promise.promise.promiseDispatch (/usr/local/apache-ambari-2.7.5-src/ambari-admin/src/main/resources/ui/admin-web/node_modules/bower/node_modules/q/q.js:741:41)System info:
Bower version: 1.3.8
Node version: 4.5.0
OS: Linux 3.10.0-1062.el7.x86_64 x64
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] Ambari Admin View 2.7.5.0.0 ........................ FAILURE [03:29 min]
[INFO] ambari-utility 1.0.0.0-SNAPSHOT .................... SKIPPED
[INFO] ambari-metrics 2.7.5.0.0 ........................... SKIPPED
[INFO] Ambari Metrics Common 2.7.5.0.0 .................... SKIPPED
[INFO] Ambari Metrics Hadoop Sink 2.7.5.0.0 ............... SKIPPED
[INFO] Ambari Metrics Flume Sink 2.7.5.0.0 ................ SKIPPED
[INFO] Ambari Metrics Kafka Sink 2.7.5.0.0 ................ SKIPPED
[INFO] Ambari Metrics Storm Sink 2.7.5.0.0 ................ SKIPPED
[INFO] Ambari Metrics Storm Sink (Legacy) 2.7.5.0.0 ....... SKIPPED
[INFO] Ambari Metrics Collector 2.7.5.0.0 ................. SKIPPED
[INFO] Ambari Metrics Monitor 2.7.5.0.0 ................... SKIPPED
[INFO] Ambari Metrics Grafana 2.7.5.0.0 ................... SKIPPED
[INFO] Ambari Metrics Host Aggregator 2.7.5.0.0 ........... SKIPPED
[INFO] Ambari Metrics Assembly 2.7.5.0.0 .................. SKIPPED
[INFO] Ambari Service Advisor 1.0.0.0-SNAPSHOT ............ SKIPPED
[INFO] Ambari Server 2.7.5.0.0 ............................ SKIPPED
[INFO] Ambari Functional Tests 2.7.5.0.0 .................. SKIPPED
[INFO] Ambari Agent 2.7.5.0.0 ............................. SKIPPED
[INFO] ambari-logsearch 2.7.5.0.0 ......................... SKIPPED
[INFO] Ambari Logsearch Appender 2.7.5.0.0 ................ SKIPPED
[INFO] Ambari Logsearch Config Api 2.7.5.0.0 .............. SKIPPED
[INFO] Ambari Logsearch Config JSON 2.7.5.0.0 ............. SKIPPED
[INFO] Ambari Logsearch Config Solr 2.7.5.0.0 ............. SKIPPED
[INFO] Ambari Logsearch Config Zookeeper 2.7.5.0.0 ........ SKIPPED
[INFO] Ambari Logsearch Config Local 2.7.5.0.0 ............ SKIPPED
[INFO] Ambari Logsearch Log Feeder Plugin Api 2.7.5.0.0 ... SKIPPED
[INFO] Ambari Logsearch Log Feeder Container Registry 2.7.5.0.0 SKIPPED
[INFO] Ambari Logsearch Log Feeder 2.7.5.0.0 .............. SKIPPED
[INFO] Ambari Logsearch Web 2.7.5.0.0 ..................... SKIPPED
[INFO] Ambari Logsearch Server 2.7.5.0.0 .................. SKIPPED
[INFO] Ambari Logsearch Assembly 2.7.5.0.0 ................ SKIPPED
[INFO] Ambari Logsearch Integration Test 2.7.5.0.0 ........ SKIPPED
[INFO] ambari-infra 2.7.5.0.0 ............................. SKIPPED
[INFO] Ambari Infra Solr Client 2.7.5.0.0 ................. SKIPPED
[INFO] Ambari Infra Solr Plugin 2.7.5.0.0 ................. SKIPPED
[INFO] Ambari Infra Manager 2.7.5.0.0 ..................... SKIPPED
[INFO] Ambari Infra Assembly 2.7.5.0.0 .................... SKIPPED
[INFO] Ambari Infra Manager Integration Tests 2.7.5.0.0 ... SKIPPED
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  03:30 min
[INFO] Finished at: 2020-08-05T15:50:13+08:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.2.1:exec (Bower install) on project ambari-admin: Command execution failed. Process exited with an error: 1 (Exit value: 1) -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
```

碰到这个问题是因为代码里面指定的node版本和环境设置的版本不一致，可以由多种解决办法，我使用的最简单的一种就是修改`ambari-admin`工程里面的`exec-maven-plugin`插件，将：

```
<argument>${basedir}/src/main/resources/ui/admin-web/node_modules/bower/bin/bower</argument>
```
替换成：
```
<argument>bower</argument>
```
也可以修改pom文件中的`nodeVersion`和`npmVersion`，查看环境中的版本号，例如做如下配置：

```
<!--修改ambari-admin模块下的pom.xml文件 -->
<nodeVersion>3.10.10</nodeVersion>
<npmVersion>v6.17.1</npmVersion>
```
这样的话也能解决问题。


#### 下载路径问题

Ambari的监控部分需要依赖Hbase等组件作为存储，默认会使用HDP的官方网站的地址进行下载，因为该地址已经失效，我就将其修改为了Apache的下载地址，修改`ambari-metrics-timelineservice`工程下的pom文件中的properties部分：

```
    <!--TODO change to HDP URL-->
    <hbase.tar>https://archive.apache.org/dist/hbase/2.0.2/hbase-2.0.2-bin.tar.gz</hbase.tar>
    <hbase.folder>hbase-2.0.2.3.1.4.0-315</hbase.folder>
    <hadoop.tar>https://archive.apache.org/dist/hadoop/core/hadoop-3.1.1/hadoop-3.1.1.tar.gz</hadoop.tar>
    <hadoop.folder>hadoop-3.1.1.3.1.4.0-315</hadoop.folder>
    <grafana.folder>grafana-6.4.2</grafana.folder>
    <grafana.tar>https://dl.grafana.com/oss/release/grafana-6.4.2.linux-amd64.tar.gz</grafana.tar>
    <phoenix.tar>https://downloads.apache.org/phoenix/apache-phoenix-5.0.0-HBase-2.0/bin/apache-phoenix-5.0.0-HBase-2.0-bin.tar.gz</phoenix.tar>
    <phoenix.folder>phoenix-5.0.0.3.1.4.0-315</phoenix.folder>
```

将对应的key修改为上面的内容，即可下载对应的二进制包。

同时因为Ambari默认会使用HDP的maven rep去下载hbase，phoenix等依赖的jar，该maven rep里面也下架了相关的依赖，所以需要修改对应的版本让其默认从apache maven rep中进行下载：

```
  <properties>
    <!-- Needed for generating FindBugs warnings using parent pom -->
    <!--<yarn.basedir>${project.parent.parent.basedir}</yarn.basedir>-->
    <protobuf.version>2.5.0</protobuf.version>
    <hadoop.version>3.1.1</hadoop.version>
    <phoenix.version>5.0.0-HBase-2.0</phoenix.version>
    <hbase.version>2.0.2</hbase.version>
  </properties>
```
调整对应依赖的版本如上图所示，即可，中间大部分问题都会聚焦在版本的对应上，最后编译成功：

```
[INFO] Reactor Summary:
[INFO] 
[INFO] Ambari Main ........................................ SUCCESS [  7.942 s]
[INFO] Apache Ambari Project POM .......................... SUCCESS [  0.360 s]
[INFO] Ambari Web ......................................... SUCCESS [03:20 min]
[INFO] Ambari Views ....................................... SUCCESS [  2.930 s]
[INFO] Ambari Admin View .................................. SUCCESS [01:03 min]
[INFO] ambari-utility ..................................... SUCCESS [  7.990 s]
[INFO] ambari-metrics ..................................... SUCCESS [  1.056 s]
[INFO] Ambari Metrics Common .............................. SUCCESS [ 14.867 s]
[INFO] Ambari Metrics Hadoop Sink ......................... SUCCESS [03:12 min]
[INFO] Ambari Metrics Flume Sink .......................... SUCCESS [03:01 min]
[INFO] Ambari Metrics Kafka Sink .......................... SUCCESS [02:57 min]
[INFO] Ambari Metrics Storm Sink .......................... SUCCESS [  6.595 s]
[INFO] Ambari Metrics Storm Sink (Legacy) ................. SUCCESS [  7.187 s]
[INFO] Ambari Metrics Collector ........................... SUCCESS [14:11 min]
[INFO] Ambari Metrics Monitor ............................. SUCCESS [  3.587 s]
[INFO] Ambari Metrics Grafana ............................. SUCCESS [04:53 min]
[INFO] Ambari Metrics Host Aggregator ..................... SUCCESS [56:29 min]
[INFO] Ambari Metrics Assembly ............................ SUCCESS [33:47 min]
[INFO] Ambari Service Advisor ............................. SUCCESS [ 13.602 s]
[INFO] Ambari Server ...................................... SUCCESS [  01:20 h]
[INFO] Ambari Functional Tests ............................ SUCCESS [ 14.766 s]
[INFO] Ambari Agent ....................................... SUCCESS [06:55 min]
[INFO] ambari-logsearch ................................... SUCCESS [  0.665 s]
[INFO] Ambari Logsearch Appender .......................... SUCCESS [  7.123 s]
[INFO] Ambari Logsearch Config Api ........................ SUCCESS [  3.373 s]
[INFO] Ambari Logsearch Config JSON ....................... SUCCESS [  2.865 s]
[INFO] Ambari Logsearch Config Solr ....................... SUCCESS [  2.571 s]
[INFO] Ambari Logsearch Config Zookeeper .................. SUCCESS [  5.323 s]
[INFO] Ambari Logsearch Config Local ...................... SUCCESS [  3.058 s]
[INFO] Ambari Logsearch Log Feeder Plugin Api ............. SUCCESS [ 13.483 s]
[INFO] Ambari Logsearch Log Feeder Container Registry ..... SUCCESS [  4.152 s]
[INFO] Ambari Logsearch Log Feeder ........................ SUCCESS [02:15 min]
[INFO] Ambari Logsearch Web ............................... SUCCESS [17:46 min]
[INFO] Ambari Logsearch Server ............................ SUCCESS [05:05 min]
[INFO] Ambari Logsearch Assembly .......................... SUCCESS [  9.715 s]
[INFO] Ambari Logsearch Integration Test .................. SUCCESS [01:36 min]
[INFO] ambari-infra ....................................... SUCCESS [  2.566 s]
[INFO] Ambari Infra Solr Client ........................... SUCCESS [01:06 min]
[INFO] Ambari Infra Solr Plugin ........................... SUCCESS [01:20 min]
[INFO] Ambari Infra Manager ............................... SUCCESS [03:56 min]
[INFO] Ambari Infra Assembly .............................. SUCCESS [  2.229 s]
[INFO] Ambari Infra Manager Integration Tests ............. SUCCESS [01:30 min]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 04:07 h
[INFO] Finished at: 2021-05-07T16:39:14+00:00
[INFO] Final Memory: 400M/1073M
[INFO] ------------------------------------------------------------------------
centos7:~/src $ 

```

找到对应的包进行安装就行。

安装包在`~/src/ambari-server/target/rpm/ambari-server/RPMS/x86_64`目录下，使用` yum install ambari-server*.rpm
`进行安装，进行安装的时候可能会碰到：
```
centos7:~/src/ambari-server/target/rpm/ambari-server/RPMS/x86_64 $ yum install ambari-server*.rpm
Loaded plugins: fastestmirror, ovl
ovl: Error while doing RPMdb copy-up:
[Errno 13] Permission denied: '/var/lib/rpm/Triggername'
You need to be root to perform this command.
centos7:~/src/ambari-server/target/rpm/ambari-server/RPMS/x86_64 $ sudo yum install ambari-server*.rpm
bash: sudo: command not found
```
安装权限不够，sudo也不存在，这时候可以使用root的权限重新进入容器：
```
docker exec -u root -i -t 64814ad3cd7b /bin/bash
```
再进行安装即可。

上面安装的是ambari server，还需安装agent，对应的包在ambari-agent/target/rpm/ambari-agent/RPMS/x86_64/ambari-agent-2.6.2.0-0.x86_64.rpm下，安装同理。

