---
layout:     post
title:      Git 常用指令
subtitle:
date:       2023-1-4
author:     YC-Xiang
header-img: img/homepage.jpg
catalog: true
tags:
    - git
---

# GIT

## Git push

```bash
# git push [远程主机名][本地分支名]:[远程分支名]
$ git push origin HEAD:master
# HEAD是当前指向的分支,可以用git show HEAD 查看
$ git show HEAD
$ git branch -a
# 需要code view 时要加/refs/for
$ git push origin HEAD:refs/for/master

# 不加远程分支名
$ git push origin dev
# 相当于,如果远程分支不存在则会自动创建, 并创建联系
$ git push origin dev:dev
$ git branch --set-upstream-to=origin/dev
# 删除远程分支，直接推送空分支到远程分支
$ git push origin :dev
```

## git diff

```bash
# 查看unstaged的改动(还没git add)
$ git diff
# 查看staged的改动(git add过的)
$ git diff --staged
# 显示出branch1和branch2中差异的部分
$ git diff branch1 branch2 --stat
# 显示指定文件的详细差异
$ git diff branch1 branch2 具体文件路径
# 显示出所有有差异的文件的详细差异
$ git diff branch1 branch2
# 显示本地master分支与远程master分支的区别
$ git diff master origin/master
# 比较两个commit
git diff commit id1 commit id2
```
## git log
```bash
# 查看前两个commit的修改
$ git log -p -2
# 查看改动了哪些文件
$ git log --stat
# 每个commit一行展示
$ git log --pretty=oneline
# 展示合并历史
$ git log --graph
# 从一定时间开始的commit
$ git log --since=2.weeks
$ git log --since=2008-01-15
$ git log --since=2 years 1 day 3 minutes ago
# 查看哪些commit修改了function_name
$ git log -S function_name
# 查看某个文件的commit
$ git log -- path/to/file
```
You can also filter the list to commits that match some search criteria. The `--author` option allows
you to filter on a specific author, and the `--grep` option lets you search for keywords in the commit
messages.
![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/RISC-V%E4%B8%AD%E6%96%87%E6%89%8B%E5%86%8C/20230116143526.png)

## git rebase, git merge

git merge：在dev分支git merge main，// 将main最新的commit拉到dev，有合并记录

git rebase：在dev分支git rebase main //修改dev分支，将main最新的commit拉到dev，将dev最新的commit 接到main后面

e.g 在本地一个分支上有了C5，C6两个自己的commit，但此时远程master分支上别人又合并了两个C3,C4分支。如果用git pull（git fetch + git merge）会有一个新的merge commit。此时需要git rebase 将c5,c6接到最新的master代码(c4)上。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/git1.png)

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/git2.png)

## git revert

git branch -f dev HEAD^  //让dev分支指向HEAD^

本地撤销当前提交：git reset HEAD^

远程撤销当前提交：git revert HEAD

git remote show origin：查看远程信息

git branch --set-upstream-to=origin/develop（远程分支） develop：关联远程分支

那么如何查看已经配置分支关联信息呢，通过下述三条命令均可：

1. git branch -vv
2. git remote show origin
3. cat .git/config

在一个分支上修改，突然要切到另一个分支：

1. 把现在的修改 git commit
2. git stash 暂存起来，注意这个stash 也会带到另一个分支。注意git stash pop和apply的区别，apply不会将栈弹出

## git stash

git stash save "add style to our site” 添加stash信息

git stash clear :注意这是清空你所有的内容

git stash drop stash@{0} 这是删除第一个队列

git stash apply 不会删除内容

git stash pop 删除内容

git stash 不能stash untracked的文件，需要先git add，或者git stash -u

查看某个stash的具体内容：git stash show -p stash@{1}

## git 放弃修改, 放弃增加文件操作

1.本地修改了一些文件 (并没有使用 git add 到暂存区)，想放弃修改:

- 单个文件: `git checkout -- filename`
- 所有文件/文件夹: `git checkout .`

