---
title: Git新手教程-向仓库中添加commit(五)
tags:
  - Git
categories:
  - Git相关
date: 2019-10-12 00:14:59
---


### 前言

在该篇文章中，我们终于要来学习如何创建自己的提交(`commit`)，在前面的文章中，我们已经学会使用 `git init` 命令来创建新仓库，使用 `git clone` 命令来复制现有仓库，使用 `git log` 命令来查看现有的提交。以及使用非常重要的 `git status` 命令来查看仓库的状态。本篇文章会在这些知识的基础上添加 `git add` 、 `git commit` 和 `git diff` 。 在具体讲解这三个命令之前，我们先简单的看看这三个命令的作用。

- `git add` 可以让你将文件从工作目录添加到暂存区。
- `git commit` 可以让你将文件从暂存区中取出。并保存在仓库区中，也就是你实际将要提交的地方。
- `git diff` 可以显示文件两个版本之间的差异，它的输出与上篇文章中使用的 `git log -p` 命令的输出完全一样。

### git add 命令的使用

在使用 `git add` 命令之前，我们先回顾一下仓库的创建过程。我们现在自己的喜欢的目录下创建仓库，在下图中我的仓库的地址为`documents/GitTest/GitTestProject`。在接下来的文章中，都会以该仓库作为例子进行讲解。

首先我们先进入该目录，并通过 `git init`创建Git仓库：

{% asset_img git的init命令使用.jpg git的init命令使用 %}

>在没有向仓库提交任何commit时，多次运行`git init`命令是没有关系的，`git init`命令只会多次重新初始化仓库

#### 检查仓库状态！别忘了

我们一定要在运行Git相关命令后，一定要使用 `git status` 命令来检查当前仓库的状态。因为我们不能保证，我们是否遗忘了某些东西。如果你像我一样使用了 `git status`命令，那么你能得到下列输出结果：

```bash
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

#### 开始添加文件

当我们使用 `git status` 检查了仓库确实没有任何文件后，那接下来我们来创建一些文件。这里我分别创建了三个文件，`Git总目录.md`、`Git练习.md`、`JVM系列之总目录.md`，这个时候我们再使用 `git status` 来查看我们仓库的状态，我们能得到下列结果：

```bash
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)

  Git总目录.md
  Git练习.md
  JVM系列之总目录.md

nothing added to commit but untracked files present (use "git add" to track)
```

要将文件提交到暂存区，我们需要使用 `git add` 命令，这里我们将`Git总目录.md`文件添加到暂存区中，使用命令
`git add Git总目录.md`，我们再使用 `git status` 查看我们的仓库状态，我们能得到下列结果：

>还记得`暂存区`吗？暂存区是Git目录下的一个文件，存储的是即将进入下个 commit 内容的信息。可以将暂存区看做准备工作台，Git 将在此区域获取下个 commit。

```bash
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

  new file:   Git总目录.md

Untracked files:
  (use "git add <file>..." to include in what will be committed)

  Git练习.md
  JVM系列之总目录.md
```

这个时候，在命令行中的 `Untracked files` 下，就只有`Git练习.md`与`JVM系列之总目录.md`了，

>细心的小伙伴肯定看到`(use "git rm --cached <file>..." to unstage)`，该命令可以帮助我们将你 `git add` 错误提交的文件，从暂存区中移除，此外，在命令行输出中出现了"unstage"（撤消暂存）字眼。将文件从工作目录移到暂存区叫做`"staging"`（暂存）。如果已移动文件，则叫做`"staged"`（已暂存）。从暂存区将文件移回工作目录将`"unstage"`（撤消暂存）。

#### 使用 git add 添加剩余的文件

当我们已经将 `Git总目录.md` 添加到暂存区中后，我们可能还想将剩下的两个文件 `Git练习.md` 、 `JVM系列之总目录.md` 也添加到暂存区中。当然我们可以一个一个的使用使用 `git add` 命令添加剩余的文件，我们也可以这样:

```bash
git add Git练习.md JVM系列之总目录.md
```

>使用`git add <file1> <file2> … <fileN>`这种方式，我们可以添加多个文件，其中`<file>`代表一个或多个文件。

除了使用上述方法以外，我们还可以使用一个特殊的命令行字符 `.（点）` ，`.（点）` 代表当前目录，可以用来表示所有文件和物理（注意！注意！注意！包括所有嵌套文件和目录）。

```bash
git add Git练习.md JVM系列之总目录.md
#等于
git add .
```

如果你使用 `.（点）` 添加了多余的文件，那么我们可以使用`git rm --cached <file1> <file2> … <fileN>`命令，将多余的文件从暂存区中移除。

### git commit 命令的使用

当我们将上文提到的三个文件都添加到暂存区之后，现在需要将暂存区中的内容提交到仓库中去，也就是使用 `git commit` 命令，当然在运行该命令之前，我们要时刻使用 `git status` 命令查看当前仓库的状态。使用 `git stasus` 查看状态：

```bash
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

 new file:   Git总目录.md
 new file:   Git练习.md
 new file:   JVM系列之总目录.md
