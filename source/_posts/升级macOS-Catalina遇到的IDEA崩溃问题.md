---
title: 升级macOS Catalina遇到的IDEA崩溃问题
date: 2019-07-11 13:04:11
tags:
- 工具
---
**小白鼠在把主力开发机升级到Catalina后发现，IDEA崩溃了？？！!**

![][image-1]
---- 
在Applications/idea/macOs下的终端启动崩溃后，可以看到以下日志：
_A fatal error has been detected by the Java Runtime Environment:_
_SIGSEGV (0xb) at pc=0x00007fff2dc0a930, pid=4593, tid=0x000000000000f603_
_JRE version: OpenJDK Runtime Environment (8.0202-b49) (build 1.8.0202-release-1483-b49)_
_Java VM: OpenJDK 64-Bit Server VM (25.202-b49 mixed mode bsd-amd64 compressed oops)_
_Problematic frame:_
_C [CoreGraphics+0x195930]() ERRORCGDataProviderCreateWithDataBufferIsNotReadable+0x10_
_Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again_
_If you would like to submit a bug report, please visit:_
_http://bugreport.java.com/bugreport/crash.jsp_
_The crash happened outside the Java Virtual Machine in native code._
_See problematic frame for where to report the bug._

经过各种搜索，可以确认问题是 JBR （ Jetbrains Runtime ） 里面引入的新字体渲染 harfbuzz 导致的问题。解决方法有几种：
1. 回滚你的操作系统。
2. 启动idea-\>double shift调出搜索框-\>输入switch book JDK-\>切换idea启动的jdk。

![][image-2]

对于2，可以选择oracle官方的jdk，但这样虽然可以coding，但idea上的很多操作都无法使用。经过又一番搜索，在[这里][2]找到了jbrsdk的最新版下载地址，下载这里的jdk，把boot JDK替换就可以重新愉快的Coding了。以后主力机升级Beta系统真的要谨慎，实在是坑太多。。

[2]:	https://bintray.com/jetbrains/intellij-jbr/jbrsdk8-osx-x64/1596.1

[image-1]:	http://pt1160j8s.bkt.clouddn.com/idea-crack
[image-2]:	http://pt1160j8s.bkt.clouddn.com/idea-switchbookjdk.png