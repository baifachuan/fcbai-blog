---
title: 一次无缘无故的运行态下的ClassNotFound问题
tags: 编程基础
categories: 编程基础
abbrlink: ca281eba
date: 2021-10-12 13:58:30
---

spring的jar默认会把所有依赖打进jar包里面，也就是fit jar，这样会造成整个包非常大，不利于依赖管理，发布等操作。特别是像这种数据服务项目会依赖很多hadoop的sdk，轻轻松松上500MB+，所以不得不说java发展到现在，确实过于臃肿了，使用go去做gateway也确实是一个很好的选项。

为了解决发布包过大的问题，我把工程的发布做了修改，分离了发布包与依赖项，也就是最后发布出来的并不是一个单纯的jar包，而是一个目录，大概如下：

```shell
root/
	xxxx.jar
	lib
	config
	application.yaml
```
root是根目录，jar是真正的业务代码的内容，所有的第三方依赖全部丢到了lib里面，然后把lib下的所有依赖写入MANIFEST.MF，达到在运行的classpath中可以找到的效果，配置统一归类到config目录，这样的好处是在不涉及引入新的依赖或者依赖更新的情况下，只需要更新业务的内容就行，而这个jar通常只有几百KB，从CI/CD或者测试来说效率非常高。

这样运行一直没有什么问题，但是某一天团队突然碰到一个问题，在程序启动的时候出现Class NotFound的错误，按理说这个错误实在不应该，因为这一段时间都没有涉及过依赖的修改，所有的依赖都在lib里面，仅仅修改的是业务的代码。

然后看NotFound的Class是一个公司内部的SDK的class，这个SDK安安静静地躺在lib目录下没有发生过任何变动，但是为什么会发生NotFound的错误呢，错误肯定毋庸置疑就是jvm在加载的时候所有类加载器都没有找到这个class，首先依赖肯定在，其次为何只有这一个依赖找不到，我能肯定是在运行的时候classpath下没有找到这个第三方jar，于是我使用`jar xvf xxx.jar`解压后打开META-INF下的MANIFEST.MF看了下，发现里面的jar的名字是XXXX-0.0.1-public-20211009.121515-24.jar，再和lib下的一对比发现后面的时间戳貌似发生了变化，lib下的public-后面的时间戳比这个小。

于是问题肯定找到了，就是lib下的jar和打包写入到MANIFEST.MF的jar的名字不对，造成运行的classpath的时候找不到，因此抛出了Class Not Found的错误，找到问题了就得看原因是什么。

因为lib更新频率低，而业务jar更新频率很快，每一次业务jar打包都会从maven里面拉取依赖的jar写入MANIFEST.MF，但是并不会实际打包第三方jar，这个时候如果maven里面的jar发生了变动，但是maven的版本号又没有发生修改，那么就可能出现这个问题。

再看了一下maven引入该包的dependency，是使用的`0.0.1-public-SNAPSHOT`版本，于是原因大概就推断出来了，因为该SDK的发布者并没有严格遵循jar发布的版本管理规则，或者说因为是SHNAPSHOT，所以就默认是最新的Jar，导致了这个问题的产生，我们的maven里面引用的是0.0.1-public-SNAPSHOT的版本，只要不升级这个版本，我们lib目录下的jar就会去更新，而如果这个时候SDK的发布者不断的往nexus仓库推送新的jar，版本号依旧是0.0.1-public-SNAPSHOT，但是jar的名字发生修改，就像后面加一个当前时间戳，这样就会造成放入lib下的jar和maven plugin在打包的时候写入`MANIFEST.MF`的名字不一致，造成运行的classpath的时候找不到这个class。


附录：maven plugin

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
                            <mainClass>com.bytedance.emr.flowagent.server.FlowAgentServerApplication</mainClass>
                        </manifest>
                    </archive>
                    <!--资源文件不打进jar包中，做到配置跟项目分离的效果-->
                    <excludes>
                        <!-- 业务jar中过滤application.properties/yml文件，在jar包外控制 -->
                        <exclude>*.properties</exclude>
                        <exclude>*.xml</exclude>
                        <exclude>*.yml</exclude>
                    </excludes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-antrun-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy-jar-file</id>
                        <phase>package</phase>
                        <goals>
                            <goal>run</goal>
                        </goals>
                        <configuration>
                            <target>
                                <copy todir="${project.build.directory}/config">
                                    <fileset dir="${project.basedir}/src/main/resources">
                                        <include name="*" />
                                        <include name="*/*" />
                                        <include name="*/*/*" />
                                    </fileset>
                                </copy>
                            </target>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```