```

嗯，美滋滋，所有的文件都在暂存区中了，那现在开始我们的提交吧。在具体提交之前，我们需要注意，如果你在下载 Git后没有设置文本编辑器，那么 Git 会默认会调用系统的默认编辑器，一般情况是 Vi 或者 Vim 。当然我们也可以配置我们自己喜欢的文本编辑器。这里我配置的是 Sublime Text ，配置命令如下所示：

- >VS Code 配置-->[VS Code as Git editor](https://code.visualstudio.com/docs/editor/versioncontrol#_vs-code-as-git-editor)
- >Sublime Text-->[OS X Command Line](https://www.sublimetext.com/docs/3/osx_command_line.html)
- >如果你对默认的编辑器(Vi或者Vim)感兴趣，你有可能需要这篇文章 [Linux vi/vim](https://www.runoob.com/linux/linux-vim.html)

```bash
git config --global core.editor "'/Applications/Sublime Text.app/Contents/SharedSupport/bin/subl' -n -w"
```

如果你像我一样配置了Sublime Text，那么我们会得到下图：

{% asset_img Git_Commit文本编辑.jpg Git_Commit文本编辑 %}

在`第一行`中，就是我们需要输入此次commit的信息，因为这是我们的第一次提交，所以这里我填的是 `Initial commit` ,当然你可以根据你的喜好填写信息。其他被`#`标记的行都是注释信息，都会被忽略。当我们使用 `git commit` 命令后，我们在控制台会得到如下输出：

```bash
[master (root-commit) 18522c6] Initial commit
 3 files changed, 45 insertions(+)
 create mode 100644 Git总目录.md
 create mode 100644 Git练习.md
 create mode 100644 JVM系列之总目录.md
```

>如果你配置了Git文本编辑器，那么会在你输入内容，退出编辑器后，会自动提交commit。

这个时候我们再使用 `git status` 查看我们的仓库状态,输出结果为：

```bash
On branch master
nothing to commit, working tree clean
```

上述表明，所有暂存区中的文件，都提交到Git的仓库区中了。现在我们就将第一个commit提交到仓库中去了。当然有可能你提交的描述信息很简短，那么你可以使用`-m`选项来跳过编辑器。如下所示：

```bash
git commit -m "initial commit"
```

当然通过 `-m` 选项提交的消息只包含标题的，不会包含正文，如果你想怎么知道怎么写一个阅读性良好的commit message，那么你有可能阅读下面这两篇文章；

