---
title: Gradle系列-引导篇（一）
date: 2019-02-23 22:15:49
categories:
- Gradle相关
tags: 
- Gradle
---



![gradle.png](https://upload-images.jianshu.io/upload_images/2824145-6fb4a4059228244a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>题外话：其实本来不想取这个名字的，但是感觉不取这个名字感觉没有几个人看啊。大家肯定觉得这个名字比较高大上吧！哈哈哈哈。好了，收。

### 前言
在平时Android开发中我们常常使用Gradle来构建我们的项目，我相信大家都可能遇到以下问题：
- 开启项目提示界面一直显示Gradle Build Running 
- Gradle传递性依赖冲突
- 多渠道打包
- .....等

相信大家在平时使用的时候，遇到问题都通过搜索引擎来解决，有些小伙伴可定会想,"`作为一个Android开发者，我没有必要去详细的了解Gradle到底去这么使用，平时开发任务本来就比较重，哪里有时间有精力来学习呢`" 但是个人觉得对Gradle的了解，对于我们平时开发项目有很重要的帮助。

### Java项目的构建
要知道Gradle是什么以及其作用。我们需要从整个Java项目的构建说起，看下图：
![gradle-java-builds.png](https://upload-images.jianshu.io/upload_images/2824145-3f62357c67fc183d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中我们可以看出在平时Java项目的构建流程，或多或少我们会涉及以下操作：
- 通过`javac`命令将一些Java源文件编译为class文件。
- 将类文件和资源文件（图像和字符串的资源）压缩为Jar包。
- 通过`javadoc`命令提取Java源文件的中的注释。生成文档。
- 运行一些单元测试，或程序验收测试。
- 将Jar文件部署到资源库中。

既然Java项目的构建会经历以上或更多的步骤，那么我们接下来看看Android项目的构建流程。

### Android项目的构建

对于Android项目的构建，主要会经历和解决下面这些问题：
- Android对于Java源文件`并未按照标准Java字节码编译`，但可为Android运行时自定义字节码。
- Android具有三种资源类型（R.java、Application source code、JavaInterfaces)，且按照不同的方式打包。
- 还有一个难题就是你定义的资源需要与所包括的资源库中的资源进行汇集，在编译其他任何程序之前，需要知道所有这些资源的识别符。
- Android应用多数情况下会对应用进行加密。

那么汇集所有的资源以及情况后，整个Android的构建流程看起来是这个样子：

![Android构建流程图.png](https://upload-images.jianshu.io/upload_images/2824145-eb33e56f256fe391.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了方便大家理解这里对其中主要的构建过程进行描述(`上图中绿色椭圆部分`)：
- aapt：aapt(Android Asset Packaging Tool)工具会打包应用中的资源文件，如AndroidManifest.xml、layout布局中的xml等，并将xml文件编译为二进制形式，当然assets文件夹中的文件不会被编译，图片及raw文件夹中的资源也会保持原来的形态，(`需要注意的是raw文件夹中的资源也会生成资源id。AAPT编译完成之后会生成R.java文件`)。
- aidl：AIDL工具会将所有的aidl接口转化为java接口。
- Java Compiler(Java编译器)：当AAPT与AIDL工具将需要处理的数据处理好后，Java 编译器会将所有的java代码，包括R.java与aidl文件编译成`.class文件`。
- dex：dex工具会将上述产生的.class文件及第三库及其他.class文件编译成`.dex文件`（dex文件是Dalvik虚拟机可以执行的格式），dex文件最终会被打包进APK文件。
- apkbuilder：apkbuilder工具会将编译过的资源及未编译过的资源（如图片等）以及.dex文件打包成APK文件。
- Jarsingner：生成APK文件后，需要对其签名才可安装到设备，平时测试时会使用debug keystore，当正式发布应用时必须使用release版的keystore对应用进行签名。Jarsigner工具会根据相应的keystore生成相应的签名APK文件。
- zipalign(release mode)：zipalign工具，它能够对打包的应用程序进行优化。在你的应用程序上运行zipalign，使得在运行时Android与应用程序间的交互更加有效率。

>在Android中，每个应用程序中储存的数据文件都会被多个进程访问：安装程序会读取应用程序的manifest文件来处理与之相关的权限问题；Home应用程序会读取资源文件来获取应用程序的名和图标；系统服务会因为很多种原因读取资源（例如，显示应用程序的Notification）。此外，就是应用程序自身用到的资源文件。
>在Android中，当资源文件通过内存映射对齐到4字节边界时，访问资源文件的代码才是有效率的。但是，如果资源本身没有进行对齐处理（未使用zipalign工具），它就必须回到老路上，显式地读取它们——这个过程将会比较缓慢且会花费额外的内存。

从整个Android项目的构建来看，我们会感叹“为啥我就简单的创建一个应用，为毛有非常多的事情需要做。”，所以为了方便处理这些，我们都会想是不是可以写一个能自动处理这些过程的程序化脚本呢？`所以Gradle出现了！！！！`。

### 为毛选择Gradle？
对于以前传统的项目构建工具，只是编译和打包源代码。而现在项目的构建需要负责更多的工作，它们会运行测试、从多个来源购买编码资源、生成文档、创建多个构建变种、发布应用程序和管理依赖性。而Gradle不仅具备这些能力与功能，还解决了Android开发人员面临的一些最棘手的问题，如下所示：

- 如何自动构建和测试应用，以快速实现生产力？
- 如何管理依赖和变种。使专业开发人员只需要单击一次就能提取出其应用的数十个变种？
- 如何构建处理及处理非常大的应用？
- ...

哎呀说这么多，`其实最大的原因是Google爸爸已经选择Gradle做为Android Studio的构建系统`，在Android Studio中将Android应用的整个流程指派给了Gradle。当我们点击`运行`按钮时，Android studio会在运行过程中设置Gradle，并在后台监控。通过学习有关Gradle知识。我们可以扩展此默认行为。以构建能力更强且经过适当测试的应用。

`既然Gradle大法这么好，为毛我们不去学习呢？`

### 总结
Gradle是项目的构建工具，解决了我们平时开发中，项目测试、项目打包、项目依赖等问题。

###最后
Gradle系列会继续写。如果大家喜欢我的写作风格的话。欢迎大家点赞。

最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start
