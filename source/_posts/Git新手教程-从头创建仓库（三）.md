---
title: Git新手教程-从头创建仓库(三)
tags:
  - Git
categories:
  - Git相关
date: 2019-10-12 00:14:12
---


### 前言

在上篇文章中，我们学习了版本控制系统的一些专业术语，我们在计算机上也安装了Git，并为Git做了一些初始配置，比如添加点击邮件，名字。及配置默认的Git的默认编辑器。在本篇文章中，我们将学习仓库的创建。在该篇文章中，我们不仅将学习如下三个命令`git init`、`git clone`、和`git status`，还将会学习该命令在`IntelliJ IDEA` or `Android Sutdio`中图形界面的对应操作。

在具体阅读文章之前，我们先简单的了解一下这三个命令的作用。

- `git init`: 可以让我们在计算机上从头创建全新仓库
- `git clone`: 可以让我们远端拷贝仓库到本地
- `git sataus`: 可以让我们查看仓库的状态

### 创建仓库

在对仓库进行commit或执行其他操作之前，我们需要创建一个实际存在的仓库，要使用Git新建仓库，我们需要使用 `git init` 命令。

>`init` 是英语单词 `"intializ”` 的简称。

当使用 `git init` 命令时，我们在命令行中会得到如下提示：

{% asset_img git的init命令使用.jpg git的init命令使用 %}

在上图中我的仓库的地址为 `documents/GitTest/GitTestProject` ，当然你也可以根据你自己目录及项目名称来创建Git仓库。

#### git init 命令的作用

当我们使用 `git init` 命令创建仓库后，我们能在当前目录下得到一个隐藏的 `.git` 文件夹，该文件夹下的内容记录了我们所有的commit，及其他信息。该文件下的内容如下所示：

{% asset_img Git目录结构.jpg Git目录结构 %}

>如果你已经使用Git管理你的项目，请不要直接修改`.git`文件夹下的任何文件。该文件下的内容是Git自身进行维护的。如果更改，会导致丢失一些重要的内容。切记不要删除、编辑这些文件。

该目录下的各个文件的作用大致如下：

- branchs 目录 - 存储了所有项目有关的分支信息。
- config 文件 - 存储了所有与项目有关的配置设置。
Git 会查看 Git 目录下你当前所使用仓库对应的配置文件（`.git/config`）中的配置值。这些值仅适用于当前仓库。
例如，假设你将 Git 全局配置为使用你的个人电子邮箱。如果你想针对某个项目使用你的工作邮箱，则此项更改会被添加到该文件中。
- Head 文件 - 指示当前Git项目处于哪个分支。
- description 文件 - 此文件仅用于 GitWeb程序使用，我们无需关系。
- hooks 目录 - 我们会在此处放置客户端或服务器端脚本，以便用来连接到 Git 的不同生命周期事件
- info 目录 - 目录包含一个全局性排除（global exclude）文件，用以放置那些不希望被记录在 `.gitignore` 文件中的忽略模式（ignored patterns）
- objects 目录 - 此目录将存储我们提交的所有`commit`
- refs 目录 - 此目录存储了指向 `commit` 的指针（通常是“分支”和“标签”)

