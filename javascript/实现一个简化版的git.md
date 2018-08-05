由于工作需要，最近开始接触和使用git，为了能在使用git时更加自如（不至于把代码库删除），在工作之余也翻看了一些关于git的资料。不过很遗憾，在网络上能找到的资料大多只是介绍了git的基本用法，而对于git的底层原理却甚少提及。

庆幸，git的官方网站上有一本免费的[电子书](https://git-scm.com/book/en/v2)，其中的内容涉及到git的存储结构和内部命令，在看完这部电子书后，除了对git的底层原理有了进一步了解之外，还意外的发现，要实现一个简化版的git好像也不是不可能的事。带着好奇心，我打开了git在github上的[仓库](https://github.com/git/git)：

```shell
$ git clone https://github.com/git/git.git
$ cd git
$ git reset --hard v0.99
```

通过上面的命令，我把git的源码克隆到本地，并回到了0.99版本，这个版本的git非常简单，但常用的功能都已经包含其中，这非常有利于我的学习。在经过几天的学习和复制粘贴，我重新使用了typescript实现了其中的部分功能（`init`, `add`, `commit`, 未完待续），接下来，我会分开几篇文章来讲述我的实现过程：

* egg的诞生及egg的基本结构
* egg中的核心数据结构`Cache`
* 实现`init`命令，用于初始化数据库
* 实现`add`命令，用于向数据库中添加文件记录
* 实现`commit`命令，用于向数据库中添加提交记录

最后我把所有的代码都提交到[github](https://github.com/lizzz0523/egg)方便翻看