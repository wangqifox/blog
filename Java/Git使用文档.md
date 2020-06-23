---
title: Git使用文档
date: 2020/06/20 14:48:00
---

Git是目前广泛使用的版本控制系统，一直以来我对Git的使用都停留在比较表面的层次，一旦涉及到版本回退、代码合并等操作就会遇到困难。本文是对Git日常使用的一个整理，用于加深对git的理解，方便需要时查阅。同时这也是我们小组培训时的材料。

<!-- more -->

## 基本介绍

版本控制系统是代码开发中不可或缺的工具，在git之前就存在多种版本控制工具，最著名的比如CVS、SVN。

Git的开发者是Linux的创建者Linus，最初Linus选择了一个商业的版本控制系统BitKeeper。后来因为种种问题BitKeeper公司停止了对Linux社区的免费使用权。于是Linus花了两周时间自己用C写了一个分布式版本控制系统，这就是Git。

Git与CVS和SVN最大的不同是后者是集中式的版本控制系统，而Git是分布式版本控制系统。

分布式版本控制系统可以没有“中央服务器”，每个人的电脑上都是一个完整的版本库，工作时理论上不需要联网。但是实际使用时通常有一台充当“中央服务器”的电脑，但这个服务器的作用仅仅是用来方便“交换”大家的修改，没有它大家也一样干活，只是交换修改不方便而已。

## 创建仓库

仓库，英文名`repository`，可以简单理解成一个目录，这个目录里面的所有文件都可以被Git管理起来，Git可以跟踪每个文件的修改、删除，如果需要的话可以执行“还原”操作。

### 创建本地仓库

git不依赖网络就可以工作，因此可以直接在本地创建一个仓库来管理仓库中的文件。操作方式如下：

选择一个目录比如`git-demo`，执行`git init`，仓库就这样建好了。

仓库建好之后目录下多了一个`.git`目录，这个目录就是Git用来跟踪管理仓库的。

### 添加远处仓库

有了本地仓库，理论上就可以使用Git来管理我们的代码。不过为了多人协作的方便以及代码的安全，我们需要一台运行git的服务器来同步多人对仓库的修改，以及备份我们的代码。我们可以自己搭建一台git服务器，也可以直接使用现成的git服务器比如`github`。

首先注册`github`。创建`ssh key`并添加到`github`中。接着在`github`中创建一个新的仓库。获得新仓库的地址，比如：`git@github.com:wangqifox/git-demo.git`。

然后在本地仓库下执行`git remote add origin git@github.com:wangqifox/git-demo.git`。这样就把本地仓库和远程仓库关联起来了。此时远程仓库的名称是`origin`，这是Git默认的叫法，用户可以设置成别的名称。

### 从远处仓库克隆

如果已经有了一个远程仓库，那么就不用手动新建一个本地仓库，直接使用`clone`命令克隆远程仓库到本地：

`git clone git@github.com:wangqifox/git-demo.git`

## 仓库管理

### Git的基本概念

理解Git的各种操作，首先要知道Git的3个基本概念：**工作区**、**暂存区**、**本地仓库**。

工作区就是我们能看到的这个目录，对于文件的修改就在工作区中进行，对于工作区中的修改Git是无法**追踪**到的，即Git不会记录这些修改。

前面我们看到，仓库创建的时候生成了一个名为`.git`的隐藏目录，这个目录不属于工作区，它是Git的本地仓库。

`.git`目录中保存了很多东西，其中最重要的就是称为`stage`（或者`index`）的暂存区，还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`。

![git仓库](media/git%E4%BB%93%E5%BA%93.png)

### 使用Git来管理文件

了解了Git的基本概念之后，我们就可以使用Git来正式管理我们的文件。

在刚才的仓库`git-demo`中新建一个文件`1.txt`。此时文件`1.txt`处于工作区中，并没有添加到暂存区中，使用`git status`命令查看状态：

```
$ git status
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	1.txt

