title: git Internal笔记
date: 2016-10-23 16:02:43
tags: [git, LCS]
---
## git VS svn
- 在架构上
  - svn是集中式的，每个操作都只能在连上服务器的情况下完成，安全可控性更高
  - git是分布式的，每个repository都是独立完整的
- 当文件内容发生变化时
  - svn保存的是文件的增量内容
  - git保存是文件snapshot(即文件的完整内容)，容易恢复历史版本
- 版本号
  - svn采用revision number，即递增版本号。需要中心节点统一控制分配，便于记忆
  - git使用hash来作为版本号。因为当文件内容发生改变时，hash值也会改变，基本不会发生冲突的情况，可以保证全局唯一性。因此可以被当做版本标识（也可用于保证文件一致性）

  试想如果在分布式的版本控制系统中采用revision number，两个用户pull同一个仓库，假设版本号为200。然后都在本地修改并commit，都分配版本号201，那么在它们push的时候就会发生冲突。
  **更常用的用于分布式系统版本管理方法是采用[Vector Clock](https://en.wikipedia.org/wiki/Vector_clock)**

## git model
### blob object(数据对象)
blob是git中最基本的存储对象，文件的内容加上特定头部信息的**SHA-1**校验和作为文件命名。优点是hash全局唯一，能够保证分布式系统的一致性。但是代价是丢失了文件名，也不方便记忆引用。
![](/img/git_blob_object.png)
```
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30
```

### tree object(树对象)
上一小节中，单个文件可以作为blob object保存，那么丢失的文件名该怎么解决，directory又该怎么保存呢？

在git中，Git以类似于UNIX文件系统的方式存储内容。目录文件中的每一行都包含一个指向数据对象或者子树对象的 SHA-1 指针，以及相应的模式、类型、文件名信息。
![](/img/git_tree_object.png)
可以发现在blob object中丢失的文件名在tree object中都有保存。tree object的文件名也是其内容的hash值

### commit object(提交对象)
<img src="/img/git_commit_object.png" style="float:right;margin-left:1em;max-width:500px;"/>从树对象的概念可知，一个repository最后都可以以一个顶层的树对象表示。而提交对象就是包含一个commit的改动部分和提交元信息。
- Tree object of current directory
- Parent commit(s)
- Author, Committer & Timestamp
- Notes (commit message)

下面是一个commit object包含的信息，可以发现每个commit都会指向一个parent commit，总而形成一个有向无环图。每次commit都会复用未改变的object，而不会把整个目录重新复制一遍。参考下图，当目录548e8a5在第二次commit时未发生改变时，可以直接复用该tree object。所以虽然git保存的是文件快照，但是总的占用空间也较小。
```
$ git cat-file commit HEAD
tree f39298cc2122aa61ef5863a64f373e72463ffe95
parent 9f83a302a0e7cc4f7e526c77eb11a6b1068071f2
author cheng zhe <chengzhe.nju@gmail.com> 1508906732 +0800
committer cheng zhe <chengzhe.nju@gmail.com> 1508906732 +0800

add notes about git internal
```

### object storage(对象存储)
<img src="/img/git_object_storage.png" style="float:right;margin-left:1em;"/>在上述小节中讲过，git object的文件名是在content的前面加上一个header，再进行**SHA-1**操作得到。而存储git object时也会将该header写进磁盘。如右图所示，header信息包含了git object的类型，长度以及一个分隔符。接着是文件的内容。git为了节省空间，会将header+content进行zlib压缩，然后再写进磁盘。存储的路径则是hash值。

所有的git object都是以这种方式存储的，区别仅仅在于header信息里的类型标识(blob,tree,commit)以及各自对象的内容格式

### git reference(Git 引用)
<img src="/img/git_ref.png" style="float:right;margin-left:1em;max-height:350px;"/>通过上述的介绍，git每个版本间都通过commit parent连接起来，通过git log可以输出当前commit的所有历史commit信息。但是我们得知道当前的commit的SHA-1 hash值才能搞定，而git reference就是负责保存该hash值，相当于一个"文件指针"，保存在.git/refs 目录中的都是该种类型的文件。
```
$ find .git/refs
.git/refs
.git/refs/heads
.git/refs/heads/master
.git/refs/tags
```
在git中branch, tag, Remotes, HEAD都是引用。而HEAD引用则引用的是当前的分支引用（即指向其他引用的指针）
```
$ cat .git/HEAD
ref: refs/heads/master

$ cat .git/refs/heads/master
5b474b21cebf1385bfe8f3d1470c5d89c7c964b5

$ git checkout test
$ cat .git/HEAD
ref: refs/heads/test
```
## git argorithm: diff

## rebase VS merge

## reference
https://stackoverflow.com/questions/802573/difference-between-git-and-cvs
https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain
