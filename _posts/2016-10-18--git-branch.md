---
layout: post
title: git branch
date: 2016-10-18
category: "git"
---

# 创建分支

___

首先，我们假设你正在你的项目上工作，并且已经有一些提交。

![一个简单的提交历史。](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-branching-1.png)

现在，你已经决定要解决你的公司使用的问题追踪系统中的 #53 问题。 想要新建一个分支并同时切换到那个分支上，你可以运行一个带有 `-b` 参数的 `git checkout` 命令：

```
$ git checkout -b iss53
Switched to a new branch "iss53"
```

它是下面两条命令的简写：

```
$ git branch iss53
$ git checkout iss53
```

![创建一个新分支指针。](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-branching-2.png)

# 合并分支

___

## fast-forward

假设你现在的分支情况如下：

![基于 `master` 分支的紧急问题分支（hotfix branch）。](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-branching-4.png)

你可以将hotfix分支合并到master分支：

```shell
$ git checkout master
$ git merge hotfix
Updating f42c576..3a0874c
Fast-forward
 index.html | 2 ++
 1 file changed, 2 insertions(+)
```

由于当前 `master` 分支所指向的提交是你当前提交（有关 hotfix 的提交）的直接上游，所以 Git 只是简单的将指针向前移动。这种情况下的合并操作没有需要解决的分歧，所以叫做 “快进（fast-forward）”。

现在的分支情况如下：

![`master` 被快进到 `hotfix`。](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-branching-5.png)

## 合并不同的分支

将 `iss53` 分支到 `master` 分支：

```shell
$ git checkout master
Switched to branch 'master'
$ git merge iss53
Merge made by the 'recursive' strategy.
index.html |    1 +
1 file changed, 1 insertion(+)
```

和之前将分支指针向前推进所不同的是，Git 将此次三方合并的结果做了一个新的快照并且自动创建一个新的提交指向它。 这个被称作一次合并提交，它的特别之处在于他有不止一个父提交。

![一个合并提交。](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-merging-2.png)

## 带冲突的分支合并

有时候合并操作不会如此顺利。 如果你在两个不同的分支中，对同一个文件的同一个部分进行了不同的修改，Git 就没法干净的合并它们。 如果你对 #53 问题的修改和有关 `hotfix` 的修改都涉及到同一个文件的同一处，在合并它们的时候就会产生合并冲突：

```shell
$ git merge iss53
Auto-merging index.html
CONFLICT (content): Merge conflict in index.html
Automatic merge failed; fix conflicts and then commit the result.
```

此时 Git 做了合并，但是没有自动地创建一个新的合并提交。 Git 会暂停下来，等待你去解决合并产生的冲突。

 出现冲突的文件会包含一些特殊区段，看起来像下面这个样子：

```
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
 please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```

你解决了所有文件里的冲突之后，对每个文件使用 `git add` 命令来将其标记为冲突已解决。然后通过`git commit`提交。



# 远程分支

___

从一个远程仓库clone到本地时，`clone` 命令会为你自动将远程仓库命名为 `origin`，拉取它的所有数据，创建一个指向它的 `master` 分支的指针，并且在本地将其命名为 `origin/master`。

![克隆之后的服务器与本地仓库。](https://git-scm.com/book/en/v2/book/03-git-branching/images/remote-branches-1.png)

如果你在本地的 `master` 分支做了一些工作，然而在同一时间，其他人推送提交到origin服务器并更新了它的 `master` 分支，只要你不与 origin 服务器连接，你的 `origin/master` 指针就不会移动。

![本地与远程的工作可以分叉。](https://git-scm.com/book/en/v2/book/03-git-branching/images/remote-branches-2.png)

### git fetch

如果要同步你的工作，运行 `git fetch origin` 命令。 这个命令查找 “origin” 是哪一个服务器，从中抓取本地没有的数据，并且更新本地数据库，移动`origin/master` 指针指向新的、更新后的位置。

![`git fetch` 更新你的远程仓库引用。](https://git-scm.com/book/en/v2/book/03-git-branching/images/remote-branches-3.png)

### git push

当你想要公开分享一个分支时，需要将其推送到有写入权限的远程仓库上。 本地的分支并不会自动与远程仓库同步 - 你必须显式地推送想要分享的分支。

如果希望和别人一起在名为 `serverfix` 的分支上工作，你可以 运行`git push (remote) (branch)`:

```shell
$ git push origin serverfix
Counting objects: 24, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (15/15), done.
Writing objects: 100% (24/24), 1.91 KiB | 0 bytes/s, done.
Total 24 (delta 2), reused 0 (delta 0)
To https://github.com/schacon/simplegit
 * [new branch]      serverfix -> serverfix
```

如果并不想让远程仓库上的分支叫做 `serverfix`，可以运行 `git push origin serverfix:awesomebranch` 来将本地的 `serverfix` 分支推送到远程仓库上的`awesomebranch` 分支。

### 跟踪分支

本地分支可以与远程分支对应，这样当使用`git pull`和`git push`时不用指定远程分支。

#### 创建跟踪分支

当克隆一个仓库时，它通常会自动地创建一个跟踪 `origin/master` 的 `master` 分支。

当从远程分支check out时，也会自动创建一个跟踪远程分支的本地分支，`git checkout -b [branch][remotename]/[branch]`。

这是一个十分常用的操作所以 Git 提供了 `--track` 快捷方式：

```shell
$ git checkout --track origin/serverfix
Branch serverfix set up to track remote branch serverfix from origin.
Switched to a new branch 'serverfix'
```

如果想要将本地分支与远程分支设置为不同名字，你可以轻松地增加一个不同名字的本地分支的上一个命令：

```
$ git checkout -b sf origin/serverfix
Branch sf set up to track remote branch serverfix from origin.
Switched to a new branch 'sf'
```

#### 设置跟踪分支

设置已有的本地分支跟踪一个刚刚拉取下来的远程分支，或者想要修改正在跟踪的上游分支，你可以在任意时间使用 `-u` 或 `--set-upstream-to` 选项运行 `git branch` 来显式地设置。

```
$ git branch -u origin/serverfix
Branch serverfix set up to track remote branch serverfix from origin.
```

#### 查看跟踪分支

如果想要查看设置的所有跟踪分支，可以使用 `git branch` 的 `-vv` 选项。

```
$ git branch -vv
  master    1ae2a45 [origin/master] deploying index fix
* serverfix f8674d9 [origin/serverfix: ahead 3, behind 1] add fix
```

需要重点注意的一点是这些数字的值来自于你从每个服务器上最后一次抓取的数据。 这个命令并没有连接服务器，它只会告诉你关于本地缓存的服务器数据。 如果想要统计最新的领先与落后数字，需要在运行此命令前抓取所有的远程仓库。 可以像这样做：`$ git fetch --all; git branch -vv`

### git pull

当 `git fetch` 命令从服务器上抓取本地没有的数据时，它并不会修改工作目录中的内容。 它只会获取数据然后让你自己合并。 然而，有一个命令叫作 `git pull` 在大多数情况下它的含义是一个 `git fetch`紧接着一个 `git merge` 命令。 如果有一个像之前章节中演示的设置好的跟踪分支，不管它是显式地设置还是通过 `clone` 或 `checkout` 命令为你创建的，`git pull` 都会查找当前分支所跟踪的服务器与分支，从服务器上抓取数据然后尝试合并入那个远程分支。

### 删除远程分支

```
$ git push origin --delete serverfix
To https://github.com/schacon/simplegit
 - [deleted]         serverfix
```

