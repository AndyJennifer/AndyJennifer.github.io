---
title: Git新手教程-向仓库中添加commit(五)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在该篇文章中，我们终于要来学习如何创建自己的提交(`commit`),在前面的文章中，我们已经学会使用`git init`命令来创建新仓库，使用`git clone`命令来复制现有仓库，使用`git log`命令来查看现有的提交。以及使用非常重要的`git status`命令来查看仓库的状态。本篇文章会在这些知识的基础上添加 `git add`、`git commit` 和`git diff`。 在具体讲解这三个命令之前，我们先简单的看看这三个命令的作用。

- `git add`可以让你将文件从工作目录添加到暂存区。
- `git commit`可以让你将文件从暂存区中取出。并保存在仓库区中，也就是你实际将要提交的地方。
- `git diff`可以显示文件两个版本之间的差异，它的输出与上篇文章中使用的 `git log -p`命令的输出完全一样。

### git add 命令的使用

在使用`git add`命令之前，我们先回顾一下仓库的创建过程。我们现在自己的喜欢的目录下创建仓库，在下图中我的仓库的地址为`documents/GitTest/GitTestProject`，在接下来的文章中，都会以该仓库作为例子进行讲解。

![git的init命令使用.jpg](https://upload-images.jianshu.io/upload_images/2824145-bcc127717a62d01f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>在没有向仓库提交任何commit时，多次运行`git init`命令是没有关系的，`git init`命令只会多次重新初始化仓库

#### 检查仓库状态！别忘了

我们一定要在运行git相关命令后，一定要使用`git status`命令来检查当前仓库的状态。因为我们不能保证，我们是否遗忘了某些东西。如果你像我一样使用了`git status`命令，那么你能得到下列输出结果：

```bash
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

#### 开始添加文件

当我们使用`git status`检查了仓库确实没有任何文件后，那接下来我们来创建一些文件。这里我分别创建了三个文件，`Git总目录.md`、`Git练习.md`、`JVM系列之总目.md`，这个时候我们再使用`git status`来查看我们仓库的状态，我们能得到下列结果：

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

要将文件提交到暂存区，我们需要使用`git add`命令，这里我们将`Git总目录.md`文件添加到暂存区中，使用命令
`git add Git总目录.md`，我们再使用`git status`查看我们的仓库状态，我们能得到下列结果：

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

这个时候，在命令行中的`Untracked files`下，就只有`Git练习.md`与`JVM系列之总目录.md`了，

>细心的小伙伴肯定看到`(use "git rm --cached <file>..." to unstage)`，该命令可以帮助我们将你 `git add` 错误提交的文件，从暂存区中移除，此外，在命令行输出中出现了"unstage"（撤消暂存）字眼。将文件从工作目录移到暂存区叫做`"staging"`（暂存）。如果已移动文件，则叫做`"staged"`（已暂存）。从暂存区将文件移回工作目录将`"unstage"`（撤消暂存）。

#### 使用git add 添加剩余的文件

当我们已经将 `Git总目录.md`添加到暂存区中后，我们可能还想将剩下的两个文件`Git练习.md`、`JVM系列之总目.md`也添加到暂存区中。当然我们可以一个一个的使用使用`git add`命令添加剩余的文件，我们也可以这样:

```bash
git add Git练习.md JVM系列之总目.md
```

>使用`git add <file1> <file2> … <fileN>`这种方式，我们可以添加多个文件，其中`<file>`代表一个或多个文件。

除了使用上述方法以外，我们还可以使用一个特殊的命令行字符点`.`,点`.`代表当前目录，可以用来表示所有文件和物理（注意！注意！注意！包括所有嵌套文件和目录）。

```bash
git add Git练习.md JVM系列之总目.md
#等于
git add .
```

如果你使用点`.`添加了多余的文件，那么我们可以使用`git rm --cached <file1> <file2> … <fileN>`命令，将多余的文件从暂存区中移除。

### git commit 命令的使用

当我们将上文提到的三个文件都添加到暂存区之后，现在需要将暂存区中的内容提交到仓库中去，也就是使用`git commit`命令，当然在运行该命令之前，我们要时刻使用`git status`命令查看当前仓库的状态。使用`git stasu`查看状态：

```bash
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

 new file:   Git总目录.md
 new file:   Git练习.md
 new file:   JVM系列之总目录.md
```

嗯，美滋滋，所有的文件都在暂存区中了，那现在开始我们的提交吧。在具体提交之前，我们需要注意，如果你在下载Git后没有设置文本编辑器，那么Git会默认会调用系统的默认编辑器，一般情况是Vi或者Vim。当然我们也可以配置我们自己喜欢的文本编辑器。这里我配置的是Sublime Text，配置命令如下所示：

- >VS Code 配置-->[VS Code as Git editor](https://code.visualstudio.com/docs/editor/versioncontrol#_vs-code-as-git-editor)
- >Sublime Text-->[OS X Command Line](https://www.sublimetext.com/docs/3/osx_command_line.html)
- >如果你对默认的编辑器(Vi或者Vim)感兴趣，你有可能需要这篇文章[Linux vi/vim](https://www.runoob.com/linux/linux-vim.html)

```bash
git config --global core.editor "'/Applications/Sublime Text.app/Contents/SharedSupport/bin/subl' -n -w"
```

如果你像我一样配置了Sublime Text，那么我们会得到下图：

![Git_Commit文本编辑.jpg](https://upload-images.jianshu.io/upload_images/2824145-5359bd76333c2309.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在第一行中，就是我们需要输入此次commit的信息，因为这时我们的第一次提交，所以这里我填的是`Initial commit`,当然你可以根据你的喜好填写信息。其他被`#`标记的行都是注释信息，都会被忽略。昂我们使用`git commit`命令后，我们在控制台会得到如下输出：

```bash
[master (root-commit) 18522c6] Initial commit
 3 files changed, 45 insertions(+)
 create mode 100644 Git总目录.md
 create mode 100644 Git练习.md
 create mode 100644 JVM系列之总目录.md
```

这个时候我们在使用`git status`查看我们的仓库状态,输出结果为：

```bash
On branch master
nothing to commit, working tree clean
```

上述表明，所有暂存区中的文件，都提交到Git的仓库区中了。现在我们就将第一个commit提交到仓库中去了。当然有可能你提交的描述信息很简短，那么你可以使用`-m`选项来跳过编辑器。如下所示：

```bash
git commit -m "initial commit"
```

#### 具有多个作用的 git add

我们修改了文件。git 看到该文件已被修改。到目前为止，一切正常。注意，要提交 commit，待提交的文件必须位于暂存区。要将文件从工作目录移到暂存区，我们应该使用哪个命令？答对了，是 git add！

我们使用 git add 向暂存区添加了新建的文件，同样的，我们也能使用同一命令将修改的文件暂存。

现在使用 git add 命令将文件移到暂存区，并使用 git status 验证文件是否位于暂存区。

### git commit

git commit 小结
git commit 命令会取出暂存区的文件并保存到仓库中。

$ git commit
此命令：

将打开配置中指定的代码编辑器
（请参阅第一节课中的 git 配置流程，了解如何配置编辑器）

在代码编辑器中：

- 必须提供提交说明
- 以 # 开头的行是注释，将不会被记录
- 添加提交说明后保存文件
- 关闭编辑器以进行提交
- 然后使用 git log 检查你刚刚提交的 commit！

### 良好的提交说明

集合自己博客<日志提交规范>和oppo的日志提交规范来写

### Git diff

### 为何需要该命令

你可能会像我一样，在晚上开始构建项目的下个功能，但是在完成之前就去睡觉了。也就是说，当我第二天开始工作的时候，有一些没有提交的更改。这很正常，因为我还没有完成新的功能，但是我不记得自上次 commit 起我到底完成了哪些代码。git status 将告诉我们哪些文件更改了，但是不会显示到底是什么样的更改。

git diff 命令可以用来查找此类信息！
git diff
The git diff 命令可以用来查看已被加入但是尚未提交的更改。

```bash
git diff
```

要查看 git diff 的实际运行效果，我们需要一些未经提交的更改！在 index.html 中，我们重新组织标题的措辞。将标题从"Expedition"改为"Adventure"。保存文件，然后在终端上运行 git diff。

你应该会看到以下结果：

终端显示了 git diff 命令的输出结果。
哇，看起来是不是很熟悉啊？和运行 git log -p 的结果一样！告诉你个秘密，git log -p 其实就是在后台使用了 git diff。所以你实际上已经知道如何阅读 git diff 的输出结果！

如果你不知道每个部分都是什么内容，请参阅上节课中带注解的"git log -p"输出结果。
git diff 小结
总结下，git diff 命令用来查看已经执行但是尚未 commit 的更改：

```bash
git diff
```

此命令会显示：

- 已经修改的文件
- 添加/删除的行所在的位置
- 执行的实际更改

### 最后

站在巨人的肩膀上，才能看的更远~