nothing added to commit but untracked files present (use "git add" to track)
```

可以看到，此时`1.txt`文件处于`untracked`的状态。

> 一般情况下，Git中的文件有两种状态：
> 未被追踪的文件（untracked）：如果这个文件还未被纳入版本控制系统中，我们称之为“未被追踪的文件”。这就表示版本控制系统不能监视或者追踪它的改动。一般情况下未被追踪的文件会是那些新建的文件，或者是那些没有被纳入版本控制系统中的忽略文件。
> 已追踪的文件（tracked）：所有那些已经被纳入版本控制系统的项目文件我们称之为“已追踪的文件”。Git会监视它们的任何改动，并且你可以提交或放弃对它的修改。

我们需要使用`git add 1.txt`命令将`1.txt`文件的修改添加到暂存区中。执行`git add`命令之后再使用`git status`命令查看状态：

```
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   1.txt
```

`status`命令显示新增了一个`1.txt`文件。`git add`命令实际上就是把要提交的所有修改放到暂存区。

修改放到暂存区之后，就可以使用`git commit`命令一次性把暂存区的所有修改提交到当前分支：

```
$ git commit -m "add 1.txt"
[master (root-commit) 5491bbb] add 1.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 1.txt
```

`-m`参数后面跟的是此时提交的注释。

> 高质量的注释
> 
> 花一点时间写一个好的提交注释是非常值得的，这样可以让开发团队的其他成员非常容易地明白你做这次提交的目的和你的改动（过了一段时间对你自己也有帮助）。
> 
> 针对你的改动写一个简短的注释标题（原则上不要超过50个字符），然后使用一个空行来分隔注释的内容。注释的内容要尽可能的详细并且要能回答以下几个问题：为什么要做这次修改？与上一个版本相比你到底改动了什么？

如果你有一个很长的提交注释，并且注释中包含很多段落，那你就不需要使用`-m`参数了，Git会为你打开一个文本编辑器（具体打开哪个文本编辑器，你可以在`core.editor`设置它），在文本编辑器中，你可以仔细写上此时提交的注释。

> 什么才是一个好的提交
> 一个高质量的手动提交对你的项目和你自己是非常有意义的。什么才是一个好的提交呢？在这里有一些基本的原则：
>
> 提交要仅仅对应一个相关的改动
> 首先，一次提交应该仅仅只对应一个相关的改动。不要把那些互相毫无关联的改动打包在同一次提交中。如果这次提交出现了什么问题，解决和撤销它将是非常困难的。
> 
> 完整的提交
> 千万不要提交那些没有完全完成的改动。如果你想要临时保存一下你当前的工作，例如一个类似于剪贴板(clipboard)的功能，你可以使用Git提供的“Stash”功能。但是一定不要直接提交它。
> 
> 提交前测试
> 当你提交你的改动时，不要理所当然地认为你的改动永远正确。在你提交你的改动到你的仓库前，进行有效的测试是非常重要的。
>
> 高质量的提交注释
> 一次高质量的提交需要一个好注释。
> 
> 最后，你须要养成一个频繁地进行提交的习惯。这样做将自然而然的让你避免一个很庞大的提交，并且使这些提交可以更好只对映一个相关的改动。

使用`git log`命令我们可以看到我们所有提交的日志。

```
$ git log
commit 5491bbb4f82522eb9e4c84bc7862aef7959078a6 (HEAD -> master)
Author: wangqi <wangq2880@163.com>
Date:   Sat Jun 20 16:49:24 2020 +0800

    add 1.txt
```

每个提交都包括如下的元数据（`metadata`）：

- Commit Id：每个提交都拥有一个唯一的ID。在一些集中式的版本控制系统中（比如`svn`）会使用一个依次递加的版本号码，但是因为Git是分布式的版本控制系统，无法确定每个用户提交的先后顺序，因此它采用这种唯一的哈希编码来标识一次提交。在大多数项目中，这个哈希编码的前七位字符就已经能足够代表一个唯一的提交ID了，一般我们都会用这个简短的7位 ID 来代表一个提交。
- Author Name & Email：提交人的姓名和电子邮件
- Date：提交日期
- Commit Message：提交注释

**注意，Git管理的其实不是文件，而是修改。**新增一行是一个修改，删除一行是一个修改，创建一个文件也是一个修改。

我们可以简单做个验证：

1. 在`1.txt`文件中添加一行`11111`
2. 执行`git add 1.txt`将文件添加到暂存区
3. 再在`1.txt`文件中添加一行`22222`

此时再查看状态：

```
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   1.txt

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   1.txt
```

可以看到有两次修改，一次修改被记录在暂存区中，另一次修改并没有被保存到暂存区中。

这时执行`git commit`命令，再查看状态

```
$ git commit -m "modify 1.txt"
[master c10b61b] modify 1.txt
 1 file changed, 1 insertion(+)
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   1.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

