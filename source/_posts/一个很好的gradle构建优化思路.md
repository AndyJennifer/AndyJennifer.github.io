---
title: 一个很好的gradle构建优化思路
tags:
  - Gradle
categories:
  - 构建速度优化
date: 2020-10-15 21:17:39
---


## Android 项目构建速度优化

随着项目的不断扩大，最影响我们的 Code 效率的是项目的编译。下面我就带着大家从 Android 构建流程中去分析如何提高项目的构建效率。

{% asset_img 上车.jpg 上车 %}

## 一切从项目编译过程说起

Android 项是从编译到打包流程如下所示：

{% asset_img 打包流程.png 打包流程 %}

为了方便大家理解这里对其中主要的构建过程进行描述(上图中绿色椭圆部分)：

- aapt：`aapt(Android Asset Packaging Tool)` 工具会打包应用中的资源文件，如 `AndroidManifest.xml、layout` 布局中的 `xml` 等，并将 xml 文件编译为二进制形式，当然 assets 文件夹中的文件不会被编译，图片及 raw 文件夹中的资源也会保持原来的形态，(需要注意的是 raw 文件夹中的资源也会生成资源 id。AAPT 编译完成之后会生成 `R.java 文件`)。
- aidl：AIDL 工具会将所有的 aidl 接口转化为 java 接口。
- Java Compiler(Java编译器)：当 AAPT 与 AIDL 工具将需要处理的数据处理好后，Java 编译器会将所有的java代码，包括R.java与 aidl 文件编译成 .class 文件。
- dex：`dex` 工具会将上述产生的 .class 文件及第三库及其他 .class 文件编译成 .dex 文件（dex文件是Dalvik虚拟机可以执行的格式），dex文件最终会被打包进APK文件。
- apkbuilder：`apkbuilder` 工具会将编译过的资源及未编译过的资源（如图片等）以及 .dex 文件打包成APK文件。
- Jarsingner：生成 APK 文件后，需要对其签名才可安装到设备，平时测试时会使用 debug keystore，当正式发布应用时必须使用 release 版的 keystore 对应用进行签名。Jarsigner工具会根据相应的keystore生成相应的签名APK文件。
- zipalign(release mode)：`zipalign` 工具，它能够对打包的应用程序进行优化。在你的应用程序上运行 `zipalign` ，使得在运行时 Android 与应用程序间的交互更加有效率。

### 优化的思路

到了这里有可能有小伙伴会说 "MD,你说了这么说多，那具体优化思路是什么呢？“ 不急，不急，慢慢道来。

{% asset_img 问号.jpg 问号 %}

从 Android 项目编译的过程中，我们能从几个维度来减少编译的时间：

- 减少不必要的资源编译
- 减少拉取第三方库的时间
- 减少编译时间
- 减少 apbuilder 的打包时间

除了以上维度，还有一个重要的维度，**工欲善其事必先利其器**。 Android Studio、Android Gradle 插件、 工具的每次更新，都会获得构建方面的优化和新功能。那么从以上几个维度：我们能引入如下的优化方式：

{% asset_img 优化方式.png 优化方式 %}

### 项目实际的改动点

因为时间原因，所以改动点并不是很大，并没有做到类似字节跳动的增量更新方案。但是仍然有着不错的效果。优化前后时间对比如下所示：

|               | 全量编译（Rebuild Project) |        增量编译  | 不动任何文件(直接编译）     |
|:------------- |       :---------------:  | :-------------:   |      -------------:   |
| 修改前         | 平均 18m                  |       平均 18-20m  |      平均 45s         |
| 修改后         | 平均 8m                   |       平均 8m      |      平均 33s         |

整个优化思路如下所示：

- 确保 Android 工具最新
- 使用增量注解处理器

在下面会详细描述整个优化的详细过程。

#### 将 Android Gradle 插件从 `3.4.2` 升级到 `3.6.0`

升级到 `3.6.0` 后具体优势如下所示：

- 并行执行任务
- 新的默认打包工具（在构建应用的调试版本时，该插件会使用一个新的打包工具 zipflinger 来构建 APK。这一新工具应该能够提高构建速度)
- 简化了 `R` 类的生成过程（Android Gradle 插件通过仅为项目中的每个库模块生成一个 R 类并与其他模块依赖项共享这些 R 类，简化了编译类路径。这项优化应该会加快构建速度）

>这里并没有将 Android Gradle 插件升级到最新版本 `4.1.0`，是因为在项目中使用的  `androidannotations` 库最大能支持 Android Gradle 插件版本为 `3.6.0`。

#### 将 androidannotations 注解库升级到 `4.7.0`

在项目中，我们使用了低版本（4.6.0）的 [androidannotations](https://github.com/androidannotations/androidannotations) 库来生成中间类。而使用这类的 `注解处理器` 库会导致一个问题，就是当有新的中间类生成时会引发全量编译。为了提交构建速度，避免全量编译。故将该库升级到最新的 [4.7.0](https://github.com/androidannotations/androidannotations/wiki/ReleaseNotes#4.7.0) 版本。在该版本中该库支持 [增量注解处理](https://docs.gradle.org/current/userguide/java_plugin.html#sec:incremental_annotation_processing)

> androidannotations 4.7.0 最大支持的 Android Gradle 插件版本为 `3.6.0`

#### 其他改动点

- 移除掉了 [fastdex](https://github.com/typ0520/fastdex) 插件（在实际的项目中并没有使用）
- 移除掉了会在 debug 模式下执行的统计耗时任务(如 findbug，jacoco)
- 修改了部分 atrr.xml 文件及 layout 文件 （因为升级版本造成的编译冲突，所以需要修改）

### 后续可以做的

- 移除掉无用的库
- 增量 java/kotlin 编译
- transform 优化
- dexBuilder 增量效果量化
- ......

### 参考资料

- [Google官方优化构建速度文档](https://developer.android.google.cn/studio/build/optimize-your-build)
- [字节跳动](https://juejin.im/post/6854573211548385294#heading-9)
- [gradle 官方构建速度优化](https://guides.gradle.org/performance/)
- [有赞优化方案](https://tech.youzan.com/you-zan-android-bian-yi-jin-jie-zhi-lu-zeng-liang-bian-yi-ti-xiao-fang-an-savitar/)
- [Gradle 并行项目构建](https://docs.gradle.org/4.10.3/userguide/multi_project_builds.html#sec:parallel_execution)

### 最后

站在巨人的肩膀上，才能看的更远~