2.本地新增了一些文件 (并没有 git add 到暂存区)，想放弃修改:

- 单个文件/文件夹: `rm  -rf filename`
- 所有文件: `git clean -nxfd`

> -f 删除untracked files <br/>
> -d 连untracked 的目录一起删掉 <br/>
> -x 连 gitignore 的untrack 文件/目录也一起删掉（慎用,一般这个是用来删掉编译出来的.o之类的文件用的）<br/>
> -n 先看看会删掉哪些文件，防止重要文件被误删

3.本地修改/新增了一些文件，已经 git add 到暂存区，想放弃修改:
- 单个文件/文件夹: `git reset HEAD filename`
- 所有文件/文件夹: `git reset HEAD .`

4.本地通过 git add 和 git commit 后，想要撤销此次 commit：

- 撤销 commit, 同时保留该 commit 修改：`git reset commit_id` (撤销之后，你所做的已经 commit 的修改还在工作区)

- 撤销 commit, 同时本地删除该 commit 修改：`git reset --hard commit_id` (撤销之后，你所做的已经 commit 的修改将会清除，仍在工作区/暂存区的代码也将会清除)

>这里的commit id可以通过git log查看选取前6位，commit_id是想要回到的节点


## git rebase
> 不要通过rebase对任何已经提交到公共仓库中的commit进行修改（你自己一个人玩的分支除外）

### 合并多个commit为一个完整commit
[https://www.jianshu.com/p/4a8f4af4e803](https://www.jianshu.com/p/4a8f4af4e803)

`git rebase -i HEAD~3` 修改HEAD往后三个分支（包括HEAD)

或者`git rebase -i 某个commit` 修改某个commit前的所有提交

然后`git push -f`可以修改远程的commit记录

### 将某一段commit粘贴到另一个分支上

## 生成/打patch

生成patch：git diff > patch.diff

检查patch：git apply --stat patch.diff

查看能否打入：git apply --check patch.diff

[https://adtxl.com/index.php/archives/471.html](https://adtxl.com/index.php/archives/471.html)

[https://blog.csdn.net/u013318019/article/details/114860407](https://blog.csdn.net/u013318019/article/details/114860407)

## git tag

[https://www.runoob.com/git/git-tag.html](https://www.runoob.com/git/git-tag.html)

```bash
# 查看所有标签
$ git tag

# 创建
$ git tag -a v1.0

# 删除
$ git tag -d v1.0

#查看
$ git show v1.0

# 切换
git checkout v1.0
```
tag 需要单独上传`git push origin <tagname>` 和删除``git push origin --delete <tagname>`

## 创建分支并与远程某分支关联：
```bash
# 可以先更新远程分支信息
git remote update origin --prune

# git checkout -b 本地新分支 远程分支。远程分支可以git branch -a查看
git checkout -b test origin/test

# 用下面的操作两个分支会对不上，无法git push，因为test分支可能是从master分支新建过来的，git log都不对应，需要一个干净的分支
git chekcout -b test
git branch --set-upstream-to=origin/develop
```

# Pro Git

## First-Time Git Setup

1. `[path]/etc/gitconfig` system全局配置。pass the option `--system` to `git config`。
2. `~/.gitconfig` or `~/.config/git/config` user全局配置。`--global`。
3. `config` file in the Git directory (that is, `.git/config`) 某个库的本地配置。`--local`。

You can view all of your settings and where they are coming from using:
```bash
git config --list --show-origin
```

### Your Identity

```bash
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```
### Your Editor

```bash
git config --global core.editor vim
```
# MISC
在repository中不小心上传了大文件或者隐私数据：
https://help.github.com/articles/removing-sensitive-data-from-a-repository/

**git alias**: https://git-scm.com/docs/git-config#Documentation/git-config.txt-alias

# References
[An online game to learn Git](https://learngitbranching.js.org/)
[Pro Git](https://git-scm.com/book/en/v2)