发现第二次的修改并没有被提交到版本库中。

### 撤销修改

撤销修改分为两种情况：

一种情况是修改并没有保存到暂存区，查看状态：

```
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   1.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

可以发现，Git已经提示你了，使用`git restore <file>`命令可以丢弃工作区的修改。这个命令让文件回到最近一次`git commit`或`git add`时的状态。

```
$ git restore 1.txt
$ cat 1.txt
111111111
```

可以看到文件已经复原了。

**注意，老版本的git撤销工作区的修改使用`git checkout -- file`命令**

另一种情况是修改已经保存到暂存区了，查看状态：

```
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   1.txt
```

Git提示我们使用`git restore --staged <file>`命令可以把暂存区的修改撤销掉(unstage)：

```
$ git restore --staged 1.txt
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   1.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

执行之后暂存区的修改被放回到工作区。

**注意，老版本的git撤销暂存区的修改使用`git reset HEAD <file>`命令**

### 版本回退

如果我们的修改不止保存到了暂存区，还从暂存区提交了仓库，那么我们需要使用版本回退来撤销这些修改。

每当我们执行`git commit`方法将暂存区的修改提交到仓库，仓库中就保存了这个修改的一个“版本”。一旦我们需要恢复到某一个“版本”，我们就可以执行`git reset`命令回退到那个“版本”。

首先使用`git log`命令查看提交历史，如果嫌输出信息太多，可以加上`--pretty=oneline`参数：

```
$ git log --pretty=oneline
ae6e66aededc8535066d43aafad2694892b7e140 (HEAD -> master) 4
d4af13be66f81d00ff2e4a0f95af138e8983cd6b 3
9aacbec915380c871ccdd8ed42573d93e7affe62 2
c10b61bb85ea5e3002525b1845e99f58a868c98b modify 1.txt
5491bbb4f82522eb9e4c84bc7862aef7959078a6 add 1.txt
```

左边显示的`commit id`（版本号），右边显示的是提交时的注释。

在`Git`中`HEAD`表示当前版本，指向的`commit id`是`ae6e66aededc8535066d43aafad2694892b7e140`。上一个版本表示成`HEAD^`，上上个版本就是`HEAD^^`，如果是往上100个版本写成`HEAD~100`。

所以想要回退到上一个版本我们就可以执行命令：

`git reset --hard HEAD^`

也可以直接指定上个版本的`commit id`：

`git reset --hard d4af13be6`

查看状态，可以看到已经回退到上一个版本了：

```
$ git log --pretty=oneline
d4af13be66f81d00ff2e4a0f95af138e8983cd6b (HEAD -> master) 3
9aacbec915380c871ccdd8ed42573d93e7affe62 2
c10b61bb85ea5e3002525b1845e99f58a868c98b modify 1.txt
5491bbb4f82522eb9e4c84bc7862aef7959078a6 add 1.txt
```

可以看到最新的那个版本在`log`中已经看不到了，想要返回最新的版本怎么办呢：

如果还记得`commit id`，那很简单，执行`git reset --hard ae6e66a`就可以了。

如果不记得`commit id`了，我们还可以执行`git reflog`命令来找到那一次的`commit id`。`git reflog`命令记录了你的每次命令。

回退之后使用以下命令推送到远程仓库：

```
git push origin HEAD --force
```

#### `--hard`、`--soft`、`--mixed`参数的区别：

`--soft`参数表示将本地版本库的头指针全部重置到指定版本，且将这次提交之后的所有变动都移动到暂存区。

假设我们在Git中提交了多个文件，提交记录如下：

```
$ git log --graph --pretty=oneline --abbrev-commit
* df960fc - (HEAD -> master) add 8.txt (4 minutes ago) <wangqi>
* b4eafe5 - add 7.txt (5 minutes ago) <wangqi>
* 1c7e430 - add 6.txt (9 minutes ago) <wangqi>
* 640047c - add 5.txt (14 hours ago) <wangqi>
...
```

