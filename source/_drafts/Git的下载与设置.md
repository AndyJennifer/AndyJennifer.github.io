---
title: Git的下载与设置
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们介绍了Git的基本概念，了解的Git常用的术语。相信大家对Git已经有一个基本的了解了。工欲善其事，必先利其器。让我们去下载并配置Git吧。

### Mac/Linux/Windows设置

我们可以根据自己的系统，选择不同的版本。推荐上官网直接下载最新的版本。

* 跳转到Git相关下载界[https://git-scm.com/downloads](https://git-scm.com/downloads)
  
* 选择你所需要的系统版本并安装

![git版本选择.png](https://upload-images.jianshu.io/upload_images/2824145-b856d6b188753669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 安装完成后，在命令行输入git命令，如果显示了相关信息，那么就安装成功了

![安装成功.png](https://upload-images.jianshu.io/upload_images/2824145-b8bbf9fadc890854.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Mac/Linux下Git终端配置

如果你采用命令行使用Git，那么你可以使用Git提供的自动补全脚本。配置了该脚本后，在我们在实际Git命令行操作的时候，如果忘记相关选项的名称或相关命令，可以通过双击Tab键来进行提示。配置该脚本，我们需要以下步骤：

* 下载[config](https://pan.baidu.com/s/1ywZc4bU_8qkPMeoTBxbrww)

 ![文件展示.png](https://upload-images.jianshu.io/upload_images/2824145-7e0d2393a1398b15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当你下载我分享的内容后，你会看到如上三个文件。

* 接下我们需要将`terminal-config`文件夹移动到你的用户主目录下，并重新命名该文件为`.terminal-config`(前面有个点)。这里我们使用的`mv`指令

![演示.gif](https://upload-images.jianshu.io/upload_images/2824145-92a4998dd9bc4355.gif?imageMogr2/auto-orient/strip)

* 将bash_profile文件移动到你的主目录下，并命名为.bash_profile（前面有个点)

> 如果在你的主目录下已经存在`.bash_profile`文件，那么你只需要将下载好的`brash_profile`文件中的内容复制到现有的`.brash_profile`文件中。注意如果你是Ubuntu用户，你需要把设置信息复制到`.bashrc`文件中。

![profile演示.gif](https://upload-images.jianshu.io/upload_images/2824145-35d0b0caa46e0015.gif?imageMogr2/auto-orient/strip)

### 初始化配置

```bash
# 设置你的 Git 用户名
git config --global user.name "<Your-Full-Name>"

# 设置你的 Git 邮箱
git config --global user.email "<your-email-address>"

# 确保 Git 输出内容带有颜色标记
git config --global color.ui auto

# 对比显示原始状态
git config --global merge.conflictstyle diff3

# 通过该命令显示所有配置信息
git config --list
```

如果用了`--global`选项，那么以后你所有的项目都默认使用这里的配置用户信息，如果要在某个特定项目中使用其他名字或者邮件，只要去掉--global选项重新配置就可以啦~。

>通过 `git config --list`我们可以查看已有的配置信息

### Git文本编辑器配置

当我们在命令行使用Git需要输入额外信息的时候，Git会默认会调用系统的默认编辑器，一般情况是Vi或者Vim。当然我们也可以配置我们自己喜欢的文本编辑器。

#### Sublime Text 配置

```bash
git config --global core.editor "'/Applications/Sublime Text.app/Contents/SharedSupport/bin/subl' -n -w"
```

#### VSCode 配置

```bash
git config --global core.editor "code --wait"
```

>如果你对默认的编辑器(Vi或者Vim)感兴趣，你有可能需要这篇文章[Linux vi/vim](https://www.runoob.com/linux/linux-vim.html)

### 最后

站在巨人的肩膀上，才能看的更远~
