---
title: Git新手教程-查看仓库的历史记录（四）
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们学习了如何创建仓库。现在我们将学习如何查看仓库的历史记录，之所以没有先讲解如何向仓库如何提交commit，是因为我觉得，只有先了解历史记录中包含哪些信息后，我们才能更好的创建良好的提交。有了良好的提交，才会有助于以后我们对项目的整体回顾。在本文章中，我们将介绍 `git log` 和 `git show` 两个指令，这里先简单介绍一下这两个命令的功能。

- git log: 查看现有的提交信息
- git show:可以显示给定提交的信息。

#### git log 命令

当然使用`git log`命令，我们首先需要一个现有的Git仓库，这里还是以我自己的项目[SimpleEyes项目](https://github.com/AndyJennifer/SimpleEyes)为例。

还记得我们之前的介绍的`git clone`命令吗？，我们先clone该项目吧.

```bash
git clone https://github.com/AndyJennifer/SimpleEyes
```

克隆该项目后，我们通过 cd 命令进入该项目,使用 `git log` 命令，我们能得到下列输出：

![git_log展示界面.png](https://upload-images.jianshu.io/upload_images/2824145-e3a2a3d755462acf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 历史记录分析

默认情况下，使用 `git log` 命令会显示仓库中每个commit的详细提交信息。结合上图，我们来分析下每行所代表的具体内容。这里以第一个历史提交记录为例：

- commit头信息

```bash
commit 543019ea4bca77c31ccd1d06c4ca2ca4b1d69b23 (HEAD -> master, origin/master, origin/HEAD)
```

在Git中会为每个提交生成一个ID，也就是`SHA`。在本行中不仅显示了提交的ID,当前所指向的分支为 `master` ,当然现在我们可能还不了解分支的相关信息，不过请大家放心，我们会在后续的文章中学到它的。

>`SHA`是一个长 40 个字符的字符串（由 0–9 和 a–f 组成），并根据 Git 中的文件或目录结构的内容计算得出。SHA 的全称是"Secure Hash Algorithm"（安全哈希算法）。如果你想了解哈希算法，可以参考[SHA家族](https://baike.baidu.com/item/SHA家族/9849595?fromtitle=SHA&fromid=9533316&fr=aladdin)。

- 作者
  
```bash
Author: AndyJennifer <1225868370@qq.com>
```

当前行显示了提交这个commit的作者以及对应邮件信息。现阶段这个仓库只有我一个人维护，所以这里全是我的相关信息。当然，如果你的仓库是多人协作开发，那么不同的作者提交的commit所对应的Author也会不同啦。

- 日期

```bash
Date:   Thu Aug 29 23:43:07 2019 +0800
```

日期，很简单，就是现实当前commit的时间，一般情况下，我们是不会关心它的。

- commit消息

```bash
 修改了切换视频时，controller没有设置，导致的空指针异常
```

一般情况下，我们需要为每次提交的内容进行说明，比如增加了什么功能，修改了什么bug等等。一个良好的commit 提交内容与提交消息之间应该有着一一对应的关系，而不是表述模糊，张冠李戴。

#### git log 命令行日志浏览快捷键

如果你是第一次使用`git log`命令，可能会有一个疑问，如何加载更多的历史记录呢？如何退出呢？如果大家仔细观察，在我们的界面中末尾有个冒号`：`，该冒号表明还可以显示更多的输出行。在Git中是使用 `Less` 程序作为其分页器，如果你不熟悉 less 或分页器也没有问题。大家只要知道该分页器是用于翻页并浏览内容的就行了。下面我们看看该分页器对应的按键指令。

要向下滚动：

- `j` 或 `↓` 一次向下移动一行
- `d` 按照一半的屏幕幅面移动
- `f` 按照整个屏幕幅面移动

要向上滚动：

- `k` 或 `↑` 一次向上移动一行
- `u` 按照一半的屏幕幅面移动
- `b` 按照整个屏幕幅面移动

需要注意的是，当冒号变为单词END时，表示记录显示完毕，那么我们可以按下 `q` 可以离开分页器。

### 更改 git log 显示信息的方式

#### git log --oneline

使用 `git log` 命令，会显示日期、作者、commit消息等信息，但是有些情况下，我们可能并不关心日期、或相关作者。我们只想快速的浏览具体的提交信息，那么我们可以为当前的 `git log` 命令，增加一个选项 `--oneline` 。

>注意这里是 `oneline` 而不是 `online` 。

使用 `git log --oneline` 我们能得到如下输出：

![git-log--oneline展示.png](https://upload-images.jianshu.io/upload_images/2824145-0aa247eba7fd3e18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用 `git log --oneline` 命令会：

- 每行显示一个 commit
- 显示 commit 的 SHA 的前 7 个字符
- 显示 commit 的消息
  
#### 查看修改后的文件

在了解了 `git log --oneline` 命令后，我们可能会想深入了解某个commit更改了哪个或哪些文件。这个时候我们需要 `git log` 的另一个选项 `--stat` 。

> stat 是单词 `statistics` ,为统计的意思。

使用 `git log --stat` 命令，我们能得到如下输出：

>这里为了方便讲解，我只截取了特征明显的commit

![git-log--stat展示.png](https://upload-images.jianshu.io/upload_images/2824145-4d9a5f8ab98f2d85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中最后一行，表明这次提交共涉及到`8`个文件的修改，`160`行的插入，`5`行的删除。其中build.gradle文件中删除了1行，UserPreferences.kt中添加或删除了2行......相信到这里大家就明白了，使用命令`git log --stat`会：

- 显示被修改的文件
- 显示添加/删除的行数
- 显示一个摘要，其中包含修改/删除的总文件数和总行数

### 查看文件更改

在上文中，我么已经通过添加 `--stat` 选项，可以知道修改了哪些文件，以及添加/删除了多少行代码。如果能查看文件中到底进行了哪些更改，是不是更好呢？在 `git log` 命令中具有一个可用来显示对文件作出实际更改的选项，就是 `--patch` ，可以简写为 `-p` 。

例如，我们想查看下图中具体的修改：

![git-log-p1.png](https://upload-images.jianshu.io/upload_images/2824145-29751b2dcb5484f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们使用 `git log -p` 命令，我们能够得到一下输出：

![git-log-p2.jpg](https://upload-images.jianshu.io/upload_images/2824145-518a0e1d2a9a4d74.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述输出信息中，包含5个比较重要的信息：

- 第一个：`diff`显示了原始版本与新版本间的差异，我们现在看到的`IjkVideoView.java`文件，其中`a`/app/../IjkVideoView.java为该文件的第一个版本，`b`/app/../IjkVideoView.java为新版本。
- 第二个：显示了第一个版本文件的哈希值与文件更改后的哈希值，这些哈希值和提交的 `SHA` 是不同的。
- 第三个：也是显示了不同版本的文件差异，其中 `-` 表示旧版本， `+` 表示新版本。
- 第四个：添加的行所在的位置以及添加了多少行。
  - `-333,8`表示原始版本（用 `-` 表示）,从338行开始，显示了8行。
  - `+333,10`表示新版本（用 `+` 表示）,从338行开始，现在变成了10行，这10行在命令窗口中显示了。
- 第五个：表示了在commit中实际进行的更改。
  - 用红色并以 减号（`-`）开头的行是位于文件原始版本中，但是被 commit 删除的行
  - 用绿色标示并以 加号（`+`）开头的行是 commit 新加的行

### 查看特定的commit

上文中涉及到的命令，是对全部的历史记录进行浏览，如果说能够单独的显示某个提交信息是不是很棒呢？在Git中有两种方式来查看特定的提交。

- 使用 git log 提供你要查看的 commit 的 SHA
- 使用 git show

这里我们先 git log 方式，然后再学习 git show。

#### 使用 git log 方式

在上文中，我们已经学会如何使用以下命令输出信息：

- git log
- git log --oneline
- git log --stat
- git log -p

但是你是否知道，可以向所有这些命令提供 commit 的 SHA 作为最后一个参数？例如：

```bash
git log 543019ea
```

>在Git中支持完整的 `SHA` 与 `SHA前七个字符` 作为具体的查询条件。

需要注意的是使用`git log`+ `SHA` 的这种方式，并不会单独的显示某个提交。而是命令行将从这条提交开始输出历史记录，你仍然可以通过Git提供的分页器，查看该条提交信息之后的记录。

#### 使用 git show 方式

使用 `git show + SHA` 的方式，可以显示特定的提交信息。如

```bash
git show 543019ea
```

`git show`默认情况下会显示：

- commit头信息
- 作者
- 日期
- commit 消息
- 具体的文件差异

这里就不再暂时示例图片了，希望大家多多练习，并查看最终效果吧。

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

最后还是回到我们熟悉的图形化界面的使用流程中。我们来看看IDEA为我们提供了哪些便利吧。

通过依次点击编译器底部的`Version Control`->`Log`,我们能得到如下界面：

![studioLog界面显示.jpg](https://upload-images.jianshu.io/upload_images/2824145-9e8a5d877e0713c4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在分别上图标注的内容进行介绍：

1. 提交记录展示列表：在该区域显示了我们所有的提交内容，包含commit信息、作者、日期。
2. 搜索框：我们能根据关键字搜索到我们想要查询的提交信息。
3. User筛选框：我们可以筛选其他作者的提交信息
4. Date筛选框：根据日期来查询提交信息。
5. 通过点击相关提交记录，我们能查看对应提交对文件的操作（增加，删除，更改），如果点击相关文件，可以查看当前提交内容与上个版本的区别。
6. 通过点击相关提交记录，我们能查看到详细的提交信息，包括SHA值，提交用户，邮件等其他信息。

### 最后

站在巨人的肩膀上，才能看的更远~

- [Git-基础-查看提交历史](https://git-scm.com/book/zh/v2/Git-基础-查看提交历史)
- [视频推荐-->用Git进行版本控制](https://cn.udacity.com/course/version-control-with-Git--ud123)