此时执行`git reset --soft`:

```
$ git reset --soft 640047c
$ git status
On branch master
Your branch is ahead of 'origin/master' by 4 commits.
  (use "git push" to publish your local commits)

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   6.txt
	new file:   7.txt
	new file:   8.txt
$ git log --graph --pretty=oneline --abbrev-commit
* 640047c - (HEAD -> master) add 5.txt (14 hours ago) <wangqi>
...
```

我们看到执行`git reset --soft`命令后，`HEAD`指针指向了指定的提交，该提交之后的修改都被放到了暂存区中。

`--mixed`参数与`--soft`参数的区别在于`git reset --mixed`命令执行之后，该提交之后的修改不会放在暂存区中：

```
$ git reset --mixed 640047c
$ git status
On branch master
Your branch is ahead of 'origin/master' by 4 commits.
  (use "git push" to publish your local commits)

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	6.txt
	7.txt
	8.txt
```

`--hard`参数不仅仅将本地版本库的头指针全部重置到指定版本，也会重置暂存区，并且会将工作区的代码也回退到这个版本：

```
$ git reset --hard 640047c
HEAD is now at 640047c add 5.txt
$ git status
On branch master
Your branch is ahead of 'origin/master' by 4 commits.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean
```

#### revert命令

`git revert <commit>`命令也能起到回退版本的作用，不用之处在于：

1. `reset`是向前移动指针，`revert `是创建一个提交来覆盖当前的提交，指针向后移动。
2. `revert `仅仅是撤销某次提交，而`reset`会撤销某个提交点之后所有的提交。

```
$ git revert 1c7e430
Removing 6.txt
[master 7da1247] Revert "add 6.txt"
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 6.txt
$ git log --graph --pretty=oneline --abbrev-commit
* 7da1247 (HEAD -> master) Revert "add 6.txt"
* df960fc add 8.txt
* b4eafe5 add 7.txt
* 1c7e430 add 6.txt
* 640047c add 5.txt
```

此时工作区中`6.txt`文件被删除，说明Git撤销了`1c7e430`这一次的提交，而这之后的所有提交并没有被撤销。

### 忽略文件

通常我们在开发过程中都会有一些文件或者目录是不想纳入版本控制系统的，比如Java开发中生成的`.class`、`.jar`文件和`target`目录。

