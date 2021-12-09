---
title: SpringBoot项目业务代码与库打包分离
tags: 编程基础
categories: 编程基础
abbrlink: f1bab461
date: 2021-12-09 20:30:01
---

SpringBoot是一套脚手架，默认打包方式会把依赖打进最终的包里面，所以执行`java -jar xxx.jar`就可以很方便的运行，但是这样的问题是造成整个jar的体积变得非常大。

即便是有完整的ci/cd的情况下，在做deploy的时候会也会发生较大的文件传输，像如果是做数据服务类场景，会引入大量hadoop生态圈的client，最终的jar轻松上500MB+，造成整个ci的周期很长。

仔细看大部分的标准二进制发现版，他们的程序结构其实是这样的：

```
/app
    /bin
    /sbin
    /lib
    /config 
```
这样的目录看起来就很整洁，那么有没有办法把spring的项目打包最终的结构变成这样呢：
```
/app
    /bin
    /lib
    /config
    你的业务jar
```
config放配置文件，lib放第三方依赖，包括pom的依赖，bin写一些启动脚本，这样的话业务jar通常就只有几十kb了，这样的话在做ci的时候可以通过变量控制，只需要传递几十kb就好了，速度会很快。

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>${project.parent.version}</version>
    <configuration>
        <!--表示编译版本配置有效-->
        <fork>true</fork>
        <!--引入第三方jar包时,不添加则引入的第三方jar不会被打入jar包中-->
        <includeSystemScope>true</includeSystemScope>
        <!--排除第三方jar文件-->
        <includes>
            <include>
                <groupId>nothing</groupId>
                <artifactId>nothing</artifactId>
            </include>
        </includes>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<!-- 2、完成对Java代码的编译，可以指定项目源码的jdk版本，编译后的jdk版本，以及编码 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <!-- 源代码使用的JDK版本 -->
        <source>${java.version}</source>
        <!-- 需要生成的目标class文件的编译版本 -->
        <target>${java.version}</target>
        <!-- 字符集编码 -->
        <encoding>UTF-8</encoding>
        <!-- 用来传递编译器自身不包含但是却支持的参数选项 -->
        <compilerArguments>
            <verbose/>
            <bootclasspath>${java.home}/lib/rt.jar:${java.home}/lib/jce.jar</bootclasspath>
        </compilerArguments>
    </configuration>
</plugin>

<!-- 3、将所有依赖的jar文件复制到target/lib目录 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>prepare-package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <!--复制到哪个路径，${project.build.directory} 缺醒为 target，其他内置参数见下面解释-->
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
                <overWriteReleases>false</overWriteReleases>
                <overWriteSnapshots>false</overWriteSnapshots>
                <overWriteIfNewer>true</overWriteIfNewer>
            </configuration>
        </execution>
    </executions>
</plugin>

<!-- 4、指定启动类，指定配置文件，将依赖打成外部jar包 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <!-- 是否要把第三方jar加入到类构建路径 -->
                <addClasspath>true</addClasspath>
                <!-- 外部依赖jar包的最终位置 -->
                <classpathPrefix>lib/</classpathPrefix>
                <!-- 项目启动类 -->
                <mainClass>xxx.xxx.xxxx.XXX</mainClass>
            </manifest>
        </archive>
        <!--资源文件不打进jar包中，做到配置跟项目分离的效果-->
        <excludes>
            <!-- 业务jar中过滤application.properties/yml文件，在jar包外控制 -->
            <exclude>application.yml</exclude>
        </excludes>
    </configuration>
</plugin>
```
从功能上来说，maven有很好的插件，可以直接通过配置上面的插件，达到想要的效果。
