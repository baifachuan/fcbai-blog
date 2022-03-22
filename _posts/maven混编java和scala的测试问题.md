---
title: maven混编java和scala的测试问题
tags: 编程基础
categories: 编程基础
abbrlink: 667d49f3
date: 2022-03-02 11:12:50
---

有个工程的代码主要涉及到java和scala，有部分py代码不过可以忽略，主要就是java和scala，在最开始构建工程的时候我本来想用sbt，但是考虑到sbt的单线程构建速度，以及普世度，最后还是使用的mave。

maven的插件本身是支持scala的配置，没有什么大问题，通过新增如下插件：

```
<plugin>
    <groupId>org.scala-tools</groupId>
    <artifactId>maven-scala-plugin</artifactId>
    <version>2.15.2</version>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>testCompile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
和相关依赖即可。

```
<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-library</artifactId>
    <version>${scala.version}</version>
</dependency>

<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-compiler</artifactId>
    <version>${scala.version}</version>
</dependency>

<dependency>
    <groupId>org.scala-lang</groupId>
    <artifactId>scala-reflect</artifactId>
    <version>${scala.version}</version>
</dependency>
```


主要的麻烦是在测试上，工程中我引入了单测和测试覆盖率报告的生成，但是执行mvn test的时候并不会执行scala的单测。

通过新增如下plugin：

```
<build>
    <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-surefire-plugin</artifactId>
          <version>2.7</version>
          <configuration>
            <skipTests>true</skipTests>
          </configuration>
        </plugin>
        <plugin>
          <groupId>org.scalatest</groupId>
          <artifactId>scalatest-maven-plugin</artifactId>
          <version>2.0.2</version>
          <configuration>
            <reportsDirectory>${project.build.directory}/surefire-reports</reportsDirectory>
            <junitxml>.</junitxml>
            <filereports>WDF TestSuite.txt</filereports>
          </configuration>
          <executions>
            <execution>
              <id>test</id>
              <goals>
                <goal>test</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
    </plugins>
</build>
```

这个插件工程的地址在：https://github.com/scalatest/scalatest-maven-plugin

如此新增后通过工程的root目录执行mvn test能过调用scala的测试，但是我的工程是java和scala混搭，也就是scala子模块里面引用了java工程，因此直接进入scala的子工程执行mvn test的时候会出错，报错引入的java 类 not found。