> 哪些文件需要被忽略呢？一个最简单的分辨方法就是，那些在你开发项目过程中自己生成的文件。例如，临时文件，日志和缓存文件等。
> 还有其他的例子，比如那些为编译代码所提供的密码或者个人设置文件。
> 这个链接：[https://github.com/github/gitignore](https://github.com/github/gitignore)可以帮助你更好地了解在不同的项目和开发平台上哪些内容不需要纳入版本控制中去。

如果我们要手动管理这些被忽略的文件，就必须非常小心翼翼，确保每次操作这些文件都不被保存到暂存区进而提交到版本库。好在Git提供了自动管理这些被忽略文件的机制，我们只需要在工作目录中创建一个称为`.gitignore`的文件，并在此文件中加入被忽略的文件和目录：

- 忽略一个特定的文件：给出从项目根目录开始的路径和文件名，例如`path/to/file.ext`
- 忽略项目下所有这个名字的文件：只要给出文件的全名，不要包括任何路径，例如`filename.ext`
- 忽略项目下所有这个类型的文件：例如`*.ext`
- 忽略一个特定目录下的所有文件：例如`path/to/folder/*`

在一个项目开始之前，最好首先定义好`.gitignore`文件。因为一旦某个文件被提交了，即使把它写入到`.gitignore`文件中，这个文件也不会被忽略。

如果我们创建一个文件叫`canIgnore.txt`并将它提交到版本库中：

```
$ touch canIgnore.txt
$ git add canIgnore.txt
$ git commit -m "add canIgnore.txt"
[master 6e759bf] add canIgnore.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 canIgnore.txt
```

然后我们将`canIgnore.txt`添加到`.gitignore`文件中。

此时再修改`canIgnore.txt`文件，Git不会忽略这个修改：

```
$ git status
On branch master
Your branch is ahead of 'origin/master' by 2 commits.
  (use "git push" to publish your local commits)

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   canIgnore.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

此时如果要忽略掉对该文件的修改，我们只能手动将其从Git的版本库中删除，然后提交：

```
$ git rm --cached canIgnore.txt
rm 'canIgnore.txt'
$ git status
On branch master
Your branch is ahead of 'origin/master' by 2 commits.
  (use "git push" to publish your local commits)

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	deleted:    canIgnore.txt
$ git commit -m "delete canIgnore.txt"
[master 1b5811a] delete canIgnore.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 canIgnore.txt
```

这样对`canIgnore.txt`文件的任何修改都会被Git忽略。

## 分支管理

在我们开发实际项目的过程中，如果我们提交没有开发完成的代码，会导致整个项目处于不稳定的状态，甚至别人`pull`了代码后没法开发了。但是如果等代码全部开发完再提交，又存在丢失每天进度的风险。

因此我们需要不同的分支来管理代码。我们可以在不同的分支上开发并提交，直到开发完毕后，再合并到原来的分支上。

### 创建与合并分支

在Git里有一个默认的主分支，即`master`分支。我们在`git log`命令中可以看到`HEAD`指向的就是当前的`master`分支，`master`指向最近的提交。

到目前为止，我们在`master`分支上一共有5次提交。此时`master`分支是一条线，`master`指向最新的提交，`HEAD`指向`master`。这样就能确定当前分支，以及当前分支的提交点。

![](https://www.liaoxuefeng.com/files/attachments/919022325462368/0)

每次提交，`master`分支都会向前移动一步，这样，随着不断提交，`master`分支也会越来越长。

现在我们来创建一个名为`dev`的分支：

```
git switch -c dev
```

`git switch`命令加上`-c`参数表示创建并切换，相当于以下两条命令：

```
git branch dev
git switch dev
```

**注意，在老版本git上使用`git checkout -b <branch>`来创建并切换分支**

使用`git branch`命令查看当前分支，此命令会列出所有分支，当前分支前面会标一个`*`号：

```
$ git branch
* dev
  master
```

此时，Git新建了一个指针叫`dev`，指向`master`相同的提交。`HEAD`指向`dev`，表示当前分支在`dev`上。

![](https://www.liaoxuefeng.com/files/attachments/919022363210080/l)

从现在开始，对工作区的修改和提交就是针对`dev`分支了。修改`1.txt`文件并提交。

当我们新提交一次后，`dev`指针往前移动一步，而`master`指针不变：

![](https://www.liaoxuefeng.com/files/attachments/919022387118368/l)

假如我们在`dev`上的工作完成了，就可以把`dev`合并到`master`上。

首先切换回`master`分支：

```
$ git switch master
Switched to branch 'master'
```

切换回`master`分支后，我们发现刚才对`1.txt`文件的修改不见了，因为刚才的修改是在`dev`分支上完成的，而此时`master`分支此时还指向原来的提交点。

![](https://www.liaoxuefeng.com/files/attachments/919022533080576/0)

现在我们把`dev`分支的修改合并到`master`分支上：

```
$ git merge dev
Updating ae6e66a..000d967
Fast-forward
 1.txt | 1 +
 1 file changed, 1 insertion(+)
```

`git merge`用于合并执行分支到当前分支。注意上面的`Fast-forward`信息，Git告诉我们，这次合并是“快进模式”，也就是直接把`master`指向`dev`的当前提交，所以合并速度非常快。

合并之后`master`和`dev`指向同一个提交点。

![](https://www.liaoxuefeng.com/files/attachments/919022412005504/0)

合并之后，可以删除`dev`分支：

```
$ git branch -d dev
Deleted branch dev (was 000d967).
```

在查看`branch`，就只剩`master`分支了：

```
$ git branch
* master
```

![](https://www.liaoxuefeng.com/files/attachments/919022479428512/0)

如果分支已经提交到远程仓库了，可以使用一下命令删除远程分支：

```
git push origin --delete <branchName>
或者
git push origin :<branchName>
```

### 解决冲突

合并分支往往不会向之前展示的那样顺利。

假设我们有两个分支`feature1`和`master`，这两个分支同时对`1.txt`文件的最后一行进行了修改并提交，于是我们的分支变成了这样：

![](https://www.liaoxuefeng.com/files/attachments/919023000423040/0)

此时我们再尝试将`feature1`合并到`master`分支中：

```
$ git merge feature1
Auto-merging 1.txt
CONFLICT (content): Merge conflict in 1.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Git告诉我们，`1.txt`文件存在冲突，自动合并失败，必须先手动解决冲突之后在提交。`git status`也告诉我们存在冲突：

```
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   1.txt

no changes added to commit (use "git add" and/or "git commit -a")
```

`git status`命令告诉我们为合并的文件`1.txt`，也可以执行`git merge --abort`命令放弃此次合并。

查看`1.txt`的内容：

```
$ cat 1.txt
111111111
222222222
333333333
444444444
<<<<<<< HEAD
yyyyyyyyy
=======
xxxxxxxxx
>>>>>>> feature1
```

Git用`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容。我们解决冲突之后再提交：

```
$ git add 1.txt
$ git commit -m "merge feature1"
[master 4f18501] merge feature1
```

现在`master`分支和`feature1`分支变成了下图所示：

![](https://www.liaoxuefeng.com/files/attachments/919023031831104/0)

用带参数的`git log`也可以看到分支的合并情况：

```
$ git log --graph --pretty=oneline --abbrev-commit
*   4f18501 (HEAD -> master) merge feature1
|\
| * 3d64f86 (feature1) x
* | b0ba2e0 y
|/
* 000d967 5
* ae6e66a 4
* d4af13b 3
* 9aacbec 2
* c10b61b modify 1.txt
* 5491bbb add 1.txt
```

### 禁用`Fast-forward`模式

合并分支时，Git会尽量使用`Fast-forward`模式，合并之后的分支如下图所示：

![](https://www.liaoxuefeng.com/files/attachments/919022412005504/0)

这种模式下，删除分支后会丢失掉分支信息。可以在合并分支时加上`--no-ff`参数来禁用`Fast-forward`模式：

```
$ git switch -c dev
$ vim 1.txt
$ git add 1.txt
$ git commit -m "5"
[dev 1a0dc77] 5
 1 file changed, 1 insertion(+)
$ git switch master
Switched to branch 'master'
$ git merge --no-ff -m "merge with no-ff" dev
Merge made by the 'recursive' strategy.
 1.txt | 1 +
 1 file changed, 1 insertion(+)
```

因为本次合并要创建一个新的`commit`，所以加上`-m`参数，把`commit`描述写进去

不使用`Fast-forward`模式，合并之后的分支就像这样：

![](https://www.liaoxuefeng.com/files/attachments/919023225142304/0)

### git stash && git cherry-pick

假设我们现在正在`dev`分支上开发工作只进行到一半，这时需要对`master`分支上的代码紧急修复一个bug。因为`dev`分支上的代码只开发到一半还没法提交，所以我们需要执行`git stash`命令来将当前的工作区“储藏”起来，等以后恢复现场后继续工作。

```
$ git stash
Saved working directory and index state WIP on dev: 1a0dc77 5
```

这时就可以去修复`master`分支的bug了。

```
$ git switch master
Switched to branch 'master'
$ git switch -c bug
Switched to a new branch 'bug'
$ vim 1.txt
$ git add 1.txt
$ git commit -m "u"
[bug 0580a5b] u
 1 file changed, 1 insertion(+)
```

修复完成后，切换回`master`分支，并完成合并，最后删除`bug`分支：

```
$ git switch master
Switched to branch 'master'
$ git merge --no-ff -m "merged bug" bug
Merge made by the 'recursive' strategy.
 1.txt | 1 +
 1 file changed, 1 insertion(+)
$ git branch -d bug
Deleted branch bug (was 0580a5b).
```

修复完bug，在回到`dev`分支继续开发：

```
$ git switch dev
Switched to branch 'dev'
```

执行`git stash pop`命令恢复刚才保存的工作现场。

```
$ git stash pop
On branch dev
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   1.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (fa285bcc404cfde3cf9f65965937c73d23732f78)
```

我们知道`dev`分支是从早期的`master`分支分出来的，因此这个bug在`dev`分支也存在。那么如何修复`dev`分支上同样的`bug`呢？

我们只需要把刚才修复bug时提交的修改“复制”到`dev`分支就可以了。注意，我们只想复制修复bug时所提交的修改，并不是把整个`master`分支合并到`dev`分支。

Git专门提供了一个`cherry-pick`命令，让我们能复制一个特定的提交到当前分支。

```
$ git cherry-pick 0580a5b
error: Your local changes to the following files would be overwritten by merge:
	1.txt
Please commit your changes or stash them before you merge.
Aborting
fatal: cherry-pick failed
```

注意，如果`0580a5b`这次提交修改的文件和当前`dev`修改了相同的文件，执行`cherry-pick`命令会失败，我们需要先提交`dev`的修改再执行：

```
$ git cherry-pick 0580a5b
Auto-merging 1.txt
CONFLICT (content): Merge conflict in 1.txt
error: could not apply 0580a5b... u
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
```

此时执行`cherry-pick`命令会合并两次提交，如果有冲突我们还需要手动解决冲突。

### 远程仓库操作

前面介绍仓库创建的时候，介绍了远程仓库的添加与克隆。

在本地仓库添加了远程仓库之后，可以使用`git remote -v`命令查看远程仓库的情况：

```
$ git remote -v
origin	git@github.com:wangqifox/git-demo.git (fetch)
origin	git@github.com:wangqifox/git-demo.git (push)
```

可以看到，远程仓库`origin`的地址是：[git@github.com:wangqifox/git-demo.git](git@github.com:wangqifox/git-demo.git)。如果没有推送权限，就看不到`push`的地址。 

添加远程仓库之后，可以使用`git push -u origin master`命令将本地仓库的所有内容推送到远程仓库中。加上`-u`参数，Git不但会把本地的`master`分支的内容推送到远程仓库的`master`分支，还会把本地的`master`分支和远程的`master`分支关联起来，在以后的推送或者拉取时就可以简化命令。

再执行`git push -u origin dev`推送`dev`分支的修改到远程仓库。

以后如果要在推送提交到远程仓库就可以简单执行：`git push`，不用再使用完整的命令：`git push origin master`。

此时如果有另外的小伙伴`clone`我们的远程仓库，他只能看到`master`分支，看不到本地的`dev`分支。此时可以使用`git branch -a`查看所有的分支：

```
$ git branch -a
* master
  remotes/origin/HEAD -> origin/master
  remotes/origin/dev
  remotes/origin/master
```

可以看到远程的`dev`分支，但是本地并没有该分支。我们需要手动创建该分支：

```
$ git switch -c dev origin/dev
Branch 'dev' set up to track remote branch 'dev' from 'origin'.
Switched to a new branch 'dev'
```

接着使用`git pull`命令拉取远程分支最新的提交，如果执行失败：

```
$ git pull
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
See git-pull(1) for details.

    git pull <remote> <branch>

If you wish to set tracking information for this branch you can do so with:

    git branch --set-upstream-to=origin/<branch> dev
```

原因是没有指定本地`dev`分支与远程`origin/dev`分支的链接。根据提示，设置dev和origin/dev的链接：

```
$ git branch --set-upstream-to=origin/dev dev
```

### rebase操作

`rebase`用于代替`merge`操作。**`rebase`操作相对于`merge`操作更加复杂，使用时需要慎重。**

对于合并操作，Git会尽量使用`Fast-forward`模式，这样合并之后两个分支拥有完全相同的历史。

不过对于两个分支都有新的提交，或者手动禁用`Fast-forward`模式的情况下，Git会创建一个新的提交点，这个提交点整合了两个分支的修改。

这样一来，提交历史就有了多个分叉。有人并不喜欢这种分叉，相反，他们希望项目拥有一个单一的历史发展轨迹。比如一条直线。在历史纪录上没有迹象表明在某些时间它被分成过多个分支。

这就需要使用`rebase`命令。比如我们要在`master`分支上合并`dev`分支的提交，执行：

```
git rebase dev
```

首先，Git会“撤销”所有分支`master`上的那些在与分支`dev`的共同提交之后发生的提交。当然，Git 不会真的放弃这些提交，其实你可以把这些撤销的提交想像成“被暂时地存储”到另外的一个地方去了。

接下来它会整合那些在分支`dev`（这个我们想要整合的分支）上的还未整合的提交到分支`master`中。在这个时间点，这两个分支看起来会是一模一样的。

最后，那些在分支`master`的新的提交（也就是第一步中自动撤销掉的那些提交）会被重新应用到这个分支上，但是在不同的位置上，在那些从分支`dev`被整合过来的提交之后，它们就被`rebased`了。

整个项目开发轨迹看起来就像发生在一条直线上。

`rebase`操作最大的陷阱是它会改写历史记录。

前面我们看到最后一步分支`master`的提交会被重新添加到`dev`被整合过来的提交之后，这些`master`分支的提交虽然内容和原本的一样，但实际上是不同的提交。

如果还仅仅只是操作那些尚未发布的提交，重写历史记录本身也没有什么很大的问题。但是如果你重写了已经发布到公共服务器上的提交历史，这样做就非常危险了。其他的开发者可能已经上原始提交的基础上开始工作了，此时通过`rebase`操作删除了这个原始提交将是非常可怕的。

因此你应该只使用`rebase`来清理你的本地工作，千万不要尝试着对那些已经被发布的提交进行这个操作。

## 标签管理

标签(`tag`)其实就是对某一个提交赋予一个有意义的名称。原因是`commit id`是一串无意义的字符串，非常不好记。

通常在发布一个版本时，我们会对那个时刻的版本打一个标签。这样在将来要取出某个版本时，只需要将对应标签的版本取出来就可以了。

对当前版本打标签非常简单，切换到需要打标签的分支上，然后执行`git tag <name>`就可以了：

```
$ git switch master
$ git tag v1.0
```

默认标签是打在最新提交的`commit`上的。如果需要对历史提交打标签，只需要找到对应的`commit id`，然后执行：

```
$ git tag v0.9 edf9717
```

使用`git tag`查看标签：

```
$ git tag
v0.9
v1.0
```

注意，标签不是按时间顺序列出，而是按字母排序的。可以使用`git show <tagname>`查看标签信息：

```
$ git show v0.9
commit edf97174338f7ffdbaabb87e6d0848cb9359f85e (tag: v0.9)
Author: wangqi <wangq2880@163.com>
Date:   Sun Jun 21 11:03:52 2020 +0800

    t

diff --git a/1.txt b/1.txt
index aafb674..b02fbd5 100644
--- a/1.txt
+++ b/1.txt
@@ -6,3 +6,4 @@ yyyyyyyyy
 xxxxxxxxx
 555555555
 uuuuuuuuu
+ttttttttt
```

还可以创建带有说明的标签，用`-a`指定标签名，`-m`指定说明文字：

```
$ git tag -a v0.1 -m "version 0.1 released" 000d967
```

如果标签打错了，也可以删除：

```
$ git tag -d v0.1
Deleted tag 'v0.1' (was 425da48)
```

因为创建的标签都只存储在本地，不会自动推送到远程。所以，打错的标签可以在本地安全删除。

如果要推送某个标签到远程，使用命令`git push origin <tagname>`：

```
$ git push origin v1.0
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:wangqifox/git-demo.git
 * [new tag]         v1.0 -> v1.0
```

或者，一次性推送全部尚未推送到远程的本地标签：

```
$ git push origin --tags
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:wangqifox/git-demo.git
 * [new tag]         v0.9 -> v0.9
```

如果标签已经推送到远程，要删除远程标签就麻烦一点，先从本地删除：

```
$ git tag -d v0.9
Deleted tag 'v0.9' (was edf9717)
```

然后，从远程删除。删除命令也是push，但是格式如下：

```
git push origin :refs/tags/v0.9
To github.com:wangqifox/git-demo.git
 - [deleted]         v0.9
```

或者

```
git push origin --delete tag <tagName>
```












> https://www.liaoxuefeng.com/wiki/896043488029600
> https://juejin.im/post/5eeac089e51d457421362edf
> https://www.git-tower.com/learn/git/ebook/cn/command-line/advanced-topics/rebase
> https://www.git-tower.com/learn/git/ebook/cn/command-line/basics/starting-with-an-unversioned-project#start
> https://www.jianshu.com/p/952d83fc5bc8