---
title: Gradle系列-Gradle的安装(二)
categories:
- Gradle相关
tags: 
- Gradle
---

{% asset_img gradle.png gradle %}

>一直在忙公司的项目，要不就是写另一个系列的博客，一直抽不出时间来写Gradle系列文章。好了，我知道都是借口，不管怎么样，Gradle系列要开始更新啦。你准备好没有呢。

### 前言

在前面的文中，我们简单的了解了Gradle在项目构建中的作用。如果有小伙伴对Gradle不是很熟悉。可以参看上篇文章[Gradle系列-引导篇（一）](https://www.jianshu.com/p/71ee58ccf5b2)中对Gradle中的介绍。那现在我们既然已经知道了Gradle的作用了。现在我们就一起开始下载Gradle把~~

### 要求

Gradle可以运行在大部分的操作系统上，需要`Java Jdk`的版本在`7与7之上`，你可以这么检查，通过运行命令行`java -version`来检查你的版本，那么你可能得到如下结果：

```bash
C:\Users\andy>java -version
java version "1.8.0_191"
Java(TM) SE Runtime Environment (build 1.8.0_191-b12)
Java HotSpot(TM) Client VM (build 25.191-b12, mixed mode)
```

Gradle带有自己的`Groovy`库，因此不需要安装Groovy，任何现有的Groovy安装都会被Gradle忽略。Gradle会使用你设置的JDk的`path`，或者你也可以设置一个`JAVA_HOME`的环境变量去指向当前JDK安装的路径。

>在Gradle中，每个构建脚本都是基于Groovy语言，其中Groovy是基于JVM虚拟机的一个动态语言。它的语法和Java非常相似。同时Groovy也是完全兼容Java。在后续的文章中，我会对Groovy进行介绍。

### 手动的安装Gradle

#### 第一步、下载最新版本的Gradle

我们可以通过官网[https://gradle.org/releases/](https://gradle.org/releases/)去下载我们想要下载的Gradle版本。
{% asset_img 官方下载界面.png 官方下载界面 %}

点击进入官方下载页，我们可以看到一些Gradle版本的信息与下载地址，其中下载分为了两个版本：

- 只有二进制（binary-only)
- 完整版包含文档与源代码等(complete)

这里我们选择下载最新的版本`v4.10.2`的完整版。

#### 第二步、解压压缩包

##### Linux&MacOs 用户

下载完成后，我们任然需要执行大量的其他工作来安装Gradle，我们需要解压之前的压缩包，然后移动解压的文件到我们想要移动的位置（Gradle官方建议放在`/usr/local/gradle`目录下），然后我们还需要添加一些环境变量。你可以自己执行所有这些操作。但是如果将下面这一大块脚本粘贴到控制台中，它将会自动执行所有的操作。具体脚本如下所示：

```bash
unzip ~/Downloads/gradle-3.3-all.zip -d /usr/local/gradle/ &&\
echo '# Adding Gradle to system path
export GRADLE_HOME=/usr/local/gradle/gradle-3.3
PATH=$GRADLE_HOME/bin:$PATH
export PATH' >> ~/.bash_profile &&\
source ~/.bash_profile
```

注意，如果最后一个命令行错误，也就是`source ~/.bash_profile`，你可能需要使用`sudo`来运行它。

##### Windows 用户

对于Windows 用户，为了使Windows找到Gradle，我们需要通过以下方式来添加环境变量。
首先找到我们的控制面板>系统>高级系统设置>环境变量>系统变量>新建.....

- 添加变量名称，`GRADLE_HOME` 然后设置变量对应的值，这里的值是你当前解压的gradle对应的地址。这里我的是`E:\gradle-4.10.2`。
- 当设置完成后，你需要在`Path`末尾中添加刚才你添加的变量值，也就是`;%GRADLE_HOME%\bin\`

#### 第三步、检测

一旦你安装好Gradle，并配置好相应的环境变量后。那么你就可以直接使用`gralde`命令。检测你是否安装Gradle成功，最简单的方法就是通过使用命令`gradle -version`。也就是会得到如下界面：

```bash
C:\Users\andy>gradle -version

Welcome to Gradle 4.10.2!

Here are the highlights of this release:
 - Incremental Java compilation by default
 - Periodic Gradle caches cleanup
 - Gradle Kotlin DSL 1.0-RC6
 - Nested included builds
 - SNAPSHOT plugin versions in the `plugins {}` block

For more details see https://docs.gradle.org/4.10.2/release-notes.html


------------------------------------------------------------
Gradle 4.10.2
------------------------------------------------------------

Build time:   2018-09-19 18:10:15 UTC
Revision:     b4d8d5d170bb4ba516e88d7fe5647e2323d791dd

Kotlin DSL:   1.0-rc-6
Kotlin:       1.2.61
Groovy:       2.4.15
Ant:          Apache Ant(TM) version 1.9.11 compiled on March 23 2018
JVM:          1.8.0_131 (Oracle Corporation 25.131-b11)
OS:           Windows 10 10.0 x86
```

### 运行你的Hello World吧

 当所有东西都安装好，并将环境变量配置好后。按照国际惯例，现在我们可以给世界打招呼了。你现在只需要将该文件[build.gradle](https://pan.baidu.com/s/1psH0mg7qjWpOQhUQ5O1ROg)下载下来。然后拖到你的桌面上。通过命令行进入你的桌面目录，然后输入`gradle -q helloWorld`，这里以我自己的操作为例，这里我的桌面地址为`C:\Users\andy\Desktop`。具体如下所示：

```bash
C:\Users\andy>cd C:\Users\andy\Desktop
C:\Users\andy\Desktop>gradle -q helloWorld
Hello, World!
C:\Users\andy\Desktop>
```

这里肯定有很多小伙伴对Gradle命令不是很熟悉。这里的`gradle -q helloWorld`是什么意思呢？为什么我的文件名必须为`build.gradle`呢。关于这些问题。我都会在后续文章中，为大家进行解答。现阶段，我们只需要着重把环境配置好，就OK啦。

### 最后

Gradle的学习路程，是比较漫长的。不可能通过一篇文章就能介绍清楚了。希望大家和我一起加油咧。等Gradle学成。迎娶白富美不是问题。

### 参考

站在巨人的肩膀上。可以看得更远。

- [Gradle官网](https://gradle.org/)
- 《Android Gradle权威指南》
