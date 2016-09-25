## Git 学习笔记

在上一期加入了 **[itsCoder](http://itscoder.com/)** 组织后，由于对 Git 操作不熟悉导致了一些问题，给组织搞了一波事情。所以，今天打算写一篇 Git 学习笔记以便学习和今后回顾。

###  Git 基础学习

Git 是一个开源的分布式版本控制工具，它的开发者是大名鼎鼎的 Linux 操作系统的作者 Linus Torvalds 。优点有**速度快**、**完全分布式**、**允许成千上万个并行开发的分支**、**有能力高效管理类似 Linux 内核一样的超大规模项目**。

学习 Git 最重要的是理解其内部原理，分清楚工作区、暂存区、版本库，还有就是理解 Git 跟踪并管理的是修改，而非文件。下面列举一下 Git 的一些特点

**1. 直接记录快照，而非差异比较**

Git 和其它版本控制系统的主要差别在于 Git 对待数据的方法。 概念上来区分，其它大部分系统以文件变更列表的方式存储信息。Git 更像是把数据看作是对小型文件系统的一组快照。 每次提交更新，或在 Git 中保存项目状态时，它主要对当时的全部文件制作一个快照并保存这个快照的索引。 

**2. 近乎所有操作都是本地执行**

在 Git 中的绝大多数操作都只需要访问本地文件和资源，一般不需要来自网络上其它计算机的信息。比如要浏览项目的历史，Git 不需外连到服务器去获取历史，它只需直接从本地数据库中读取。 如果你想查看当前版本与一个月前的版本之间引入的修改，Git 会查找到一个月前的文件做一次本地的差异计算，而不是由远程服务器处理或从远程服务器拉回旧版本文件再来本地处理。

**3. Git 保证完整性**

Git 中所有数据在存储前都计算校验和，然后以校验和来引用。 这意味着不可能在 Git 不知情时更改任何文件内容或目录内容。 这个功能建构在 Git 底层，是构成 Git 哲学不可或缺的部分。 若你在传送过程中丢失信息或损坏文件，Git 就能发现。Git 数据库中保存的信息都是以文件内容的哈希值来索引，而不是文件名。

**4. Git 一般只添加数据**

你执行的 Git 操作，几乎只往 Git 数据库中增加数据。 很难让 Git 执行任何不可逆操作，或者让它以任何方式清除数据。 同别的 VCS 一样，未提交更新时有可能丢失或弄乱修改的内容；但是一旦你提交快照到 Git 中，就难以再丢失数据，特别是如果你定期的推送数据库到其它仓库的话。

**5. 三种状态：已提交、已修改、已暂存**

已提交表示数据已经安全的保存在本地数据库中。 已修改表示修改了文件，但还没保存到数据库中。 已暂存表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

Git 仓库目录是 Git 用来保存项目的元数据和对象数据库的地方。 这是 Git 中最重要的部分，从其它计算机克隆仓库时，拷贝的就是这里的数据。

工作目录是对项目的某个版本独立提取出来的内容。 这些从 Git 仓库的压缩数据库中提取出来的文件，放在磁盘上供你使用或修改。

暂存区域是一个文件，保存了下次将提交的文件列表信息，一般在 Git 仓库目录中。 有时候也被称作“索引”，不过一般说法还是叫暂存区域。

基本的 Git 工作流程如下：

1. 在工作目录中修改文件。
2. 暂存文件，将文件的快照放入暂存区域。
3. 提交更新，找到暂存区域的文件，将快照永久性存储到 Git 仓库目录。

如果 Git 目录中保存着的特定版本文件，就属于已提交状态。 如果作了修改并已放入暂存区域，就属于已暂存状态。 如果自上次取出后，作了修改但还没有放到暂存区域，就是已修改状态。

### Git 命令行学习

**配置用户信息**

```c
$ git config --global user.name "Your Name"
$ git config --global user.email "Your Email"
```

**检查配置信息**

```java
$ git config --list
```

**初始化**

```java
$ git init
```

**克隆**

```java
$ git clone url
$ git clone url name #重命名成 name
```

**标签**

```java
$ git tag 
```

状态**

```java
$ git status
$ git diff
```

**提交**

```java
$ git add file.txt #将"当前修改"移动到暂存区(stage)
$ git commit -m "Add file.txt" #将暂存区修改提交
```

**回退**

```java
# 取消commit(比如需要重写commit信息)
$ git reset --soft HEAD

# 取消commit、add(重新提交代码和commit)
$ git reset HEAD
$ git reset --mixed HEAD

# 取消commit、add、工作区修改(需要完全重置)
$ git reset --hard HEAD
```

**查看记录**

```java
$ git reflog
$ git log
```

**远程操作**

```java
$ git remote add origin url
```

**查看分支**

```java
$ git branch
```

**创建分支**

```java
$ git branch branchname
```

**切换分支**

```java
$ git checkout branchname
```

**创建并切换分支**

```java
$ git checkout -b branchname
```

**合并分支**

```java
$ git merge branchname
```

**删除分支**

```java
$ git branch -d branchname
```

### 参考与扩展阅读

- #### [GitPro2](http://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%85%B3%E4%BA%8E%E7%89%88%E6%9C%AC%E6%8E%A7%E5%88%B6)

- #### [git-简明指南](http://rogerdudler.github.io/git-guide/index.zh.html)

- #### [learnGitBranching](http://pcottle.github.io/learnGitBranching/)

