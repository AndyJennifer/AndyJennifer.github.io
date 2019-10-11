---
title: Git新手教程-远程仓库(九)
tags:
  - Git
categories:
  - Git相关
---

### 前言

在前面的文章中，我们一直介绍的在本地Git的仓库相关知识点。而在实际的项目开发中，大多数情况下，我们往往需要和他人进行合作。因此学习如何与他人协作开发项目使我们必须要学习与掌握的知识点。在接下的的文章中，我们将讲解什么是远程仓库，以及如何运用远程仓库。在本文中将介绍如下命令：

- git remote：管理远程仓库。
- git push ：将修改推送到远程仓库上。
- git pull ：将从远程仓库上获取更新，并合并。
- git fetch ：将从远程仓库上获取更新

### 远程仓库

当使用 Git 来管理我们的项目时，这对本地项目来说非常方便，但是如果需要与他人共享这些本地仓库，我们就需要依托其他工具，比如 `GitLab` 、`GitHub`、等其他用于托管版本控制仓库的服务，来创建远程仓库。那什么是远程仓库呢？

>Git 是用于管理仓库的工具，主要在命令行上使用，`Git lab` or `GitHub` 是托管服务，我们通常会在浏览器中与其进行交互。

#### 什么是远程仓库

我们都知道 Git 是一个**分布式**版本控制系统，那么这就意味着，使用 Git 是不存在主仓库的概念的，因此每个开发者都是使用的都是仓库的一个副本，那么也就是说，远程仓库其实是我们本地仓库的`副本`，只是它们位于其他地方。比如位于 `GitLab` 或者 `GitHub` 上，如下所示：

