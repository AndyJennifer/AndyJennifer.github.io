---
title: Git新手教程-撤销更改(七)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们学习了标签、分支、和合并。现在我们将学习Git中另外的三个命令`git commit --amend` , `git revert` , `git reset`。下面简单的介绍中几个命令的功能：

- `git commit --amend` ：可以修改最后一次提交中的内容，加东西，加文件。修改 commit 信息。
- `git revert` ：可以撤销对应的 commit ，并产生一个新的 commit 。
- `git reset` ：可以清除最近的 commit 。并不能任意删除，需要按照顺序。

### git commit --amend

在平时的项目开发中，有时候我们可能提交完相应文件后，才发现漏掉了几个文件没有添加，或者我们 commit 消息并没有书写完整或有错别字。那这个时候我们该整么办呢？或许我们会执行一个新的提交来添加我们遗漏的内容，但是这样一点都不优雅！！在Git中为我们提供了带有 `--amend` 选项的提交来修改我们`最近`的提交:

> amend 中文意思是修改、改善、改进。

```bash
git commit --amend
```

这里大家可能还是不是很明白，我们看下面这个简单的例子：

![撤销更改1.png](https://upload-images.jianshu.io/upload_images/2824145-d0b6ea830d1e9fe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述例仓库中我们创建了一个 commit ，该 commit 消息并没有书写完整。这个时候我们想修改它，那么我们就可以使用命令 `git commit --amend` ，当输入该命令后，我们能得到如下弹窗：

> 你也可以使用 `git commit --amend -m + 提交信息` 跳过编辑器，直接修改 commit 信息。

![撤销更改2.png](https://upload-images.jianshu.io/upload_images/2824145-0d525cb66eb42171.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候，我们就可以完善该 commit 信息，然后保存并离开。这里我改成了 `删除了多余的语句` ，这时我们再使用 `git log` 命令，我们会发现我们的 commit 消息已经被更改了。如下所示：

![撤销更改3.png](https://upload-images.jianshu.io/upload_images/2824145-0de4916bf1cdc5f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然在上述例子中，我们只是简单的修改了 commit 信息，并没有修改或添加一些新的文件，如果你修改或添加了新的文件，并想将这些修改的文件添加到最近的 commit 中去的话，那么你可能要经历以下步骤：

- 编辑文件
- 保存文件
- 暂存文件 (git add)
- 运行 git commit --amend

在实际的项目中，如果你想修改最近的 commit ，那么你需要使用 `git commit --amend` 来更新最近的 commit ，而不是创建新的 commit 。

### git revert

在上述例子中，我们知道了如何修改最近的 commit ，但是如果我们想还原这个 commit 。那这个时候，我们又该怎么办呢？我们可以使用 `git revert` 命令来告诉 Git 还原之前创建的commit，该命令使用的方式如下：

> 当你告诉 Git 还原（revert） 具体的 commit 时，git 会执行和 commit 中的更改完全相反的更改。假设 commit A 添加了一个字符，如果 Git 还原 commit A，那么 Git 将创建一个 `新的 commit` ，并删掉该字符。如果删掉了一个字符，那么还原该 commit 将把该内容添加回来！

```bash
git revert <SHA-of-commit-to-revert>
```

还是以上述例子来进行讲解，比如我们想还原下图中红色框中的 commit ：

![撤销更改4.png](https://upload-images.jianshu.io/upload_images/2824145-0de4916bf1cdc5f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以使用 `git revert b71b40` ，需要注意的是使用该命令，默认会创建一个新的提交。如下图所示：

> 这里 `b71b40` 是对应 commit 的 SHA 的前七个字符，当然你也可以使用完整的 SHA 。

![撤销更改5.png](https://upload-images.jianshu.io/upload_images/2824145-e10e3a4bd493b3ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一般情况下，我们可以使用Git系统默认的 revert 信息。当我们保存并退出后，再使用 `git log` 命令查看我们的日志提交记录，我们能得到下图：

![撤销更改6.png](https://upload-images.jianshu.io/upload_images/2824145-000528eef2e0175b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### git reset

#### git reset 简介

在上述例子中，我们学会了如何还原一个提交，但是如果我们本是这个提交就是错误的，我们并不想要这个提交，也不想仓库记录中包含 `revert` 的提交记录。这个时候我们又该怎么办呢？我们可以使用 `git reset` 命令来告诉 Git 重置 commit 。

重置（reset） 似乎和 还原（revert） 相似，但它们实际上差别很大。还原会创建一个新的 commit，并还原或撤消之前的 commit。但是重置会清除 commit！

> `git reset` 命令是少数几个可以从仓库中清除 commit 的命令，如果某个 commit 不再存在与仓库中，它所包含的内容也会消失。不过不用担心，Git 会在完全清除任何内容之前，持续跟踪提交记录大约30天，我们可以通过 `git reflog` 来查看仓库中所有的改变。关于 `git reflog` 你可以查看下方内容：

- [git reflog](https://git-scm.com/docs/git-reflog)
- [Rewriting history](https://www.atlassian.com/git/tutorials/rewriting-history)

#### git reset 命令的使用

`git reset` 命令相比其他 Git 命令功能要多一点，可以用来：

- 将 HEAD 和当前分支指针移到目标 commit。
- 清除 commit 。
- 将 commit 的更改移动到暂存区中。
- 取消暂存 commit 的更改。

`git reset` 并不直接使用 commit 的 SHA ，而是使用特殊的 `"祖先引用"` 来告诉 Git 将 `HEAD` 指针移动到哪个commit。我们来看看这些特殊的符号。

- `^` ： 表示父 commit
- `~` ： 表示第一个父 commit

我们可以通过以下方式引用之前的 commit：

- 当前 commit 的父 commit

```bash
HEAD^
HEAD~
HEAD~1
```

- 当前 commit 的祖父 commit

```bash
HEAD^^
HEAD~2
```

- 当前 commit 的曾祖父 commit

```bash
HEAD^^^
HEAD~3
```

`^` 和 `~` 的区别主要体现在通过合并而创建的 commit 中。合并 commit 具有两个父级。对于合并 commit，`^` 引用用来表示第一个父 commit，而 `^2` 表示第二个父 commit。第一个父 commit 是当你运行 git merge 时所处的分支，而第二个父 commit 是被合并的分支。

我们看下面的例子，来一起来理解：

> 可以使用 `git log --oneline  --graph --all` 来查看所有的分支信息。

![显示所有分支信息.png](https://upload-images.jianshu.io/upload_images/2824145-e19d7ffed53186d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为 `HEAD` 指向 `b71b405` commmit。

- 那么 `f8d5383` commit 我们可以这样表示：
  - HEAD^
  - HEAD~1
- 那么 `9fdb3f6` commit 我们可以这样表示：
  - HEAD^^2（ `^2` 表示合并分支上的父提交）
- 那么 `48099db` commit 我们可以这样表示：
  - HEAD^^
  - HEAD~2

了解了祖先引用，现在我们来了解 `git reset` 命令的使用：

```bash
git reset <reference-to-commit>
```

一般情况下，使用该命令，我们会添加如下选项：

- `--mixed` (默认不指定任何选项)移动到工作目录，不会暂存我们的文件，工作内容与原来相同，但是SHA不同，因为时间戳不同。
- `--soft` 移动到暂存区，这些改动仍然存在，而且已经暂存好了。
- `--hard` 会删除对应所有的提交的内容。

直接理解这些选项比较困难，下面我们将结合图片与实际例子来进行讲解。

##### --mixed 例子

在下图中，假如我们的仓库中已经有了如下提交 `D、E、F、G、H`，其中 `master` 指向最近的提交 `H`，`HEAD` 指向 `master`。

这个时候我们如果使用 `git reset --mixed HEAD~1` 那么会将 `master` 与 `HEAD` 将会指向前一个提交 `G`。同时 `G` 提交会移动到工作目录中。如下图所示：

>注意，`git reset --mixed HEAD~1` 等同于 `git reset --mixed HEAD^` ，也等同于 `git reset HEAD~1` (git reset 命令默认选项为 --mixed)。

![rest_mixed演示.jpg](https://upload-images.jianshu.io/upload_images/2824145-046cfe19a3409594.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当 `H` 提交修改的文件被移动到到工作目录后，文件的状态都为 `modifed`，也就是我们需要重新添加到暂存区，然后进行 commit 。我们继续看下面的例子：

![mixed_展示.png](https://upload-images.jianshu.io/upload_images/2824145-7b7bcf70c52623de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述仓库中有3个提交，其中 `HEAD` 指向 `bb780f9` 上的 `master` , 这个时候如果我们运行 `git reset HEAD~` 命令，会将 commit `bb780f9` 中的文件移动到工作目录中，如下所示：

![mixed_2.png](https://upload-images.jianshu.io/upload_images/2824145-5cf834b7f76dc83d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用 `git status` 来查看我们的仓库状态，我们会发现使用 `--mixed` 选项，是不会暂存我们的更改的，也就是不会将相应提交的文件放入暂存区中。

##### --soft 例子

当使用 `--soft` 选项时，不仅会移动 `master` 与 `HEAD` 指针，还会将相应修改添加到暂存区中，如下所示：

![reset_soft演示.jpg](https://upload-images.jianshu.io/upload_images/2824145-86bb9a9bbf9c85a8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们继续查看下面的例子：

![soft展示.png](https://upload-images.jianshu.io/upload_images/2824145-caef357d1f2b4198.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### --hard 例子

最后我们再来看看 `--hard` 选项：

![reset_hard演示.jpg](https://upload-images.jianshu.io/upload_images/2824145-903e12026efab806.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用 `--hard` 将清除对应 commit 所作的更改，继续查看下面的👇的例子：

![hard展示.png](https://upload-images.jianshu.io/upload_images/2824145-bc65d682b862de62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当使用了 `--hard` 选项后，发现仓库中对应的提交消失了。

##### 取消暂存区中的内容

当我们直接使用 `git reset` 或 `git reset HEAD`时，表示清除暂存区中的 commit 。下面我们来看这个例子：

```bash
edit
git add A.java B.java   (1）
git reset               (2)
```

1. 我们添加了两个Java文件到暂存区中。
2. 但是在后续中，我们发现我们还需要对这两个文件进行修改，那么我们可以使用 `git reset` 命令将这另个文件从暂存区中移除。当然我们也可以使用 `git reset HEAD` 命令，这两个命令是等价的。

那么如果我们只想将 `B.java` 文件从暂存区中移除，那我们又该怎么办呢？我们可以使用如下命令：

```bash
git reset HEAD B.java
```

当然在Git中支持多个文件的取消暂存，具体命令如下：

```bash
git reset HEAD <file>...
```

在上述命令中 `<file>...` 代表一个或多个文件。

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

最后还是回到我们熟悉的图形化界面的使用流程中。我们来看看IDEA为我们提供了哪些便利吧。

#### 使用 git commit --amend

在创建相应 `commmit` 时，我们可以勾选下图中的
`Amend commit` 选项。如下所示：

![ide_amend.png](https://upload-images.jianshu.io/upload_images/2824145-6bdf8b264ab654dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上述操作与使用 `git commit --amend` 命令的效果一样。

#### 使用 git revert

通过依次点击编译器底部的`Version Control`->`Log`，然后选择想要 revert 的 commit ，点击鼠标`右键`，选择 `Revert Commit` 就可以啦~

![ide_revert.png](https://upload-images.jianshu.io/upload_images/2824145-fee04c55bcf1c6c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 使用 git reset

通过依次点击编译器底部的`Version Control`->`Log`，然后选择想要 reset 的 commit ，点击鼠标`右键`，选择 `Reset Current Branch to Here..` 就完成第一步啦~

![ide_reset_1.png](https://upload-images.jianshu.io/upload_images/2824145-b1a4faa24e390f4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然在第二步中，我们需要选择 `git reset` 中的选项，在 IDEA 中提供了四种选项 `Soft、Mixed、Hard、Keep` ，如下所示：

![ide_reset_2.png](https://upload-images.jianshu.io/upload_images/2824145-8d64de035894b139.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，我们不熟悉的只有 `--keep` 选项，因为该选项在平时中的项目并不常用，所以这里就不做更多介绍了，有兴趣的小伙伴，可以查看官方文档中 [git reset](https://git-scm.com/docs/git-reset) 中对其的介绍。

### 最后

站在巨人的肩膀上，才能看的更远~

- [git reset](https://git-scm.com/docs/git-reset)
- [撤销更改](https://classroom.udacity.com/courses/ud123/lessons/f02167ad-3ba7-40e0-a157-e5320a5b0dc8/concepts/50740c4a-054c-46da-910a-8d1344b18d33)
