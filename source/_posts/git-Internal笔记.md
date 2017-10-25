title: git Internal笔记
date: 2016-10-23 16:02:43
tags: [git, LCS]
---
# git VS svn
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

# git model
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
从树对象的概念可知，一个repository最后都可以以一个顶层的树对象表示。而提交对象就是包含一个commit的改动部分和提交元信息。

# git argorithm: diff

# rebase VS merge

# reference
https://stackoverflow.com/questions/802573/difference-between-git-and-cvs