![远程仓库1.jpg](https://upload-images.jianshu.io/upload_images/2824145-d8a2a4fc6492ad5d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>需要注意的是，我们并不限于使用一个远程仓库，我们可以根据我们自己的需求创建多个远程仓库。

当我们创建远程仓库后，我们就可以和他人进行协同开发了，比如你可以将本地仓库的更改推送到远程仓库，然后其他同事可以从远程仓库中拉取更改到他们的本地仓库中。如下所示：

![远程仓库2.jpg](https://upload-images.jianshu.io/upload_images/2824145-09c30c2f3de25d92.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

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

在 GitHub 中也提供了两种 URL 方式，第一种`HTTPS` 方式，第二种 `SSH` 方式。这里我们采用 HTTPS 的方式。同时 GitHub 中有三种方式来处理远程仓库与本地仓库的关联：

>关于 GitHub 提供的 SSH 方式，可以参看 [GitHub-SSH](https://help.github.com/articles/connecting-to-github-with-ssh/)

- 第一种，在本地创建`新`的仓库并与远程仓库关联。
- 第二种，将`已有`的本地仓库中的内容推送到现在的 GitHub 新建的远程仓库中。
- 第三种，从已有的仓库中导入代码，使用这种模式需要我们提供相应的URL。

在上图 GitHub 中已经提供了关联远程仓库的相关指令，大家可以根据实际的需求来选择不同的方式来关联远程仓库。

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

如果我们本地仓库已有如下提交，我们需要将本地仓库中的内容推送到刚才在 GitHub 中创建的远程仓库，我们需要使用命令 `git push` ，使用该命令会将本地仓库中所有的提交都推送到远程仓库中，如下所示：

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
  - 如果你配置 GitHub 使用 SSH 协议，并提供过 SSH 密匙，就不需要执行上一步。如果你对使用 SSH 连接 GitHub 感兴趣，请参阅使用 SSH 连接 GitHub 文档([GitHub-SSH](https://help.github.com/articles/connecting-to-github-with-ssh/))。
  - 如果你要输入用户名和密码，用户名会在输入后显示出来，但密码不会显示。只需继续输入密码，完成后按 Enter 键即可。
- 如果你的密码出错，不用担心，它会让你重新输入
- Git 会压缩文件使之变小，然后将其推送至远程仓库
- 这里创建了一个新分支，在页面底部可看到[new branch]，后面是 master -> master

当我们将本地仓库中的内容推送到 GitHub 中的远程仓库后，我们查看 GitHub 中我们之前创建的项目：

![推送到远程仓库3.png](https://upload-images.jianshu.io/upload_images/2824145-84c2962847110504.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在 GitHub 中显示了仓库中有三个提交以及一个分支和一个贡献者。

在推送到远程仓库之后，我们来查看我们的本地仓库发生了什么变化吧~ 使用如下命令：

```bash
git log --oneline
```

>因为本地仓库没有其他分支，所以我这里直接使用 `git log --oneline` ，如果你的项目有其他分支，那么可以使用 `git log --oneline --graph --all`。

在我们的本地仓库中使用了如下命令后，我们能得到下图：

![推送到远程仓库4.png](https://upload-images.jianshu.io/upload_images/2824145-8818705c89b2f33d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上述红框中，`origin/master` 其实是一个跟踪分支（跟踪分支是与远程分支有直接关系的本地分支），该分支告诉我们，当前本地仓库关联了一个远程仓库，该远程仓库的别名为 `origin`，同时该远程仓库有一个 `master` 分支。并且该仓库的 `master` 分支指向 commit `f4d7e04` ，也就是说远程仓库拥有并包含所有 `f4d7e04` 下的 commit 。

>需要注意的一点是，这个 `origin/master` 跟踪分支并不能实时表现被跟踪分支在远程仓库上的位置。如果我们之外的其他人对远程仓库做了更改，我们本地仓库中的 `origin/master` 跟踪分支不会移动。我们必须告诉它检查更新，它才会移动。

### 从远程仓库中拉取修改

我们已经学会了如何将本地仓库中的 commit 推送到远程仓库，现在我们试想另一种情况，远程仓库中存在一些 commit ，但是我们的本地仓库中没有这些 commit ，这种情况出现的原因有多个，比如我们是团队协作开发一个项目，有一名同事将更新推送到了远程仓库，或者你在不同的电脑上开展同一个项目，比如你在公司的电脑上向远程仓库推送了更新，但是你的个人电脑中的本地仓库没有这些更新。

#### git pull

如何将远程仓库中的更新拉取到本地仓库呢？我们看下面的这个例子：

![从远程仓库中拉取修改1.jpg](https://upload-images.jianshu.io/upload_images/2824145-3edccd80e572a382.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在我们的本地仓库中有五个提交，但是远程仓库中有六个提交，本地的 `master` 指向提交 `H` ，而远程仓库中的 `master` 指向 `H` 之后的提交 `J`, 由于两个仓库处于不同步的状态，我们需要将远程仓库中的更改同步到本地仓库中。这个时候我们可以使用 `git pull + 远程仓库别名 + 拉取分支`，在本示例中我们可以使用如下命令：

```bash
git pull origin master
```

![从远程仓库中拉取修改2.jpg](https://upload-images.jianshu.io/upload_images/2824145-0a0427ffa735a2ee.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用该命令后，会将远程仓库中的更改拉取到本地仓库，并与本地仓库进行`合并(merge)`，并更新。

#### 实际操作例子

我们已经知道 `git pull` 命令的使用方式，现在结合我们之前的项目我们来实战一下，找到我们之前在 GitHub 中创建的项目，点击 `Add a README`，我们创建一个 `README.md` 文件，如下所示：

![从远程仓库拉取修改3.png](https://upload-images.jianshu.io/upload_images/2824145-995f3fde020829d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择会跳转到一个新的编辑界面，这里我们不对该文件进行编辑，我们直接填写如下界面：

![从远程仓库拉取修改4.png](https://upload-images.jianshu.io/upload_images/2824145-da930d0bccf39088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，我们可以选择填写相关信息。这里我就不填写了，直接点击`Commit new file`。当点击后，我们再查看项目，我们会发现多了一个 `README.md`文件与一个 `commit` 如下所示：

![从远程仓库拉取修改5.png](https://upload-images.jianshu.io/upload_images/2824145-dc1ea710aca34945.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为我们之前本地仓库中只有 `3` 个提交，这个时候与远程仓库不同步，那么现在我们可以使用命令：

```bash
git pull origin master
```

使用该命令后，我们能得到下图：

![从远程仓库拉取修改6.png](https://upload-images.jianshu.io/upload_images/2824145-eea485413d5c0587.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过上述操作，就能将远程仓库中的内容拉取到本地仓库了。但是我们需要注意，使用 `git pull` 命令会`自动`将本地分支与跟踪分支进行`合并(merge)`，如果你不需要，可以使用另一个命令 `git fetch`。

### git fetch

`git fetch` 主要用于从远程仓库中拉取 commit ， 但是`不会`在拉取到 commit 后自动的将本地分支与远程的分支进行合并（merge)。什么意思呢？先不急，我们先看 `git fetch` 命令的使用方式， `git fetch` 与 `git pull` 使用方式基本相同，也是 `远程仓库别名 + 拉取分支`，如下所示：

```bash
git fetch origin master
```

了解了该命令的使用方式后，我们来看看使用 `git fetch` 与 `git pull` 命令的区别:

>在下图中，我们本地仓库只有 `D、E、F、G、H` 五个提交，而远程仓库拥有本地仓库中没有的提交 `J` 。

![从远程仓库中拉取修改7.jpg](https://upload-images.jianshu.io/upload_images/2824145-6cfcd4848cd922e0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上图，大家应该会发现几点

- `git fetch` 与 `git pull` 命令都会将远程分支上的 commit 拉取到本地仓库。
- `git fetch` 与 `git pull` 命令都会将本地跟踪分支（上图中 `origin/master`)将会移动到最新的 commit 。
- `git fetch` 不会将本地分支（上图中 `master`)指向拉取的最新 commit（上图中 `J` commit) 。

也就是说使用 `git fetch` 命令并没有使本地分支进行移动，如果我们希望本地 `master` 具有 `origin/master` 分支上的 commit。则我们需要合并(merge)，假如我们已经在本地仓库的 `master`分支上，我们需要使用命令，

```bash
git merge origin/master
```

简单的来讲，`git fetch` 相当于 `git pull` 的一半的操作，剩下的一半是合并(merge)。

#### git fetch 主要使用场景

`git fetch` 的主要使用场景是当你的远程仓库与本地仓库都有对方没有的 commit 时。请看如下场景：

![从远程仓库中拉取修改8.jpg](https://upload-images.jianshu.io/upload_images/2824145-0c04f73d27f8099d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，本地仓库与远程仓库在 `H` 提交之前状态一直是同步的，但是后续的过程中，本地仓库增加了一个提交 `8`，远程仓库也增加了一个提交 `J` ，如果这个时候，你想从远程仓库中拉取更改到本地，你可能会想使用命令 `git pull origin master` ，但是你会发现并没有任何作用，在如上情况下，我们需要使用 `git fetch origin master` ，使用该命令后，会使跟踪分支（origin/master) 指向最新的 commit ，如下所示：

![从远程仓库中拉取修改9.jpg](https://upload-images.jianshu.io/upload_images/2824145-ffb46a7108e7a938.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果我们想将跟踪分支（origin/master) 的 commit 应用到本地 `master` 分支上 ，我们需要在 `master` 分支上使用命令 `git merge origin/master` ， 那接下来我们能得到下图：

![从远程仓库中拉取修改10.jpg](https://upload-images.jianshu.io/upload_images/2824145-e095389ed957257b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因为合并(merge)后会产生一个合并提交 `4`，又因为远程仓库中中并没有本地仓库中的 `8`，所以如果我们需要远程仓库也拥有这些提交，那么我们可以使用命令 `git push origin master`，使用该命令后，我们能得到下图：

![从远程仓库中拉取修改11.jpg](https://upload-images.jianshu.io/upload_images/2824145-90987540e096afd9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，使用 `git push origin master`命令后，远程仓库会拉取本地仓库的更改，同时会将本地的 `跟踪分支（origin/master）`指向本地最新的提交 `4` 。

### IntelliJ IDEA or Android Sutdio 图形化界面的使用

在上述文章中，我们讲解了相关指令，现在我们来看看相关指令在 IDEA 中的使用。

#### 创建 GitHub 远程仓库

创建远程仓库有多种方式，这里我们还是以在 GitHub 中创建远程仓库为例， 要想在 IDEA 中创建 GitHub 中的远程仓库，我们需要在配置界面（在 Mac 中为 Preferences，Windows 为 Settings )中找到 `Version Control` -> `GitHub`选项 ，然后添加自己的 GitHub账号，如下所示：

![idea_1.png](https://upload-images.jianshu.io/upload_images/2824145-fc55198c22f105c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在添加完毕账号后，找到工具栏中的 `VCS` -> `Import into Version Control` -> `Share project on Github`。 如下所示：

![idea_2.png](https://upload-images.jianshu.io/upload_images/2824145-a03ad38d661de9fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择后会让我们填写，远程仓库的名称，及远程仓库的别名，及对远程仓库的描述信息等，如下图所示：

![idea_3.png](https://upload-images.jianshu.io/upload_images/2824145-5f774242464c107b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

填写相关信息后，我们就可以点击 `Share` 按钮 创建远程仓库啦。

#### pull push  fetch

当我们创建远程仓库后，我们可以通过点击鼠标`右键`，依次选择 `Git` -> `Repositiry`，然后选择使用 `git pull` 、`git push` 还是 `git fetch` 。 如下所示：

![idea_4.png](https://upload-images.jianshu.io/upload_images/2824145-cc0b63cf64e23eb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>上图中红色方框所包括的`蓝色左下箭头`为 `git pull`，也就是从远程仓库拉取更新。

当然如果你要使用 `git push` 命令，也可以直接使用快捷键 ``Alt+ ` （Windows)`` 或 ``option + ` (Mac)`` 的方式。如下所示：

![idea_5.png](https://upload-images.jianshu.io/upload_images/2824145-a9f83544803d6ca9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
