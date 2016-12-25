---
title: itscoder 开发步骤 —— git 篇
date: 2016-12-24 14:50:11
categories: Git
tags: [git ,itscoder]
---
# 引言
给 itscoder 开发步骤做一个笔记，我先抛砖迎玉了。

# 1. 查看分支
  包括上源分支和本地分支，上源分支会有两个，一个是个人 fork 下来的 origin 源，还有一个是主项目 upstream 源，该源主要为了以后新的一期开始时候，进行更新源。如果没有 upstream 源，请跳至 4,5,6 步。
  
```
  $ git branch -a
  * hymane/phase8
    master
    phase8
    remotes/origin/HEAD -> origin/master
    remotes/origin/master
    remotes/upstream/master
    remotes/upstream/phase6
    remotes/upstream/phase7
    remotes/upstream/phase8
```

# 2. 删除本地分支
  新的一期开始后，上一期不需要的分支可以进行删除了。一般我们需要将本地 username/phasex 和 phasex 两个分支删除掉，x 代表上一期。
   
```
  git branch -D [branch]
```

# 3. 删除远程分支
  本地分支删了之后，还需要将远程对应分支删掉，一般需要将 origin/username/phasex 和 origin/phasex 两个分支删掉。如：`git push origin :hymane/phase7`。
   
```
  git push [repoName] :[branch]
```

# 4. 查看远程源
  查看当前仓库的上源库，`-v` 查看详细上源信息，包括是否可以 fetch 和 push。如下，我们有了两个源，一个是个人 github 库，一个是 fork 的原始库地址，我们可以根据这个 upstream 来更新我们的 origin，保持和主源同步。
  
```
  $ git remote -v
  origin  https://github.com/hymanme/weeklyblog.git (fetch)
  origin  https://github.com/hymanme/weeklyblog.git (push)
  upstream        https://github.com/itsCoder/weeklyblog.git (fetch)
  upstream        https://github.com/itsCoder/weeklyblog.git (push)
```

# 5. 添加远程源
  如果上一步查看的结果里面只有一个上源，即只有 origin,我们需要手动添加我们的主源。如：`git remote add upstream https://github.com/itsCoder/weeklyblog.git`，其中 upstream 可以自定义名字。
  
```
  git remote add [repoName] [URL]
```

# 6. 将某远程源合并至本地
  新的一期来临时，本地和 origin源 不需要的分支也删除干净了，接下来就要将自己的仓库和主仓库进行同步操作了。
  
```
    git fetch [repoName]
    git merge [repoName]/[branch]
    or
    git pull [repoName]/[branch]
```

  如

```
   git fetch upstream         //更新
   git merge upstream/master  //合并主分子，本地分支即将同步成最新

   git merge upstream/phase8  //合并最新一期的分支
   git checkout phase8        //切换到最新一期分支
   //按开发规范，在最新一期分支切换一个个人开发分支，username/phasex 分支。
   git branch checkout -b hymane/phase8  
```

# 7. 在本地工作区的 phasex 文件夹下进行博客书写。
  接下来就是修改、add、commit、push。

