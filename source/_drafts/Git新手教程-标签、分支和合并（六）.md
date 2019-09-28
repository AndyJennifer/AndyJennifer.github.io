---
title: Git新手教程-标签、分支和合并(六)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在之前的文章中，我们已经对仓库和提交已经有一定的了解了，在该篇文章中，我们将学习`git tag`、`git branch`、 `git checkout` 和 `git merge`。下面简单的介绍一下几个命令的功能:

- 使用 `git tag` 你可以为特定提交添加标签。标签是提交的额外标记，可以指示有用的信息。
- 使用 `git branch` 你可以创建分支。用于并行开发项目的不同功能。而不会对哪些提交属于那个功能而感到困惑。
- 使用 `git checkout` 你可以在不同的分支和标签之间进行切换。
- 使用 `git merge` 可以将不同分支上的更改自动合并在一起

### 标签

我们先从最简单的标签开始。

假设我们已经向仓库提交了 A、B、C、D 四个commit，其中最后的一个提交也就是`D`，标志着我们项目的1.0版本。如何在Git中指出这一点呢？大家可能会觉得，我们可以在D的提交中说明这是我们的版本1.0。但是这并不是最理想的。Git像其他版本控制系统一样，提供了给提交打`Tag(标签)`的功能。也就是说我们可以给`D`提交打上标签。如下所示：

> `Tag(标签)`一般表示比较重要的节点信息。

![Tag展示.jpg](https://upload-images.jianshu.io/upload_images/2824145-e82ce4dcad43c93a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

即使更多的提交被添加到仓库中（在下图中，我们假设在D提交之后又陆续的提交了E、F、G三个提交)，`Tag(标签)`仍然锁定着某个提交，如下所示：

