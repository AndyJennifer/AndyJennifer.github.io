---
title: Git新手教程-标签、分支和合并(六)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在之前的文章中，我们已经对仓库和提交已经有一定的了解了，在该篇文章中，我们将学习`git tag`、`git branch`、 `git checkout` 和 `git merge`。下面简单的介绍一下几个命令的功能:

- 使用`git tag`你可以为特定提交添加标签。标签是提交的额外标记，可以指示有用的信息。
- 使用`git branch` 你可以创建分支。用于并行开发项目的不同功能。而不会对哪些提交属于那个功能而感到困惑。
- 使用`git checkout`你可以在不同的分支和标签之间进行切换。
- 使用`git merge`可以将不同分支上的更改自动合并在一起

### 标签

我们先从最简单的标介绍开始。

假设我们已经向仓库提交了A、B、C、D四个commit，其中最后的一个提交也就是`D`，标志着我们项目的1.0版本。如何在Git中指出这一点呢？大家可能会觉得，我们可以在D的提交中说明这是我们的版本1.0。但是这并不是最理想的。Git像其他版本控制系统一样，提供了给提交打`Tag(标签)`的功能。也就是是说我们可以给`D`提交打上标签。如下所示：

> `Tag(标签)`一般表示比较重要的节点信息。

![Tag展示.jpg](https://upload-images.jianshu.io/upload_images/2824145-e82ce4dcad43c93a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即使更多的提交被添加到仓库中（在下图中，我们假设在D提交之后又陆续的提交了E、F、G三个提交)，`Tag(标签)`仍然锁定着某个提交，如下所示：

![Tag展示2.jpg](https://upload-images.jianshu.io/upload_images/2824145-a96ba9e37b54d5fc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

既然标签有着如此重要的作用，那现在我们来看看在Git中如何创建标签吧。在Git中标签有两种类型：`轻量标签`与`附注标签`。轻量标签在Git中，是不存储任何的信息，只是一个特定提交的引用。而附注标签存储在Git数据中，且附注标签中包含了打标签者的名称、电子邮件，日期等信息，一般情况我们都会使用附注标签。

#### 创建轻量标签

创建轻量标签的方式非常简单使用`git tag`+`标签名称`。如下所示：

```bash
git tag v1.0.0
```

#### 创建附注标签

创建附注标签非常简单，需要在创建轻量标签的方式上，添加选项`-a`。如下所示：

```bash
git tag -a v1.0.0
```

通过上述方式，Git会运行你配置的文本编辑器输入标签信息。如下所示：

![git_tag_3.png](https://upload-images.jianshu.io/upload_images/2824145-8ea44be6c92264ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>上图中，我配置的是Sublime Text，当然你也可以使用`git config --global core.editor`命令配置你喜欢的编辑器。

除了使用上述方式创建附注标签以外，我们还可以使用`-m`选项来指定存储在标签中的信息，如果你添加了`-m`选项，那么Git就不会再运行编辑器了。

```bash
git tag -a v1.0.0 -m "发布新版本v1.0.0"
```

#### 轻量标签与附注标签的区别

在讲解了轻量标签与附注标签的使用之后，我们来看一下这两个标签的实际区别，我们可以通过`git show`命令查看实际的区别，我们先查看附注标签，如下所示：

![git_show_1.png](https://upload-images.jianshu.io/upload_images/2824145-c1b7bf2c9a0ec3a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过使用`git show v1.0.0`我们你可以看到命令行的输出中是包含Tag信息的：

```bash
tag v1.0.0
Tagger: AndyJennifer <1225868370@qq.com>
Date:   Sun Sep 15 18:20:28 2019 +0800

发布新版本v1.0.0
```

而创建的轻量标签中，是不包含该信息的，如下所示：

![git_show_2.png](https://upload-images.jianshu.io/upload_images/2824145-274c2943992112fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是，创建的标签总是与commit进行绑定的。比如在上图中标签是指向`441796`这个提交的。

#### 标签的删除

要删除掉本地的标签，我们可以使用命令`git tag -d <tagname>`。例如使用下面的命令删除标签:

```bash
git tag -d v1.0.0
Deleted tag 'v1.0.0' (was 3132ff5)
```

其中`-d`选项表示`delete(删除)`。

#### 向以前的提交的 commit 添加标签

通过运行`git tag -a v1.0.0`，我们可以给最近的commit添加标签，但是如果你想向仓库中很久之前的Commit添加标签呢？很简单！我们可以在原来的命令基础上，直接添加对应的commit的`SHA`即可。

```bash
git tag -a v1.0.0 abd12d0
```

在上述命令中，我们为`abd12d0`这个提交打上了标签。

### 分支

在Git中，分支就像是科幻电影中的`平行世界`。每个平行世界都运行在独立的环境中，在正常情况下，每个平行世界中所作的事，对于现实世界或另一个平行世界是无感知的且不影响的。在某些特殊的情况下，平行世界与现实世界可能重合，那么就会产生混乱与冲突。那这个时候就需要我们来解决这些冲突与问题。

![平行世界.jpg](https://upload-images.jianshu.io/upload_images/2824145-9abd4b16881e58c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

扯太远了，其实分支在Git中是用于进行开发或者对项目进行修正，因为所有的修改都是在分支中进行，所以并不会影响到项目（项目一般在主分支上，一般是`master`分支）。当我们在分支中进行更改后，我们可以将分支中的内容合并到分支上，这种分支组合过程，称为`合并（merge)`。后续我们会一一讲解这些内容。

那么接下来，我们来看看Git中分支的相关使用方式和重要的概念。

#### master分支与Head指针

在下图仓库中，我们创建了一些提交，并在提交`D`中创建了名为`v1.0`的标签，除了标签以外还包含一个隐藏的`master`分支。在我们没有创建分支的情况下，Git会为我们创建一个名为`master`的分支。其实`master`的本质其实是一个`指针`。

![分支1.png](https://upload-images.jianshu.io/upload_images/2824145-4324a593bb3e2ab1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>master并没有什么特殊的意义，它只是第一个默认分支名。

随着我们不断向仓库进行提交，这些提交都会添加到该分支上。同时`master指针`也会指向对应提交。如下所示：

![分支2.png](https://upload-images.jianshu.io/upload_images/2824145-d8d2162b0e2e757c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，`masetr` 指针随着提交而不断移动。图中`红线`所连接的提交都属于`master分支`上。

>相对于`Tag(标签)`，分支指针不会永久的指向某次提交，而是会随着提交不断的移动。

#### HEAD指针

在某些情况下，我们可能会创建另一个分支。在下图中，我们又创建了另一分支 `branch1` 。

![分支3.png](https://upload-images.jianshu.io/upload_images/2824145-59aac87a94310dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候如果我们再进行一个提交，哪个分支会移动呢？是 `master` 还是 `branch1` 分支呢？这里就涉及到另一个知识点----> `HEAD` 指针。

在Git中，`HEAD` 总是指向当前活跃的分支。在你没有切换分支的情况下，**默认指向master分支**。如下图所示。

![分支4.jpg](https://upload-images.jianshu.io/upload_images/2824145-2f6693469ebb4b42.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为当前 `HEAD指针` 指向 `MASTER` 分支，所以当我们再向仓库提交一个`I`提交对象时，该提交仍然会在 `MASTER` 分支上。如下所示：

![分支5.jpg](https://upload-images.jianshu.io/upload_images/2824145-0088d194d5e6439e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然我们可以改变 `HEAD` 指针所指向的内容， 使用`checkout`命令，我们可以切换`HEAD`指针所指向的分支。比如我们可以通过命令`git checkout branch1`，将 `HEAD` 指针指向 `branch1` 。

当我们切换分支到`branch1`后，我们后续的提交，都会添加到 `branch1` 分支上。比如下图，我们又创建了一个`A提交`，那么这时`branch1`分支指针会指向该提交。

![分支6.jpg](https://upload-images.jianshu.io/upload_images/2824145-fe3aa6cc999fa477.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 分支中的提交的可见性

需要注意的是，分支中的提交对于其他分支来说是不可见的。

![分支7.jpg](https://upload-images.jianshu.io/upload_images/2824145-5f0010fe90016128.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，如果我们当前切换到了`master分支上`，那么在我们工作目录与文件系统中，`branch1`分支的`A`与`branch2`分支中的`J`，这两个提交对文件造成的更改，将不会出现在`master分支`上的任何文件中。如果需要查看这个两个对应的文件修改，只需要切换到我们想要查找的提交所在的分支就行了。

我们基本已经熟悉了分支的相关概念与整体流程后，下面我们来学习一下
相关命令的使用。

#### 创建分支

创建分支其实很简单，只需使用 `git branch` 并提供要创建的分支对应的名称。如果你想创建一个叫做`"branch1"`的分支，只需运行以下命令：

```bash
git branch branch1
```

需要注意的是，如果你在某个提交上创建了一个分支，那么该提交之前的所有提交也是属于当前分支，在下图中，在`H`提交中使用了创建分支命令创建了`branch1`分支，那么在`H`提交之前的`D、E、F、G`也都属于`branch1`分支。

![分支3.png](https://upload-images.jianshu.io/upload_images/2824145-59aac87a94310dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中`branch1`分支如蓝线所示。

##### 以前的提交的 commit 添加分支

当然，与创建标签一样，我们也可以向以前的提交的 commit 添加分支。在下述命令中，我们在 `SHA` 为`dfs14fo`的提交中创建了test分支。

```bash
git branch test dfs14fo
```

#### 切换分支

要切换到另一个分支，我们需要使用命令`git checkout`命令，比如我们需要切换到分支`dev`上，那么我们可以使用如下命令：

```bash
git checkout dev
```

通过上述操作，`HEAD`指针会指向`dev`，需要注意的是，当你切换分支时，在你的工作目录中，会删除其他分支中`commit`引用的所有文件，同时会引用`dev`分支中的 `commit`引用的文件。当然你不用担心真的删除了你另一分支上的文件。Git会将它们都存储在仓库中。当你切换回来后，又会将对应分支引用的文件显示在你的工作目录中。

#### 分支的查看

如果你需要查看当前项目的分支，可以使用`git branch`命令，通过该命令我们不仅可以知道仓库中所创建的分支，还能查看当前活跃的分支，也就是`HEAD`指针指向的分支。

![git_branch.png](https://upload-images.jianshu.io/upload_images/2824145-a926cef561cb5c2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，`星号(*)` 标记的为当前活跃的分支。

#### 分支删除

在上文中，我们提到了，分支一般用于进行开发或对项目进行修正。当我们将分支的更改合并到`master`分支后，我们可能就不再需要该分支了。那么这个时候如果我们想删除分支，那么我们使用命令 `git branch -d` + 分支名称来删除我们想要删除的分支。这里我们以删除`dev`分支为例：

```bash
git branch -d dev
```

在删除分支的时候，我们需要注意以下两点：

- 如果我们已经使用 `git checkout dev` 命令切换到dev分支上的话，那么我们是不能删除`dev`分支的。我们需要切换到其他分支上，才能删除该分支。
- 如果 `dev`分支上有任何的commit，那么通过`-d`选项是不能删除该分支的，你需要使用大写的`D`选项，也就是使用 `git branch -D dev`。

### 高效分支

我们已经做出了所有需要做出的更改！很棒！

我们已经在三个不同的分支上进行了多项更改。我们在 git log 输出结果中看不到其他分支，触发切换到某个分支。如果能在 git log 输出结果中看到所有分支，是不是很棒？

你到现在为止已经知道，git log 命令非常强大，可以显示此信息。我们将使用新的 `--graph` 和 `--all` 选项

```bash
git log --oneline --decorate --graph --all

```

等同于

```bash
git log --oneline  --graph --all
```

### 合并

### 合并冲突

### 最后

站在巨人的肩膀上，才能看的更远~

 - [Git-分支-分支简介](https://git-scm.com/book/zh/v2/Git-分支-分支简介)

