---
title: git-flow开发使用姿势
date: 2017-02-04
categories:
  - Tools
tags:
    - git
---
![1](/images/articles/2017-02-04/1.png)

### master分支
最为稳定功能比较完整的随时可发布的代码，即代码开发完成，经过测试，没有明显的`bug`，才能合并到 `master` 中。请注意永远不要在 `maste` 分支上直接开发和提交代码，以确保 `master` 上的代码一直可用

### develop分支
用作平时开发的主分支，并一直存在，永远是功能最新最全的分支，包含所有要发布 到下一个 `release` 的代码，主要用于合并其他分支，比如 `feature` 分支； 如果修改代码，新建 `feature` 分支修改完再合并到 `develop` 分支。所有的 `feature`、`release` 分支都是从 `develop` 分支上拉的。

### feature分支
这个分支主要是用来开发新的功能，一旦开发完成，通过测试没问题（这个测试，测试新功能没问题），我们合并回`develop` 分支进入下一个 `release` 。

### release分支(预发布分支)
用于发布准备的专门分支。当开发进行到一定程度，或者说快到了既定的发布日，可以发布时，建立一个 `release` 分支并指定版本号(可以在 `finish` 的时候添加)。开发人员可以对 `release` 分支上的代码进行集中测试和修改`bug`。（这个测试，测试新功能与已有的功能是否有冲突，兼容性）全部完成经过测试没有问题后，将 `release` 分支上的代码合并到 `master` 分支和 `develop` 分支。

### hotfix分支(bug修复分支)
用于修复线上代码的`bug`。当需要修复线上`bug`时从 `master` 分支上拉。完成 `hotfix` 后，打上 `tag` 我们合并回 `master` 和 `develop` 分支 这样保证了开发分支和线上分支代码肯定会一致的。


## 使用姿势
初始化分支 以`git-flow`工作流方式(`phpstrom`需要下载`flow`插件)
默认会初始化 `master develop` 两个主干分支。如果已有了不同分支，初始化的时候，可能需要手动指定 `maste` 分支 跟 `develop` 分支。(`develop`命名可稍有差异)

### 开发分支feature
1.分支的名称都是以 `feature/*-20180102` 打头，不需要做修改  如`feature/register-20180102` 即为用户注册的功能分支
2.基于`develop`分支，可以有多个功能分支进行开发
3.`feature`分支做完后，必须合并回`develop` 分支，合并完分支后一般会删除这个 `feature` 分支（也就是 `finish` 一般由测试进行，或者经过测试允许），也可以视情况保留

### 发布分支release
1.分支名称以 `release/*-2080102` 为例
2.`release`分支基于`develop`创建； 当完成一阶段的功能开发需要上线 这时创建基于`develop`的发布分支`release`，一旦创建了`release`分支，不能再从 `develop` 分支合并新的改动到 `release` 分支 但不会影响功能分支合并到`develop`,可以基于`release`分支进行测试和`bug`修改，测试不用在另外创建用于测试的分支
3.`release` 发布的时候，合并到 `master` 和 `develop` 分支，同时打`tag`，视情况删除`release`分支，通常应该删除掉
维护修复分支(`hotfix`、`bugfix`命名可不同)
1.分支名称以 `hotfix/*` 为例 如`hotfix/homepage` 即完成首页`bug`修复
2.`hotfix` 分支基于 `master` 分支创建，开发完毕后合并到 `master` 和 `develop` 分支，同时创建 `tag`(创建`tag`视团队情况而定)
3.这是唯一可以直接从 `master` 分支 `fork`出来的分支。
开发功能时基于开发分支的规范 `bug`修复时基于维护分支的规范即可 主干分支提交时 拉取后再提交

## 命令行使用
### 创建develop分支
git branch develop
git push -u origin develop

### 创建开发新的feature
```shell
git checkout -b 分支名称 develop
# 推送功能分支到远程(可选)
git push -u origin 分支名称
```

### 功能feature开发完成
```shell
# 先拉取远程develop分支
git pull origin develop
git checkout develop
git merge —-no-ff 分支名称 //合并开发的功能分支到develop
git push origin develop
git branch -d 分支名称//删除本地的功能分支
git push origin —-delete 远程分支名称//删除远程的开发的功能分支 通常名称一样  如没提交则不删
```


### 创建release分支
```shell
git checkout -b 分支名称 develop //从develop创建release分支(预发布分支)
```

### 完成release分支
```shell
// 合并release到master分支
git checkout master
git merge —-no-ff 分支名称
git push
//合并release分支到develop(因为可能会在release做一些测试和上线前的修复)
git checkout develop
git merge —-no-ff 分支名称
git push
// 删除release分支 (可暂时保留 由于打上tag 一般都会做删除)
git branch -d 分支名称
git branch origin —-delete 远程分支名称//删除远程的开发的功能分支 通常名称一样  如没提交则不删
git tag -a tag tag名称 master
git push tags
```

### 创建hotfix分支
```shell
git checkout -b 分支名称 master
```

### 完成hotfix分支
```shell
git checkout master
git merge —-no-ff 分支名称
git push
git checkout develop
git merge —-no-ff 分支名称
git push
git branch -d 分支名称
git tag -a tag tag名称 master
git push tags
```

注意事项
1.所有开发分支从 `develop` 分支拉取新的开发分支。
2.所有 `hotfix` 分支从 `master` 拉新的`bug`修复分支。
3.所有在 `master` 上的提交都必要要有 `tag`，方便回滚。
4.只要有合并到 `master` 分支的操作，都需要和 `develop` 分支合并下，保证同步 如维护分支的合并。
5.`master` 和 `develop` 分支是主要分支，主要分支每种类型只能有一个，辅助分支`release`和`hotfix`分支需要时创建 `release`预发布分支可保留
6.`—-no-ff`合并模式 合并分支删除后 有历史 看得出合并记录 即有`merge commit`记录