- [怎么写Git Commit Message](https://www.jianshu.com/p/0117334c75fc)
- [Commit message 和 Change log 编写指南](http://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)

#### 进行第二次commit

现在我们已经进行我们的第一次commit了，那么现在我们修改`Jxm系列之总目录.md`文件，打开该文件，将文件中的语句`- Java类加载器（双亲委派模型）`删掉，并保存。如下操作：

```bash
- Java内存结构及分区
- Java对象的创建、存储及访问
- Java判断对象是否存活
- 垃圾回收算法（GC)
- Jvm中的常见的垃圾回收器
- Java类加载过程
- Java类加载器（双亲委派模型）#---> 删除这行
```

接着我们使用`git status`查看当前我们的仓库状态：

```bash
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

 modified:   Jvm系列之总目录.md

no changes added to commit (use "git add" and/or "git commit -a")
```

从控制台中，我们可以看到我们的文件`Jvm系列之总目录.md`已经被标记为`modifed`了，那现在我们如何将修改的文件提交到Git的仓库区中呢？要将内容提交到Git的仓库区中去，我们需要将文件提交到暂存区中，在之前的命令中将文件提交到暂存区中，我们需要使用命令 `git add` 命令，当然 `git add` 命令不仅只针对`新建`的文件，它仍可以将`修改后`的文件提交到暂存区中。也就是我们只要使用 `git add` 与 `git commit` 命令，我们就能将修改后的文件提交到Git的仓库中去了。

#### 简单总结

在学习了`git add` 与 `git commit` 命令后，我们简单的总结一下这两个命令。

- `git add` 可以不仅可以向暂存区中添加新的文件，同样也能将修改的文件进行暂存。
- `git commit` 会取出暂存区的文件，并保存到仓库中。该命令需要输入commit消息。

### git diff

还有最后一个命令 `git diff` 。这个命令可以帮助我们查看我们一些没有提交的更改，也就是说我们可以看到当前修改的文件与Git仓库之间的差异。还是`Jvm系列之总目录.md`文件为例，这里我们继续删除`- Java类加载过程`，如下图所示：

```bash
- Java内存结构及分区
- Java对象的创建、存储及访问
- Java判断对象是否存活
- 垃圾回收算法（GC)
- Jvm中的常见的垃圾回收器
- Java类加载过程 #---> 删除这行
```

然后我们使用 `git diff`命令查看命令行输出:

{% asset_img git_diff展示.jpg git_diff展示 %}

在上图中，红色表示当前修改的文件删除的行。我相信大家看到这个界面一定会很熟悉，还记的我们之前介绍长裤仓库的历史提交记录中，所将的`git log -p`吗？其实`git log -p`其实就是使用了`git diff`命令。关于上图中，如果大家不理解每行所代表的意思，那么可以查看《查看仓库的历史记录(四)》中`git log -p`中的介绍。

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

又到了我们熟悉的偷懒环节了。现在我们来看看一下 `git add` 与 `git commit` 与 `git diff` 在idea中的使用，

#### git add

如果你的项目已经通过Git管理，那么当你在IDEA中创建新的文件夹时，编译器会如下提示：

{% asset_img ide_add操作展示.jpg ide_add操作展示 %}

通过提示消息，我们其实就能看出，就是提示我们是否将当前新创建的文件添加到Git的暂存区中，如果你选择确定，那么就会将该文件添加到暂存区中。如果你不小心选择了`cancel`,不用担心，你仍然可以使用下列方式来添加文件到暂存区中。通过选择你要添加的文件，点击鼠标`右键`依次选择`Git`--->`add`。就可以将该文件添加到暂存区中。如下图所示：

{% asset_img git_add_延迟展示.jpg git_add_延迟展示 %}

>小提示：在ide是以一种非常直观的颜色来表示当前仓库中的文件状态:
>
> 1. 红色：表示当前文件或目录没有被跟踪。
> 2. 绿色：表示当前文件或目录已经被添加到仓库中了。
> 3. 蓝色：表示被添加到仓库中的文件或目录被修改或移动。
> 4. 橙色：表示被忽略的文件。
> 5. 白色：表示没有任何更改。

#### git commit 使用

当我们将文件添加到暂存区中后，我们可以通过ide顶部的工具栏进行commit操作，记住是顶部哟！具体如下图所示：

{% asset_img git_commit_展示.jpg git_commit_展示 %}

注意：如果你是修改已经跟踪过的文件，那么我们不需要将修改的文件通过 `git add` 命令将其添加到暂存区中，注意！！！！当我们直接使用 IDE 中的 commit 按钮时，默认是执行 `git add` 与 `git commit` 这两个命令的。

#### git diff 使用

同样的 `git diff` 也在顶部，如下图所示：

{% asset_img git_diff_ide.jpg git_diff_ide %}

#### Git使用快捷键

当然除了上述所有的操作，我们还可以使用ide提供的快捷键进行操作，使用 ``Alt+ ` （Windows)`` 或 ``option + ` (Mac)`` 的方式，可以得到以下界面：

{% asset_img idea_快捷键汇总.jpg idea_快捷键汇总 %}

1. 对应我们使用的 `git commit`
2. 对应我们使用的 `git diff`
3. 对应我们使用的 `git add`

### 最后

站在巨人的肩膀上，才能看的更远~

- [视频推荐-->用Git进行版本控制](https://cn.udacity.com/course/version-control-with-Git--ud123)
- [Git官方网站](https://Git-scm.com/book/zh/v2/)
