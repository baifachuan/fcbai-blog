---
title: Lombok对jdk.compiler内部软件包的访问与Java-16不兼容
tags: 编程基础
categories: 编程基础
abbrlink: 89e45f73
date: 2021-08-07 21:17:16
---

lombok是很常用的一个工具，老实说近几年都没有怎么关注过jdk新版本的问题，一直脑子停留在JDK8的lambda特性，之后就几乎一无所知了，也不怎么去了解，使用最多的jdk版本也就停留在java 12了。

最近换了个工作后，将jdk升级到了16，然后再继续使用lombok下发生了一些问题，在mvn编译的时候就会出错，错误内容为：

```
Caused by: java.lang.IllegalAccessError: class lombok.javac.apt.LombokProcessor (in unnamed module @0x4e670245) cannot access class com.sun.tools.javac.processing.JavacProcessingEnvironment (in module jdk.compiler) because module jdk.compiler does not export com.sun.tools.javac.processing to unnamed module @0x4e670245
    at lombok.javac.apt.LombokProcessor.getJavacProcessingEnvironment (LombokProcessor.java:433)
    at lombok.javac.apt.LombokProcessor.init (LombokProcessor.java:92)
    at lombok.core.AnnotationProcessor$JavacDescriptor.want (AnnotationProcessor.java:160)
    at lombok.core.AnnotationProcessor.init (AnnotationProcessor.java:213)
    at lombok.launch.AnnotationProcessorHider$AnnotationProcessor.init (AnnotationProcessor.java:64)
    at com.sun.tools.javac.processing.JavacProcessingEnvironment$ProcessorState.<init> (JavacProcessingEnvironment.java:702)
    at com.sun.tools.javac.processing.JavacProcessingEnvironment$DiscoveredProcessors$ProcessorStateIterator.next (JavacProcessingEnvironment.java:829)

```
可以看到是访问java内部class的时候出的问题，这个问题是因为Lombok正在使用反射来访问内部JDK API，在以前的Java版本中，这会导致警告消息，而在jdk16的版本中是直接抛出了错误。

通常，在运行Java时，可以通过在运行--add-opens=<module>/<package>=<accessing module>时将java指令作为VM参数传递来显式打开内部JDK包以进行反射。在这种情况下，需要将这些指令传递给调用java时运行的javac进程。这可以通过将传递给javac的选项加上-J前缀来实现，该选项会将其传递给底层JVM。

关于这个具体的问题在：[JEP 396: Strongly Encapsulate JDK Internals by Default](https://openjdk.java.net/jeps/396)

里面有更具体的描述。

从解决来说，如果不是高版本的jdk当然没问题，降低版本肯定能解决，不过这样当然是治标不治本，实际上可以使用maven的plugin来解决这个问题：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version>
    <configuration>
        <source>16</source>
        <target>16</target>
        <!--                    <release>16</release>-->
        <fork>true</fork>
        <compilerArgs>
            <arg>--enable-preview</arg>
            <arg>-Xlint:all</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.model=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.processing=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED</arg>
            <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.jvm=ALL-UNNAMED</arg>
        </compilerArgs>
        <!--for unmappable characters in classes-->
        <encoding>UTF-8</encoding>
        <showDeprecation>true</showDeprecation>
        <showWarnings>true</showWarnings>
        <!--for lombok annotations to resolve-->
        <!--contradictory to maven, intelliJ fails with this-->
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.16</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

使用配置中的<compilerArgs>元素传递所需选项的位置。
请注意，我在选项前面添加了-J，以便将它们传递给运行javac的JVM，而不是javac选项。
在问题中列出的--add-opens指令之上，还有一个附加的指令:

```
-J--add-opens=jdk.compiler/com.sun.tools.javac.jvm=ALL-UNNAMED
```
还需要` <fork>true</fork> `，因为否则会忽略-J选项(从mvn clean install -X的输出判断)。查看Maven文档，使用fork似乎随时都需要将true设置为<compilerArgs>。

具体的解释可以参考这个链接：
[https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#compilerArgs](https://maven.apache.org/plugins/maven-compiler-plugin/compile-mojo.html#compilerArgs)
> <compilerArgs> Sets the arguments to be passed to the compiler if fork is set to true.

这个问题在stackoverflow里面也有人问到：
[https://stackoverflow.com/questions/65380359/](https://stackoverflow.com/questions/65380359/)

从版本来说，版本升级总是会带来一些新的特性，当然，自然也会引入一些新的兼容的调整，所以偶尔还是需要关注一下新版本特性的内容。
