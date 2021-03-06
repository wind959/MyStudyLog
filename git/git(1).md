# git 常用命令和基本原理

## 1、 git工作的基本流程

git最大的优势就是它是一个分布式的版本控制系统，使得开发者之间的协作可以十分灵活。

git所用到的一般可以总结为四片区域： 远程仓库（remote）、本地仓库（repository）、工作区（workspace）和暂存区（stage）

要先理解一下这四个区域各自的作用：

###### 远程仓库：

所有文件夹及文件保存在远程的服务器上（？），也就是游客们都可以进行浏览，这也是git为分布式版本控制系统的一大因素。

###### 本地仓库：

在本地建立一个git仓库，存放一些文件及文件夹，在文件夹中含有一个.git文件。

###### 工作区：

当前用户正在编辑的项目中。

###### 暂存区:

暂时存放修改过的文件的区域。

![img](https://img-blog.csdn.net/2018040822103495?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3ZpZXdzb25pY2s=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

那么git的主要工作流程就是如下：

1、从远程仓库中clone将git资源克隆到本地；

2、从本地checkout进行代码修改；

3、在提交前先将代码提交至暂存区；

4、提交修改到本地仓库；

5、将代码推向远程仓库；