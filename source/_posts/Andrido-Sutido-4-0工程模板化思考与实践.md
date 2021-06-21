---
title: Andrido Sutido 4.1工程模板化思考与实践
tags:
  - 工程模板
categories:
  - Android Studio
date: 2021-06-07 19:58:51
---


>快一年没有更新文章了。这一段时间，发生了许多事情吧。可能是工作上，也许是生活上。不过，最终还是回归到了正常的状态中来。不管怎么样，我回来了。后续文章，大家敬请期待吧~

### 前言

因为工作需要，组内需要做一套 Android 项目工程模板的方案，主要目的是为了解决开发人员在初始化项目时重复劳动的问题。下面我想和大家分享一下，关于这个“课题”我的思考过程以及我选择的技术方案。希望该篇文章，能为需要的小伙伴提供一些思路和帮助。

>如果只想关心如何在 Android Studio 4.1+ 实现工程模板化，可以直接查看文章--

### Android 工程模板化需要解决的问题

到这里，可能有小伙伴会说：”工程模板化不就是创建样板文件？比如自动生成 `”MVVM"`、`“MVP"`、`”MVI"`文件， 还有官方提供的模板列表吗？“


{% asset_img 官方模板.jpg 官方模板 %}


对，工程模板化最终的产物就是 `模板代码`，但刚刚我们上面所说的内容，只是项目模板化的一个小点。除了 `模板代码` 以外，在 Android 项目的生成中，我们还需要从`构建`与`打包`两个维度来考虑。如：

- Project/Library 的构建（基础配置、依赖配置、编译选项、模板代码等）
- Project/Library 的打包（签名配置、打包配置）


{% asset_img 思考方向.jpg 思考方向 %}

如果我们再把这些配置细化来看，我们会发现有更多的思考点。🤔🤔🤔

{% asset_img 安卓工程模板化.png 安卓工程模板化 %}

在上图中，我大概梳理出了 Android 工程模板需要考虑的功能点。当然，不同的项目对工程模板化的要求不同，大家可以参考这个思路对自己的项目模板进行订制。

### 调研的技术方案

在有了大致的思路之后，我调研了现阶段实现 Android 项目模板的技术方案，下面分别对这几种方案进行介绍。

#### 自定义脚本

自定义脚本的方案思路其实非常简单，就是在本地建立好项目模板，然后根据用户在命令行输入的参数或配置文件，根据规则复制或创建新的文件。具体流程如下所示：

{% asset_img 脚本流程.png 脚本流程 %}

该方案的优点：

- 实现简单快速。生成好的项目可以直接通过 Android Studio 打开并重新  build 就好了。

缺点：

- 因为是通过脚本来快速生成，需要自己定义一套生成规则。
- 无法控制项目模块之间的依赖性。需要人为介入
- 对于这种项目构建的，命令行或者配置文件的方式，对开发人员来说，可能不是很友好。


#### FreeMarker 模板方案

在 Android 4.1 之前的版本，提供了一套基于 `FreeMarker` 技术的模板实现，需要在指定的文件夹中提供相应的`.ftl`文件。如:

- Windows 的路径在 `${android studio 安装路径}/plugins/android/lib/templates/`
- MacOS 的路径在 `${Android Studio.app 存放路径}/Contents/plugins/android/lib/templates/`

具体的代码生成过程如下所示：

{% asset_img 代码生成过程.jpg 代码生成过程 %}

但是在 `Android 4.1`之后，谷歌停止了对该模板的支持，具体原因是因为配置参数是 `.ftl` 的方式来描述配置信息，这种对开发者来说很容易配置错误。最为重要的是为了看到实际的模板效果，开发者必须要重启 Android Studio，这样的操作太过傻瓜式。所以谷歌了提供了另一种更友好，阅读性更强的方案。也就是接下来我们要介绍的插件方案。

对 `FreeMarke` 感兴趣的小伙伴，可以参考下列文章，这里就不再进行介绍了。

- [网易Android工程模板化实践](https://www.jianshu.com/p/a264afa089bc)
- [custom-android-code-templates](https://www.slideshare.net/murphonic/custom-android-code-templates-15537501)
- [Android ADT Template Format Document](https://www.i-programmer.info/professional-programmer/resources-and-tools/6845-android-adt-template-format-document.html)
- [FreeMarker 在线手册](http://freemarker.foofun.cn/ref_directives.html)


#### 自定义 Android Studio 插件方案

从 Android Studio 4.1 开始，Google 停止了对自定义 FreeMarker 模板的支持，也就是 `不支持书写.ftl文件并放在特定文件夹来生成模板的方式这中方式`，而是通过扩展[android-plugin](https://plugins.jetbrains.com/docs/intellij/extension-point-list.html#android-pluginxml)插件中的模板扩展点 `WizardTemplateProvider`  来实现我们自己模板工程。如下所示：


{% asset_img AndroidPlugin扩展.jpg AndroidPlugin扩展 %}

基于 `WizardTemplateProvider` 模板方案的主要流程并不复杂，下面简单介绍一下：


{% asset_img Wizard.png Wizard %}


在我们使用 Android Studio 新建 Project/Library 的时候会先获取`模板列表`，模板列表中维护了一个个 `Template`。这些 `Template` 对应到 Android Studio 中，其实就是这些：


{% asset_img 官方模板2.jpg 官方模板2 %}


而这些 `Template` 是由多个 `WizardTemplateProvider` 提供的。 `Template` 内部维护了一整套模板配置流程，包括配置界面参数->实际文件生成两个关键步骤，这两个步骤分别由内部的 `UI Parameters`、 `Recipe`, `RecipeExecutor` 组件来控制，具体如下所示：

{% asset_img Recipe.png Recipe %}

其中 `UI Parameters`、 `Recipe`, `RecipeExecutor` 主要作用如下所示：

- UI Parameters: 模板配置UI界面。主要负责录入模板配置参数。如模块名称，最小支持SDK等
- Recipe: 模板工程核心逻辑。根据 UI Parameters组件提供的配置参数，创建模板文件、模板代码等操作
- RecipeExecutor: 模板工程底层API。提供添加依赖，拷贝文件，创建文件夹，合并xml文件等基础能力


该方案的优缺点：

优点：

1. 因为是官方提供的API，比如


缺点：

- 问题如果要多项目的构建版本及什么进行统一管理。无法修改。官方就只提供了那几个API

### 最后


- [Magic Templating for Android Projects](https://medium.com/hh-ru/magic-templating-for-android-projects-f937aedfe691)
- [Template plugin for Android Studio 4.1+](https://steewsc.medium.com/template-plugin-for-android-studio-4-1-92dcbc689d39)
- [Android Studio source code and the code of relevant tamplates](https://cs.android.com/android-studio/platform/tools/base/+/mirror-goog-studio-master-dev:wizard/template-impl/src/com/android/tools/idea/wizard/template/impl/)
- [网易Android工程模板化实践](https://www.jianshu.com/p/a264afa089bc)

站在巨人的肩膀上，才能看的更远~
