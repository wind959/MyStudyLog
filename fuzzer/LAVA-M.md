# LAVA-M

LAVA-M是一款不一定说是专门为fuzzing量身定制的，但是也是对所有的软件测试有很大帮助的一款测试集。其实更准确地说，它属于是属于LAVA下的，含有四个特殊程序的一个测试集。它分别包括了base64，md5sum，uniq和who四个开源程序。

该测试集的本质是在这四个文件中插入了一些bug，但是由于这些bug被一些复杂的判断和校验所保护，所以一般的测试是很难探测到这些bug的。

## 使用

在网上下载了LAVA-M的测试包后，解压缩进入文件夹，首先要对每一个程序进行编译和安装，但是其实这些步骤都已经写在了每一个程序的文件夹下的脚本中了。

```
./validate.sh
```

所以只要执行这个脚本就可以做到自动的给你安装这个程序，但是要注意在安装过程中的几个坑：

* 安装的过程必须不能在根用户(/root)下进行，否则编译的过程中会报一个难以解决的问题；
* 安装过程中可能需要下载很多的库，包括acl，gmp等相关的库，如果发现缺少某个头文件了，就是缺少了相关的库了。（我这里在编译的过程中是修改了./validate.sh脚本，让它显示了编译的错误信息）
* 在编译安装完成后，会验证插入bug的个数，如果此时发现验证下来的bug数是0，说明在安装过程中出现了问题。

在这个文件夹中，还有许多其他文件，比如插入的bug的清单以及验证清单，还包括可以触发bug的输入，当然这个数据集还人性化地给我们提供了fuzzer使用的输入。

真正的应用程序可执行文件藏着以下路径中：（相对路径为LAVA-M/***）

```
./coreutils-8.24-lava-safe/lava-install/bin/***
```

当我没触发一个bug的时候都会有提示，例如它会告诉你触发了第几号bug。

最最后说一下插入的bug数吧：

| base64 | 44   |
| ------ | ---- |
| md5sum | 57   |
| uniq   | 28   |
| who    | 2136 |