关于branch(分支）、Head我们会在接下来的文章中进行介绍，其他的文件或目录我们可以不用关心，当然如果你想了解更多的内容，可以观看官方文档下：

- [Git 内部原理 - 底层命令和高层命令](https://git-scm.com/book/zh/v2/Git-内部原理-底层命令和高层命令)
- [自定义 Git - Git 钩子](https://git-scm.com/book/zh/v2/自定义-Git-Git-钩子)

### 克隆现有仓库

当然除了使用 `git init` 来创建仓库以外，我们还可以通过 `git clone` 命令来克隆现有的项目。通过 `git clone` 命令我们可以完全拷贝一个项目。而被克隆的项目完全是另一个项目的副本。

>在克隆任何项目之前，确保当前命令行以及在正确的目录下，克隆项目会新建一个目录，并将克隆的Git仓库放入其中。在Git中是不允许创建 `嵌套的Git仓库` 。因此当你使用 `git clone` 时，确保当前命令行所在的工作目录没有位于Git仓库中。

下面我们就使用 `git clone` 命令，来克隆我们的项目吧。克隆一个项目的具体步骤为：输入命令 git clone，然后输入你要克隆的 Git 仓库的路径。这里以我的[SimpleEyes项目](https://github.com/AndyJennifer/SimpleEyes)(一个仿开眼的kotlin项目)为例。该项目路径为：`https://github.com/AndyJennifer/SimpleEyes`。

```bash
git clone https://github.com/AndyJennifer/SimpleEyes
```

当然除了使用Http协议克隆项目以外。我们还可以使用本地协议、SSH协议与Git协议。这里以Git协议为例：

```bash
git clone git@github.com:AndyJennifer/SimpleEyes.git
```

>想要了解更多Git使用的协议可以查看官方文档-->[服务器上的-Git-协议](https://git-scm.com/book/zh/v2/服务器上的-Git-协议)

#### git clone 输出结果的简要说明

输入命令后效果如下所示：

{% asset_img git_clone展示.gif git_clone展示 %}

这里我们简单的介绍下 git clone 显示的输出结果。

第一行是"Cloning into 'SimpleEyes'…"。表示当前 Git 正在创建一个目录（名称与我们要克隆的项目一样），并将仓库放在对应项目中。其余输出结果基本都是验证信息——也就是统计远程仓库的项目数以及当前的下载百分比。

#### 克隆项目并使用不同的名称

需要注意的是通过 `git clone` 命令克隆项目，默认情况下，Git会把路径最后一级目录名作为新克隆项目的目录名(如果你的最后一级目录名包括`.git`，会将该后缀去掉)。如果说我们自定义项目名称呢？粗暴而又简单的方式是找到我们克隆项目的地址，手动重命名即可。但是我们不采用这种粗暴的方法。我们可以
使用如下命令：

```bash
git clone https://github.com/AndyJennifer/SimpleEyes  MyProject
```

对，就是如此的简单，就是在原有的命令中添加空格，并输入我们自定义的项目名称就可以啦。

### 判断仓库的状态

使用 `git status` 命令可以显示工作目录和暂存区的状态（在上篇文章我们曾经提到过，Git的工作流程主要围绕三个部分，工作区，暂存区与仓库区）。它可以让你看到哪些更改已经被转移，哪些没有转移，哪些文件没有被Git跟踪。进入我们创建的Git仓库，并输入 `git status` 时，我们能得到下图的输出结果：

```bash
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

从输出结果中，我们可以得到两条信息：

- On branch master - 这部分告诉我们 Git 位于 master 分支上。关于"master"分支（也就是默认分支）。我们将会在后续的分支文章中介绍。
- No commits yet - 表示项目中没有任何提交，关于`commit`我们也会在下文进行介绍。
- nothing to commit (create/copy files and use "git add" to track) –表示没有任何的提交信息，我们可以通过`git add`添加文件，使Git能够跟踪。

>注意：如果是第一次使用Git，我们一定要常常使用 `git status` 命令来查看仓库中文件或目录的状态。

关于更多的 `git status` 的更多内容，我们可以查看一下链接：

- [git-status](https://git-scm.com/docs/git-status)
- [官方文档的介绍](https://git-scm.com/book/zh/v2/Git-基础-记录每次更新到仓库)

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

了解了这三个命令后，我们来了解一下，在 `IntelliJ IDEA`or `Android Sutdio`中Git存储与清理的图形化界面的对应流程。虽然命令行非常牛逼，但是有时候我们也想偷个懒，对吧。让我们看看在`IntelliJ IDEA`or `Android Sutdio`中怎样操作吧。

#### 配置 Git 环境

在实际创建创建仓库之前，我们需要在我们的 IDEA 中配置 Git ，我们需要 IDEA 中依次点击 `Preferences` -> `Version Control` -> `Git` ，然后在 `Path To Git executable` 上输入安装 Git 的位置，点击右侧的 `Test` 按钮，如果成功弹出 `Git executed successfully` 弹框，说明配置成功，如下所示：

{% asset_img idea_配置_git.png idea_配置_git %}

如果你不知道 Git 的安装位置。

- Mac 操作系统：终端中输入 `which git`
- Windows 操作系统：命令行中输入 `where git`

#### git init 的使用

在编译器中，我们只要选择工具栏中的`VCS` -> `import into Version Control` -> `Creat Git Repository` 就可以从创建我自己的仓库啦。

{% asset_img git-init展示.png git-init展示 %}

通过上述方式，最终会选择一个仓库目录地址。这里由于篇幅限制就不具体描述啦。

#### git clone的使用

`git clone` 在IDEA中，有两种操作流程如下图所示：

**第一种：**
{% asset_img git-clone方式第一种.png git-clone方式第一种 %}

**第二种：**
{% asset_img git-clonet方式第二种.png git-clonet方式第二种 %}

以上两种方式都可以 clone 项目，只是叫法不同而已。当然这两个操作点击之后会走到如下流程：

{% asset_img clone界面展示.png clone界面展示 %}

就是这么简单，和我们使用命令行是一样的~

#### git status

在编译器中没有可视化的操作界面来完全显示 `git status` 命令所执行的结果，而是以一种非常直观的颜色来表示当前仓库中的文件状态。比如

- 红色：表示当前文件或目录没有被跟踪或与其他文件冲突。
- 绿色：表示当前文件或目录已经被添加到仓库中了。
- 蓝色：表示被添加到仓库中的文件或目录被修改或移动。
- 橙色：表示被忽略的文件。
- 白色：表示没有任何更改。

当然我们可以通过编译器底部的`Version Control`中的`local Changes`查看我们当前的仓库状态。如下图所示，

{% asset_img git-status可视化界面展示.png git-status可视化界面展示 %}

### 最后

站在巨人的肩膀上，才能看的更远~

- [视频推荐-->用Git进行版本控制](https://cn.udacity.com/course/version-control-with-Git--ud123)
- [Git官方网站](https://Git-scm.com/book/zh/v2/)
