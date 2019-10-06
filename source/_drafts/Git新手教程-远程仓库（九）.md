---
title: Git新手教程-远程仓库(九)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们一直介绍的在本地Git的仓库相关知识点。而在实际的项目开发中，大多数情况下，我们往往需要和他人进行合作。因此学习如何与他人协作开发项目使我们必须要学习与掌握的知识点。在接下的的文章中，我们将讲解什么是远程仓库，以及如何运用远程仓库。在本文中将介绍如下命令：

- git remote：管理远程仓库
- git push ：将修改推送到远程仓库上
- git pull ：将从远程仓库上获取更新

### 远程仓库

当使用 Git 来管理我们的项目时，这对本地项目来说非常方便，但是如果需要与他人共享这些本地仓库，我们需要其他工具，比如 `GitLab` 、`GitHub`、等其他用于托管版本控制仓库的服务，来创建远程仓库。那什么是远程仓库呢？

>Git 是用于管理仓库的工具，主要在命令行上使用，`Git lab` or `GitHub` 是托管服务，我们通常会在浏览器中与其进行交互。

#### 什么是远程仓库

我们都知道 Git 是一个**分布式**版本控制系统，那么这就意味着，使用 Git 是不存在主仓库的概念的，因此每个开发者都是使用的都是仓库的一个副本，那么也就是说，远程仓库其实是我们本地仓库的`副本`，只是它们位于其他地方。比如位于 `GitLab` 或者 `GitHub` 上，如下所示：

