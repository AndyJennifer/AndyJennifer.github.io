---
title: Git新手教程-撤销更改(七)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们学习了标签、分支、和合并。现在我们将学习Git中另外的三个命令，`git commit --amend` , `git revert` , `git reset`。下面简单的介绍中几个命令的功能：

- 使用 `git commit --amend` ：可以修改最后一次提交中的内容，加东西，加文件。修改commit信息。
- 使用 `git revet` ：可以撤销对应的 commit ，并产生一个新的 commit 。
- 使用 `git reset` ：可以清除最近的 commit 。

### git commit --amend

在平时的项目开发中，有时候我们可能提交完相应文件后，才发现漏掉了几个文件没有添加，或者我们 commit 消息并没有书写完整。那这个时候我们该整么办呢？或许我们会执行一个新的提交来添加我们遗漏的内容，但是这样一点都不优雅！！在Git中为我们提供了带有 `--amend` 选项的提交来修改我们`最近`的提交:

> amend 中文意思是修改、改善、改进。

```bash
git commit --amend
```

这里大家可能还是不是很明白，我们看下面这个简单的例子：

![撤销更改1.png](https://upload-images.jianshu.io/upload_images/2824145-d0b6ea830d1e9fe5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述例仓库中我们创建了一个 commit ，该 commit 消息并没有书写完整。这个时候我们想修改它，那么我们就可以使用命令 `git commit --amend` ，当输入该命令后，我们能得到如下弹窗：

> 你也可以使用 `git commit --amned -m + 提交信息`来直接修改 commit 信息。

![撤销更改2.png](https://upload-images.jianshu.io/upload_images/2824145-0d525cb66eb42171.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候，我们就可以修改完善该 commit 信息，然后保存并离开。这里我改成了 `删除了多余的语句` ，这个时候我们再使用 `git log`命令，我们会发现我们的 commit 消息已经被更改了。如下所示：

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

在上述例子中，我们学会了如何还原一个提交，但是如果我们本是这个提交就是错误的，我们并不想要这个提交，我也不想仓库记录中包含 `revert` 的提交记录。这个时候我们又该怎么办呢？我们可以使用 `git reset` 命令来告诉 Git 重置 commit 。

重置（reset） 似乎和 还原（revert） 相似，但它们实际上差别很大。还原会创建一个新的 commit，并还原或撤消之前的 commit。但是重置会清除 commit！

> `git reset` 命令是少数几个可以从仓库中清除 commit 的命令，如果某个 commit 不再存在与仓库中，它所包含的内容也会消失。不过不用担心，Git 会在完全清除任何内容之前，持续跟踪提交记录大约30天，我们可以通过 `git reflog` 来查看仓库中所有的改变。关于 `git reflog` 你可以查看下方内容：

- [git reflog](https://git-scm.com/docs/git-reflog)
- [Rewriting history](https://www.atlassian.com/git/tutorials/rewriting-history)

#### git reset 命令的使用

`git reset` 命令相比其他 Git 命令功能要多一点，可以用来：

- 将 HEAD 和当前分支指针移到目标 commit 。
- 清除 commit 。
- 将commit的更改移动到暂存区中。
- 取消暂存 commit 的更改。

`git reset`并不直接使用 commit 的 SHA ，而是使用特殊的 `"祖先引用"` 来告诉 Git 将 `HEAD` 指针移动到哪个commit。我们来看看这些特殊的符号。

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

了解了祖先引用，现在我们来了解 `git reset`命令的使用方式:

```bash
git reset <reference-to-commit>
```

使用该命令我们可以使用如下选项：

- `--mixed` (默认不指定任何选项)移动到工作目录，不会暂存我们的文件，工作内容与原来相同，但是SHA不同，因为时间戳不同
- `--soft` 移动到暂存区，这些改动仍然存在，而且已经暂存好了。
- `--hard` 删除 会删除对应所有的提交的内容

直接理解起来比较困难，我们还是看下面的例子吧：



### 最后

站在巨人的肩膀上，才能看的更远~