![Tag展示2.jpg](https://upload-images.jianshu.io/upload_images/2824145-a96ba9e37b54d5fc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

既然标签有着如此重要的作用，那现在我们来看看在Git中如何创建标签吧。在Git中标签有两种类型：`轻量标签`与`附注标签`。

- 轻量标签在Git中，是不存储任何的信息，只是一个特定提交的引用。
- 附注标签存储在Git数据中，且附注标签中包含了打标签者的名称、电子邮件，日期等信息，一般情况我们都会使用附注标签。

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

除了使用上述方式创建附注标签以外，我们还可以使用 `-m` 选项来指定存储在标签中的信息，如果你添加了 `-m` 选项，那么Git就会绕过你配置的编辑器。如下所示：

```bash
git tag -a v1.0.0 -m "发布新版本v1.0.0"
```

#### 轻量标签与附注标签的区别

在讲解了轻量标签与附注标签的使用之后，我们来看一下这两个标签的实际区别，我们可以通过`git show`命令查看实际的区别，我们先查看附注标签，如下所示：

![git_show_1.png](https://upload-images.jianshu.io/upload_images/2824145-c1b7bf2c9a0ec3a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过使用`git show v1.0.0`，你可以看到命令行的输出中是包含Tag信息的：

```bash
tag v1.0.0
Tagger: AndyJennifer <1225868370@qq.com>
Date:   Sun Sep 15 18:20:28 2019 +0800

发布新版本v1.0.0
```

而创建的轻量标签中，是不包含该信息的，如下所示：

![git_show_2.png](https://upload-images.jianshu.io/upload_images/2824145-274c2943992112fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是，创建的标签总是与commit进行绑定的。比如在上图中，标签是指向 `441796...` 这个提交的。

#### 标签的删除

要删除掉本地的标签，我们可以使用命令 `git tag -d <tagname>` 。例如使用下面的命令删除标签:

```bash
git tag -d v1.0.0
Deleted tag 'v1.0.0' (was 3132ff5)
```

其中 `-d` 选项表示`delete(删除)`。

#### 向以前的提交的 commit 添加标签

通过运行 `git tag -a v1.0.0` ，我们可以给最近的commit添加标签，但是如果你想向仓库中很久之前的commit添加标签呢？很简单！我们可以在原来的命令基础上，直接添加对应的commit的 `SHA` 即可。

>使用 `git log` 命令查看历史提交记录。这里我们可以提供完整的`SHA`，或`SHA` 的 `前七个` 字符，

```bash
git tag -a v1.0.0 abd12d0
```

在上述命令中，我们为 `abd12d0` 这个提交打上了标签。

### 分支

在Git中，分支就像是科幻电影中的`平行世界`。每个平行世界都运行在独立的环境中，在正常情况下，每个平行世界中所作的事，对于现实世界或另一个平行世界是无感知的且不影响的。在某些特殊的情况下，平行世界与现实世界可能重合，那么就会产生混乱与冲突。那这个时候就需要我们来解决这些冲突与问题。

![平行世界.jpg](https://upload-images.jianshu.io/upload_images/2824145-9abd4b16881e58c0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

扯太远了，分支在Git中，主要用于项目的开发或者对项目进行修正。因为所有的修改都是在分支中进行，所以并不会影响到项目（项目一般在主分支上，一般是 `master` 分支）。当我们在分支中进行更改后，我们可以将分支中的内容合并到分支上，这种分支组合过程，称为`合并（merge)`。后续我们会一一讲解这些内容。

那么接下来，我们来看看Git中分支的使用方式和重要的概念。

#### master 分支与 HEAD 指针

在下图仓库中，我们创建了一些提交，并在提交`D`中创建了名为 `v1.0` 的标签，除了标签以外还包含一个隐藏的 `master` 分支。在我们没有创建分支的情况下，Git会为我们创建一个名为 `master` 的分支。其实 `master` 的本质其实是一个 `指针` 。

![分支1.png](https://upload-images.jianshu.io/upload_images/2824145-4324a593bb3e2ab1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> master 并没有什么特殊的意义，它只是第一个默认分支名。

随着我们不断向仓库进行提交，这些提交都会添加到该分支上。同时 `master指针` 也会指向对应提交。如下所示：

![分支2.png](https://upload-images.jianshu.io/upload_images/2824145-d8d2162b0e2e757c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，`masetr` 指针随着提交而不断移动。图中`红线`所连接的提交都属于 `master` 分支。

>相对于 `Tag(标签)` ，分支指针不会永久的指向某次提交，而是会随着提交不断的移动。

##### HEAD指针

在某些情况下，我们可能会创建另一个分支。在下图中，我们又创建了另一分支 `branch1` 。

![分支3.png](https://upload-images.jianshu.io/upload_images/2824145-59aac87a94310dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候如果我们再进行一个提交，哪个分支会移动呢？是 `master` 还是 `branch1` 分支呢？这里就涉及到另一个知识点----> `HEAD` 指针。

在Git中，`HEAD` 总是指向当前活跃的分支。在你没有切换分支的情况下，默认指向 `master` 分支。如下图所示。

![分支4.jpg](https://upload-images.jianshu.io/upload_images/2824145-2f6693469ebb4b42.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为当前 `HEAD指针` 指向 `master` 分支，所以当我们再向仓库提交一个`I`提交对象时，该提交仍然会在 `master` 分支上。如下所示：

![分支5.jpg](https://upload-images.jianshu.io/upload_images/2824145-0088d194d5e6439e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然我们可以改变 `HEAD` 指针所指向的内容， 使用 `git checkout` 命令，我们可以切换 `HEAD` 指针所指向的分支。比如我们可以通过命令`git checkout branch1`，将 `HEAD` 指针指向 `branch1` 。

当我们切换分支到 `branch1` 后，我们后续的提交，都会添加到 `branch1` 分支上。比如下图，我们又创建了一个 `A提交` ，那么这时 `branch1` 分支指针会指向该提交。

![分支6.jpg](https://upload-images.jianshu.io/upload_images/2824145-fe3aa6cc999fa477.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 分支中的提交的可见性

需要注意的是，分支中的提交对于其他分支来说是不可见的。

![分支7.jpg](https://upload-images.jianshu.io/upload_images/2824145-5f0010fe90016128.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，如果我们当前切换到了 `master` 分支，那么在我们的工作目录与文件系统中， `branch1` 分支的 `A` 与 `branch2` 分支中的 `J` ，这两个提交对文件造成的更改，将不会出现在 `master` 分支上的任何文件中。如果需要查看这个两个对应的文件修改，只需要切换到我们想要查找的提交所在的分支就行了。

我们基本已经熟悉了分支的相关概念与整体流程后，下面我们来学习一下
相关命令的使用。

##### 创建分支

创建分支其实很简单，只需使用 `git branch + 分支名称` 。如果你想创建一个叫做 `"branch1"` 的分支，只需运行以下命令：

```bash
git branch branch1
```

需要注意的是，如果你在某个提交上创建了一个分支，那么该提交之前的所有提交也是属于当前分支，在下图中，在 `H` 提交中使用了创建分支命令创建了 `branch1` 分支，那么在 `H` 提交之前的 `D、E、F、G` 也都属于 `branch1` 分支。

![分支3.png](https://upload-images.jianshu.io/upload_images/2824145-59aac87a94310dd9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中 `branch1` 分支如蓝线所示。

##### 以前的提交的 commit 添加分支

当然，与创建标签一样，我们也可以向以前的提交的 commit 添加分支。在下述命令中，我们在 `SHA` 为 `dfs14fo` 的提交中创建了test分支。

```bash
git branch test dfs14fo
```

#### 切换分支

要切换到另一个分支，我们需要使用命令 `git checkout` 命令，比如我们需要切换到分支`dev`上，那么我们可以使用如下命令：

```bash
git checkout dev
```

通过上述操作，`HEAD` 指针会指向 `dev` ，需要注意的是，当你切换分支时，在你的工作目录中，会删除其他分支中 `commit` 引用的所有文件，同时会引用 `dev` 分支中的 `commit` 引用的文件。当然你不用担心真的删除了你另一分支上的文件。Git会将它们都存储在仓库中。当你切换回来后，又会将对应分支引用的文件显示在你的工作目录中。

#### 分支的查看

如果你需要查看当前项目的分支，可以使用 `git branch` 命令，通过该命令我们不仅可以知道仓库中所创建的分支，还能查看当前活跃的分支，也就是 `HEAD` 指针指向的分支。

在下图中，`星号(*)` 标记的为当前活跃的分支。

![git_branch.png](https://upload-images.jianshu.io/upload_images/2824145-a926cef561cb5c2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 查看所有分支

通过使用 `git branch`命令，我们只能查看当前活跃的分支，如果说我们的项目有不同的分支，我们又想查看所有分支提交内容，那么我们又该怎么办了呢？还记得我们之前提到过的 `git log`指令吗，这里我们将添加 `--graph` 与 `--all`选项。

- `--graph` 选项：将条目和行添加到输出的最左侧。
- `--all` 选项：将显示仓库中的所有的分支信息。

那么我们可以使用如下命令:

```bash
git log --oneline --graph --all
```

> `--oneline` 可以省略掉 commit 中的日期、作者等信息。以最简单的形式显示commit信息。

通过此命令，我们能查看仓库中的所有的分支和 commit 信息：

![显示所有分支信息.png](https://upload-images.jianshu.io/upload_images/2824145-e19d7ffed53186d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 分支删除

在上文中，我们提到，分支一般用于进行开发或对项目进行修正。当我们将分支的更改 `合并(merge)` 到 `master` 分支后，我们可能就不再需要该分支了。那么这个时候如果我们想删除分支，那么我们使用命令 `git branch -d + 分支名称`来删除我们想要删除的分支。这里我们以删除`dev`分支为例：

```bash
git branch -d dev
```

在删除分支的时候，我们需要注意以下两点：

- 如果我们已经使用 `git checkout dev` 命令切换到`dev` 分支上的话，那么我们是不能删除 `dev` 分支的。我们需要切换到其他分支上，才能删除该分支。
- 如果 `dev` 分支上有任何的commit，那么通过 `-d` 选项是不能删除该分支的，你需要使用大写的 `D` 选项，也就是使用 `git branch -D dev`。

### 合并

分支的主要作用就是让我们做出不影响 `master` 分支的更改，当我们在非 `master` 分支上做出更改后，如果觉得不需要该分支上的更改，则可以删掉该分支，或者你可以将给分支上的内容与其他分支进行合并。

>将分支组合到一起的这种行为，我们称之为`合并(merge)`。

在Git中使用 `git merge` 命令来合并分支：

```bash
git merge <other-branch>
```

在Git中 `合并(merge)` 的方式有两种类型。一种是普通合并，另一种是快进合并。下面我们就来分别了解这两种合并方式。

#### 普通合并

假设我们的项目存在这两条分支 `master` 与 `branch1` 两条分支。

![普通合并1.jpg](https://upload-images.jianshu.io/upload_images/2824145-af10415234b3c02d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候我们想将 `branch1` 合并到 `master` 分支上，由于当前`HEAD`指向 `master` 分支，所以当两个分支合并时，将会生成一个合并提交`B`将放置在`master`分支上，并且`master指针`将会向前移动。如下所示：

![普通合并2.jpg](https://upload-images.jianshu.io/upload_images/2824145-6e938d8949dd0e24.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是， `B提交` 会链接`branch1`中的 `4` 与 `A` 提交。同时当分支进行合并时，并不会影响到之前的分支，比如我们仍然可以切换到`branch1`分支上，并创建一个新的提交 `J` ,如下图所示：

![普通合并3.jpg](https://upload-images.jianshu.io/upload_images/2824145-56f371f84ba1ee9f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 快进合并

假设我们的项目存在这两条分支`master`与`branch1`支。且`branch1`分支在`master`分支前面。如下所示：

![快速合并1.jpg](https://upload-images.jianshu.io/upload_images/2824145-73b8790dce00a0a9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为`master`中是不包含 `branch1` 中的提交`(I,K,2)`。如果这个时候我们想将这些提交纳入`master`分支中，也就是需要将 `branch1` 分支 `合并(merge)` 到 `master` 分支中。当我们在`master`分支中使用命令 `git merge branch1` 时，因为 `branch1分支` 在 `master分支` 前，Git会做一个所谓的`快进合并`。如下图所示：

![快速合并2.jpg](https://upload-images.jianshu.io/upload_images/2824145-2d1f6505826e3687.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，`master`分支移动到了`branch1`分支指向的commit。需要注意的是，快进合并并不像普通合并那样再创建一个提交。

### 合并冲突

大部分情况下，我们都能成功的合并分支，但是某些情况下，Git无法自动的进行合并，当合并失败时，就称为`合并冲突`。当出现冲突时，我们需要手动的去修复文件中的冲突。看下面的例子：

在例子中，我们已经创建了一个名为`GitTestProject`的仓库，在仓库中我们已经添加了一个名为`Jvm系列之总目.md`文件到仓库中，如下所示：

![合并演示1.gif](https://upload-images.jianshu.io/upload_images/2824145-9eb857724636c1dc.gif?imageMogr2/auto-orient/strip)

其中 `Jvm系列之总目.md` 文件中的内容如下

![合并演示2.png](https://upload-images.jianshu.io/upload_images/2824145-78e82cf957740957.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候我们通过 `git branch dev` 命令创建了 `dev` 分支，并切换到该分支中，这个时候我们原文中添加了下图中`红框`中的内容。

>注意的是，你需要经常使用`git branch`来查看活跃分支

![合并演示3.png](https://upload-images.jianshu.io/upload_images/2824145-9921633bb4577a16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在修改该文件并保存后，我们可以通过`git status`查看当前仓库的状态，然后我们将在`dev`分支修改的内容进行commit。内容如下所示：

![合并演示4.png](https://upload-images.jianshu.io/upload_images/2824145-bee61934f9a6f9c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们在原来的文件中，添加了一句`Java class文件格式`。接下来，我们通过`git checkout master`切换到 `master` 分支，然后继续修改该文件，同样的修改的内容如红框所示：

![合并演示5.png](https://upload-images.jianshu.io/upload_images/2824145-2c781c67448bd9ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样的，在我们完成修改后，我们将在`master`分支修改的内容进行commit。内容如下所示：

![合并演示6.png](https://upload-images.jianshu.io/upload_images/2824145-cb33af236bd60280.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当提交完毕后，如果我们想将`dev`分支合并到 `master`分支中，使用
`git merge dev`命令，我们可以看到如下报错：

>当使用 `git merge` 命令时，你一定要注意当前所在的分支，你可以通过 `git branch` 或者 `git status` 来查看。

![合并演示7.png](https://upload-images.jianshu.io/upload_images/2824145-00945ed0e7b36fff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，Git告诉我们文件 `Jvm系列之总目录.md` 出现了冲突。这个时候我们再打开该文件，我们可能看到如下内容:

![冲突文件显示.png](https://upload-images.jianshu.io/upload_images/2824145-86fcc387f7f213ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在该文件中显示了一些特殊的一些符号，其实这些符号是Git定义的`合并冲突指示符`，下面对这些指示符进行介绍：

- `<<<<<<< HEAD` 此行下方的所有内容（直到下个指示符）显示了当前分支上的行
- `=======` 表示原始行内容的结束位置，之后的所有行（直到下个指示符）是被合并的当前分支上的行的内容
- `>>>>>>> dev` 是要被合并的分支（此例中是 dev 分支）上的行结束指示符

> 注意一个文件可能在多个部分存在合并冲突，因此检查整个文件中的合并冲突指示符，搜索 <<< 能够帮助你找到所有这些指示符。

当出现冲突时，我们需要手动的删除掉这些指示符，在该例子中，我们想保留两个分支提交的内容，那么我们可以进行如下操作：

![冲突合并.gif](https://upload-images.jianshu.io/upload_images/2824145-ab501c239d214817.gif?imageMogr2/auto-orient/strip)

>注意：因为 `dev` 分支与 `master` 分支修改的是同一文件，那么结合上文我们所讲解的合并的类型，那么上例中的提交为`普通合并`，故需要一个新的提交。

当然，当我们解决冲突后，我们仍然需要将修改的内容 `add` 到暂存区中，然后提交。如下所示：

![合并演示9.png](https://upload-images.jianshu.io/upload_images/2824145-f07998c0a4a23e70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当提交完毕后，我们再使用`git log`命令，我们就能看到我们的提交记录啦。

![合并演示10.png](https://upload-images.jianshu.io/upload_images/2824145-cd75fcc24eac0f94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述提交记录中，我们可以看到 `master` 分支上不仅包含了 `dev` 分支上的提交，还包括了一个合并的提交(merge branch 'dev')。

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

最后还是回到我们熟悉的图形化界面的使用流程中。我们来看看IDEA为我们提供了哪些便利吧。

#### Tag的创建

通过依次点击编译器底部的`Version Control`->`Log`，选择我们想创建`Tag`的commit，然后点击鼠标右键,依次选择`New`->`Tag`,并输入你想输入的 Tag 名称就行啦，具体如下所示：

![创建Tag.gif](https://upload-images.jianshu.io/upload_images/2824145-390858cefa29718c.gif?imageMogr2/auto-orient/strip)

在创建`Tag`成功后，在该commit记录中会有一个灰色的标签。

#### Tag的删除

标签的删除也特别简单，在Git提交记录中点击包含我们所创建的标签的commit,然后点击随便右键，依次选择`标签名称`->`delete`。就可以完成标签的删除啦。具体如下所示；

![删除标签.gif](https://upload-images.jianshu.io/upload_images/2824145-13e82d0dcd43e912.gif?imageMogr2/auto-orient/strip)

#### 分支的创建

在 `IntelliJ IDEA` or `Android Sutdio`中，创建分支的地方大概有三个。我先说一下他们的具体位置，与他们之间的区别。

##### 第一种方式

在编译器中，我们只要选择工具栏中的`VCS`->`Git`->`Branches`，就可以从创建分支啦。

![创建分支方式1.png](https://upload-images.jianshu.io/upload_images/2824145-a2e046f9c3db3bfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上述操作后，会弹出如下选择框，这个时候我们只要选择 `New Branch` 选项就可以创建分支了。

![创建分支方式1.2.png](https://upload-images.jianshu.io/upload_images/2824145-db2acd3fe0b49de3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面对该选择框中的内容进行简单的介绍：

- New Branch: 创建新的分支。
- Checkout Tag or Revision...:切换到相应Tag或分支。
- Local Branches:本地所有的分支。
- Remote Branches:远程分支。(关于远程分支，会在后文介绍)。

需要注意的是：通过该种方式创建的分支。分支指针所指向的commit，是你所在分支下`最新的commit!!!!`。

##### 第二种方式

第二种方式创建分支，是通过点击编译器`最右下角`的 `Git:master`（该左下角的内容可能变化，比如你切换到了dev分支上，那么这个时候显示的是 `Git:dev` 。这里以 `master` 分支为例，来创建分支。

![创建分支方式2.png](https://upload-images.jianshu.io/upload_images/2824145-b4fdf90fcf91272d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过该种方式创建的分支。分支指针所指向的commiit，是你所在分支下`最新的commit!!!!`。

##### 第三种方式

通过依次点击编译器底部的`Version Control`->`Log`，然后选中我们需要创建分支的commmit,然后点击鼠标右键，选择 `New` ->`Branch`

![创建分支方式3.png](https://upload-images.jianshu.io/upload_images/2824145-ed1e6b64405f8a2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上述这种方式所创建的分支，分支指针所指向的commit为你选中的commit。

#### 分支的删除

分支的删除也比较简单，通过依次点击编译器底部的`Version Control`->`Log`，然后找到有分支的commit，点击鼠标`右`键，找到你要删除的分支名称，然后选择 `delete` 就可以删除分支了。这里以删除分支 `fix-23` 为例：

![分支的删除.png](https://upload-images.jianshu.io/upload_images/2824145-e53f29ab141907ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 分支的合并

分支的合并也同样的简单。

- 要么我们只要选择工具栏中的`VCS`->`Git`->`Branches` 弹出下列选择框，
- 要么是通过点击编译器`最右下角`的 `Git:master`（该左下角的内容可能变化，比如你切换到了 `dev` 分支上，那么这个时候显示的是 `Git:dev`）

然后我们就可以选择相应的分支合并到对应分支下了，比如这里我们以`master` 分支需要合并 `dev` 分支为例：

![分支的合并展示.png](https://upload-images.jianshu.io/upload_images/2824145-de84024c86d9a890.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述图中，我们只用选择 `Merge into Current` 就可以了，需要注意的是，当我们选择合并分支时，我们需要切换到正确的的分支上。

##### 分支的切换

怎样切换分支？参考上图，当我们选择了 `dev` 分支后，右方有一个`Checkout` 选项，只要点击该选项，我们就能切换到对应分支啦~~~

#### 解决冲突

当合并分支出现冲突的时候，编译器会提示我们合并冲突，会弹出如下代码框：

![冲突展示框.png](https://upload-images.jianshu.io/upload_images/2824145-1ef2eb3031d3c2c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

整个代码块分为三个部分：

- 左边(`Left`) `Your version` : 代表你当前分支上的更改。
- 右边(`Right`) `Change from branch` :代表你合并的分支。
- 中间的 `Result` 代表经过处理后的内容。

内容区域的代码颜色分为四个部分：

- 红色区域：代表当前分支和合并分支都编辑过的内容，属于冲突
- 蓝色区域：代表被单方面编辑过的内容，属于更改。
- 灰色区域：代表被删除的内容，属于更改。
- 绿色区域：代表新增的内容，属于更改。

对于冲突与更改，我们都可以使用左侧或右侧的 `>>` 按钮来应用你想应用的更改，如果你并不想应用这些更改你可以使用 `X` 按钮。最终你的所有的操作都会应用到中间的`Result`代码中。

当所有的冲突都被解决，所有更改都被应用后。我们可以点击 `Apply` 按钮来完成我们最终的合并操作。

当然你也可以点击 `Accept Left` 还是 `Accept Right` 按钮来选择应用当前分支的内容还是其他分支的内容。这个根据你直接的实际需求而定。

>深度学习：查看 IntelliJ IDEA 中的官方介绍：[Resolve Conflicts](https://www.jetbrains.com/help/idea/2019.2/resolving-conflicts.html)

#### 小提示

如果说当你合并冲突的时候，不小心点击了 `Abort` 按钮，不用担心，你仍然可以点击鼠标`右键`，依次选择 `Git` ->
`Resolve Conflicts`选项来解决冲突。 如下所示：

![遗忘解决的冲突.png](https://upload-images.jianshu.io/upload_images/2824145-4f1678905a734451.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 最后

站在巨人的肩膀上，才能看的更远~

- [Git-分支-分支简介](https://git-scm.com/book/zh/v2/Git-分支-分支简介)