![远程仓库1.jpg](https://upload-images.jianshu.io/upload_images/2824145-d8a2a4fc6492ad5d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>需要注意的是，我们并不限于使用一个远程仓库，我们可以根据我们自己的需求创建多个远程仓库。

### 添加远程仓库

在上文中，我们说过远程仓库需要依托于其他工具，这里我们将我们的远程仓库放在 GitHub 上进行托管，这里再次强调一下：

- Git 是版本控制工具
- GitHub or GitLab 是托管 Git 项目的服务。

那接下来，我们将在GitHub上创建一个仓库，如果你还没有账户，那么快去[官网](https://github.com)整一个账号吧~~~ 当我们创建好账号并登录后，将位于如下主页：

![GitHub_创建仓库.png](https://upload-images.jianshu.io/upload_images/2824145-bc7b680e920143c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我们点击右侧的 `+` 符号，选择 `New Repostiry` ，那么接下来我们将跳转到创建仓库界面：

![GitHub_创建仓库2.png](https://upload-images.jianshu.io/upload_images/2824145-ab661abdaa709e73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- `Repostiry name` ：这里需要填写仓库的名称（一般情况下，我们都是使用项目名称作为仓库的名称，不用纠结一个完美的名称，仓库的名称可以随时更改）
- `Description` ：这里需要填写仓库的描述，该选项是选填项。
- `Public/Private` ：这里需要填写仓库的模式 ，`Public` 表示公开，意味着任何人都可以查看该仓库中的所有代码，`Private` 表示私有，意味着只有仓库拥有者，才有权查看该仓库中的代码。在 GitHub 中允许我们创建一定数量的私有仓库。
- `Initialize this repository with the README`：表示是否在创建仓库时创建 README 文件，如果勾选，表示创建，反之不创建。

在创建完成仓库后，我们会得到如下界面：

![GitHub_创建仓库3.png](https://upload-images.jianshu.io/upload_images/2824145-d2b3698232dafcaf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在在 GitHub 中也提供了两种 URL 方式，第一种`HTTPS` 方式，第二种 `SSH` 方式。这里我们采用 HTTPS 的方式。同时 GitHub 中有三种方式来处理远程仓库与本地仓库的关联：

- 第一种，在本地创建新的仓库并与远程仓库关联。
- 第二种，将已有的本地仓库中的内容推送到现在的 GitHub 新建的远程仓库中。
- 第三种，从已有的仓库中导入代码，使用这种模式需要我们提供相应的URL。

在上图 GitHub 中已经提供了关联远程仓库的相关指令，大家可以根据实际的需求来选择不同的方式来关联远程仓库。

>关于 GitHub 提供的 SSH 方式，可以参看 [GitHub-SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)

#### 本地仓库如何与远程仓库关联

在上述文章中，我们只介绍了如何创建远程仓库，并没有实际将本地仓库与远程仓库进行关联。那下面我们将本地已有的仓库与远程仓库进行关联。

因为我已经创建了本地仓库 `GitTestProject` ，那么根据之前 GitHub 中给我们的提示，我们需要使用命令：

```bash
git remote add origin https://github.com/AndyJennifer/GitTestProject.git
```

使用上述命令，就可以将本地仓库与远程仓库建立连接了。对于该命令，我们几点需要注意：

- 该命令格式为 `git remote add 远程仓库别名 远程仓库地址`
- `origin` 这个单词只是指代远程仓库，你可以当做它是远程仓库的别名，一般情况下都是使用 `origin` 来指代远程仓库，当然你可以将它修改为其他名称，比如 `haha` ，那么我们需要修改命令为：

 ```bash
 git remote add haha https://github.com/AndyJennifer/GitTestProject.git
 ```

- 远程仓库的地址需要填写完整的URL。

#### git remote 命令介绍

那么接下来，我们就去我们的本地仓库中关联这个远程仓库吧，如下所示：

![git_remote.png](https://upload-images.jianshu.io/upload_images/2824145-c0bc371e1d12a9c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>注意 `git remote add` 命令，需要在你本地的仓库中使用。如果你不小心关联错了远程仓库，可以使用 `git remote remove <name>` 命令将关联关系删除，关于更多命令的使用，可以查看官网中 [git remote](https://git-scm.com/docs/git-remote)中的介绍。

眼尖的小伙伴肯定看见了`红色`方框中的 `git remote`命令，该命令可以查看本地仓库与远程仓库的关联关系，当本地仓库没有与任何远程仓库进行关联时，不会显示任何远程仓库别名。反之，将会显示所有关联的远程仓库别名。

#### git remote -v 命令介绍

当然如果我们想查看远程仓库中的完整路径，我们也可以使用 `git remote -v` 命令，使用该命令后，我们能得到如下输出：

```bash
git remote -v
origin git@github.com:AndyJennifer/GitTestProject.git (fetch)
origin git@github.com:AndyJennifer/GitTestProject.git (push)
```

上述输出表明，当我们在使用 `origin` 别名时，实际使用的路径为:

```text
https://github.com/GoogleChrome/lighthouse.git
```

可能你也发现了，现在有两个远程仓库，都是 `"origin"` 且链接到相同的 URL。唯一的区别在结尾处： (fetch) 部分和 (push) 部分，我们将在接下来的文章中详细说明。

### 将更改推送到远程仓库

如果我们本地仓库已有如下提交，我们需要将本地仓库中的内容推送到刚才在 GitHub 中创建的远程仓库，我们需要使用命令 `git push` ，使用该命令会将本地仓库中所有的提交都推送到远程仓库汇中，如下所示：

![推送到远程仓库.jpg](https://upload-images.jianshu.io/upload_images/2824145-acf5ce7482b9ce4a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在推送到远程仓库之前，我们先看看一下本地仓库的提交：

![本地仓库commit.png](https://upload-images.jianshu.io/upload_images/2824145-428777e2c1895c5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在开始推送吧！要将本地仓库中的 commit 推送到远程仓库，我们需要使用 `git push` 命令，当然不是简单的调用该命令就行了，我们还需要提供`远程仓库别名`，以及容纳所提交的 commit 的`分支名`。如下所示：

```bash
git push <remote name> <branch>
```

这里我们的远程仓库别名为 `origin`，并且我想推送的 commit 位于 `master` 分支上。那么，我要使用以下命令将我的 commit 推送到 GitHub 上的远程仓库：

```bash
git push origin master
```

![推送到远程仓库2.png](https://upload-images.jianshu.io/upload_images/2824145-4db361343eecbbb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


有几点需要注意：

- 你可能需要输入用户名和密码，这取决于你如何配置 GitHub 的以及使用的远程 URL 。
- 如果你使用的是 HTTP 版本（而不是 ssh 版本）的远程仓库，就需要提供用户名和密码。
  - 如果你配置 GitHub 使用 SSH 协议，并提供过 SSH 密匙，就不需要执行上一步。如果你对使用 SSH 连接 GitHub 感兴趣，请参阅使用 SSH 连接 GitHub 文档。
  - 如果你要输入用户名和密码，用户名会在输入后显示出来，但密码不会显示。只需继续输入密码，完成后按 Enter 键即可。
- 如果你的密码出错，不用担心，它会让你重新输入
- Git 会压缩文件使之变小，然后将其推送至远程仓库
- 这里创建了一个新分支，在页面底部可看到[new branch]，后面是 master -> master

![推送到远程仓库3.png](https://upload-images.jianshu.io/upload_images/2824145-84c2962847110504.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在推送到远程仓库之后，我们来查看我们的本地仓库发生了什么变化吧~ 使用如下命令：

```bash
git log --oneline
```

>因为本地仓库没有其他分支，所以我这时直接使用 `git log --oneline` ，如果你的项目有其他分支，那么可以使用 `git log --oneline --graph --all`。

在我们的本地仓库中使用了如下命令后，我们能得到下图：

![推送到远程仓库4.png](https://upload-images.jianshu.io/upload_images/2824145-8818705c89b2f33d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述红框中，`origin/master` 该标记告诉我们，当前本地仓库关联了一个远程仓库，该仓库的别名为 `origin`，同时该仓库有一个 `master` 分支。并且远程仓库的 `master` 分支指向 commit `f4d7e04`,也就是说远程仓库拥有并包含所有 `f4d7e04` 下的 commit 。

### 从远程仓库中拉取修改

### Pull 与 Fetch

### 最后

站在巨人的肩膀上，才能看的更远~
