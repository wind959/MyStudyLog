# git 常用命令和基本原理

##### git status:

命令的基本作用是查看文件的状态，这里的状态主要有三种情况：

Untracked：忽略的文件，意味着该文件并未出现在远程仓库中；

Staged：位于暂存区中的还未上传到仓库中的状态；

Modified：已经提交到仓库中状态；

在正在项目中常常出现一种情况，就是修改的文件过多，使得在使用git status命令时会弹出过多信息，那么为了使得某些不必要的信息的弹出，git设计了一个文件叫.gitignore,

只需要在该文件中将不需要进行操作的文件写进去就能忽略这些文件。



##### git add:

这个命令的目的是把在工作区中改动的文件添加入暂存区，也就是将文件的状态变为Staged的一个过程，git add命令后面添加的是需要添加入暂存区的文件，一般在一个项目修改后

使用git add -A(提交新文件，包括删除了的)或者git add .



##### git commit：

这个的目的是为了将暂存区的改动提交到本地的版本库中，使用这个命令会在本地库中生成一个40位的hash。那么一般而言使用git commit -m "message"，在提交后写一段message提示这次

提交的内容是什么，方便后续查看。



##### git push：

将本地版本库中的文件推向远程版本库中，因为在之前没有接触分支的概念，因此一开始简单的小项目都是直接git push，意味着会默认提交到master分支，其实这个指令完整的操作

应该是：git push <远程主机名> <本地分支名> <远程分支名>。如果远程分支被省略，则默认推向跟踪中的远程分支；缺少本地分支相当于删除远程分支；如果当前分支和远程分支有追踪关系，

则分支名均可去除。