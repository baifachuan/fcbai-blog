---
title: Android Jetpack Compose碰到的kotlin兼容性问题
tags: 编程基础
categories: 编程基础
abbrlink: 7e30a477
date: 2021-03-15 19:00:11
---

安卓推出了新一代UI框架Jetpack Compose，目前需要下载新版本的Android Studio，我在构建工程的时候碰到如下问题：
```
java.lang.NoSuchMethodError: 'void org.jetbrains.kotlin.serialization.DescriptorSerializer$Companion.registerSerializerPlugin(org.jetbrains.kotlin.serialization.DescriptorSerializerPlugin)'
	at androidx.compose.compiler.plugins.kotlin.ComposeComponentRegistrar$Companion.registerProjectExtensions(ComposePlugin.kt:189)
	at androidx.compose.compiler.plugins.kotlin.ComposeComponentRegistrar.registerProjectComponents(ComposePlugin.kt:119)
	at org.jetbrains.kotlin.cli.jvm.compiler.KotlinCoreEnvironment$Companion.registerExtensionsFromPlugins$cli(KotlinCoreEnvironment.kt:575)
	at org.jetbrains.kotlin.cli.jvm.compiler.KotlinCoreEnvironment$ProjectEnvironment.registerExtensionsFromPlugins(KotlinCoreEnvironment.kt:129)
	at org.jetbrains.kotlin.cli.jvm.compiler.KotlinCoreEnvironment.<init>(KotlinCoreEnvironment.kt:169)
	at org.jetbrains.kotlin.cli.jvm.compiler.KotlinCoreEnvironment.<init>(KotlinCoreEnvironment.kt:108)
	at org.jetbrains.kotlin.cli.jvm.compiler.KotlinCoreEnvironment$Companion.createForProduction(KotlinCoreEnvironment.kt:421)
	at org.jetbrains.kotlin.cli.jvm.K2JVMCompiler.createCoreEnvironment(K2JVMCompiler.kt:226)
	at org.jetbrains.kotlin.cli.jvm.K2JVMCompiler.doExecute(K2JVMCompiler.kt:152)
	at org.jetbrains.kotlin.cli.jvm.K2JVMCompiler.doExecute(K2JVMCompiler.kt:52)
	at org.jetbrains.kotlin.cli.common.CLICompiler.execImpl(CLICompiler.kt:88)
	at org.jetbrains.kotlin.cli.common.CLICompiler.execImpl(CLICompiler.kt:44)
	at org.jetbrains.kotlin.cli.common.CLITool.exec(CLITool.kt:98)
	at org.jetbrains.kotlin.incremental.IncrementalJvmCompilerRunner.runCompiler(IncrementalJvmCompilerRunner.kt:386)
	at org.jetbrains.kotlin.incremental.IncrementalJvmCompilerRunner.runCompiler(IncrementalJvmCompilerRunner.kt:110)
	at org.jetbrains.kotlin.incremental.IncrementalCompilerRunner.compileIncrementally(IncrementalCompilerRunner.kt:282)
	at org.jetbrains.kotlin.incremental.IncrementalCompilerRunner.access$compileIncrementally(IncrementalCompilerRunner.kt:42)
	at org.jetbrains.kotlin.incremental.IncrementalCompilerRunner$compileImpl$2.invoke(IncrementalCompilerRunner.kt:98)
	at org.jetbrains.kotlin.incremental.IncrementalCompilerRunner.compileImpl(IncrementalCompilerRunner.kt:110)
	at org.jetbrains.kotlin.incremental.IncrementalCompilerRunner.compile(IncrementalCompilerRunner.kt:73)
	at org.jetbrains.kotlin.daemon.CompileServiceImplBase.execIncrementalCompiler(CompileServiceImpl.kt:607)
	at org.jetbrains.kotlin.daemon.CompileServiceImplBase.access$execIncrementalCompiler(CompileServiceImpl.kt:96)
	at org.jetbrains.kotlin.daemon.CompileServiceImpl.compile(CompileServiceImpl.kt:1649)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:566)
	at java.rmi/sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:359)
	at java.rmi/sun.rmi.transport.Transport$1.run(Transport.java:200)
	at java.rmi/sun.rmi.transport.Transport$1.run(Transport.java:197)
	at java.base/java.security.AccessController.doPrivileged(Native Method)
	at java.rmi/sun.rmi.transport.Transport.serviceCall(Transport.java:196)
	at java.rmi/sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:562)
	at java.rmi/sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:796)
	at java.rmi/sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.lambda$run$0(TCPTransport.java:677)
	at java.base/java.security.AccessController.doPrivileged(Native Method)
	at java.rmi/sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:676)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
```

原因是因为：

compose的1.0.0-alpha10目前只与Kotlin 1.4.20 兼容，而默认的4.2的Android Studio创建的工程的Kotlin是1.4.30版本，所以需要修改build.gradle的：

```
composeOptions {
        kotlinCompilerVersion '1.4.21'
        kotlinCompilerExtensionVersion '1.0.0-alpha10'
    }
```
和
```
dependencies {
        classpath "com.android.tools.build:gradle:7.0.0-alpha09"
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:1.4.21"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
```
中的kotlin版本保持一致即可。
