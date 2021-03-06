# Precise and Scalable Detection of Double-Fetch Bugs in OS Kernels

###### goal

这篇文章的背景是继续探索内核中的double-fetch的漏洞，首先普遍的认为double-fetch问题的是源自于在一次函数调用中同时对一片区域读了两次或者多次，而途中确实可以篡改读到的数据，这样就造成了double-fetch，但是作者认为double-fetch并不能如此简单的定义，由于之前工作中对于double-fetch漏洞的错误定义，导致了检测具有局限性，并且出现了许多误报和漏报，那么本文就是给出了关于double-fetch的一个完整定义，并且针对这种定义，作者又研究出了一种静态的检测double-fetch的方法。

这里还需要讲一些背景：1、内核确实是可以直接访问用户内存空间的，但是对于内核来说不会直接进入指针指向的区域，而是通过copy_from_user,get_user这种函数来获取，作者将这些函数定义为transition function，那么当然作为内核可以选择的方法是在运行内核函数时去建立一个副本，这样确实可以避免double-fetch的发生，但是这样会占用额外空间，并且占用多余的CPU时间。2、那么对于linux系统而言，最为常见的一种方式是用户态中发送一个请求，这个请求也包含了请求头和请求体，这样在第一次读取请求时先读取请求头，然后根据请求头的信息再读取请求体的信息，这样做可以在操作时减少开销。



###### definition and method

文中先介绍了作者定义的double-fetch漏洞的定义，如果一个multi-read是一个double-fetch：

1. 在一次函数调用中至少又两次从用户空间读取的过程；
2. 这两次读取的地址空间有重叠的部分；
3. 这两次读取必须包含有控制依赖或者是数据依赖；
4. deadline不能保证在第二次读取时数据还是正确的。

对于第三点，如果在代码控制流上出现上下文依赖就说明含有控制依赖；上下文中获取的数据存在依赖则说明含有数据依赖。对于控制依赖，则要时刻检测一些限制状况是否还存在；对于数据依赖，就要时刻去检查数据的正确性。

那么针对这个定义，作者给出的判断漏洞的方法：

1. 使用二元组代表一次读取过程（A，S），其中A为读取地址，S代表读取宽度；
2. 在检测是否读取有重叠的时候，只需要检测地址是否落在某个区间里；
3. 那么使用（A01,S01,{0,1}）代表了第一次和第二次从重叠的读取的区域，那么如果在这个区域中有一个变量V他受到一些控制流Vc的限制，那么需要保证第二次获取的V’这个参数也收Vc的限制
4. 同理对于数据依赖而言，如果在第一次获取的重叠的区域中的一个变量V被消费了，比如被赋值了，那么就必须保证在第二次获取时的值和第一次获取的值一致。



###### deadline overview

之所以使用上面的方法描述，很大程度上是因为这个启发了作者使用符号化的方法来判断，好处就是脱离了硬件和机器配置，并且这个理论上能够使用在子系统的代码漏洞检测上。

第一步是获取所有的multi-read，那么对于deadline而言他要生成每一个multiread的符号化的执行路径。这一步需要借助将linux源代码放入llvm中编译生成中间表达IR并且分析IR来做到。（IR记录了作者所需要的信息，并且这个IR是保存在LLVM的一个表中的，它近似于所谓的符号化表示）。

首先程序扫描源码中所有fetch的函数，然后对于每一个函数再向前或者向后找是否有其他fetch，找到一对fetch函数后就去生成这对fetch函数的控制流，当然除了捕获两个fetch函数，还要记录一下包住这两个函数的一个大函数F。

然后有了这三个函数之后，就可以生成再F中有关F0和F1的控制流了，当然需要做的一步是去掉没必要的指令。最后需要将整个控制流线性化。（这里有个现实问题就是会不会重复的调用同一个fetch多次）。

真正运行的时候需要做到一点，系统要从LLVM的IR中获取的信息转化为SR，在这一步中有许多问题，比如如果碰到了分支状况，则Deadline只会走其中一条路，并且插入一个assert标签。那如果遇到库函数呢？则Deadline不会深入到库函数中，而是需要认为编写规则来弥补这些问题，当然能这样做的前提是系统能够碰到的库函数并不多。

关于对于内存空间的建模，摒弃了传统的建模方式，即使用一个数组来表示连续的内存区域，而使用select和store方法来表示读取和存入，但是有问题就是原本的建模方式中保证了读取过程数据的不变性。那么在新的建模方法中，作者将某一个数组元素直接指向一个内存对象，这样deadline只关注指向相同内存对象的函数。



###### implement and evaluation

1. 为了追求尽可能多的代码覆盖率，它同时也编译了驱动代码，文件系统的代码和固件代码等，这个通过修改配置文件可以做到。
2. 由于编译过程中linux的内核代码和LLVM并不兼容，因此需要在编译过程中生成一份类似日志文件然后将这份文件交给llvm编译。

评估的手法是先和之前工作对比，然后证明了deadline能够检测出之前工作中所能检测出的漏洞；除此之外还检测出了一些新的漏洞，并且已经有部分漏洞给出了补丁。然后作者在漏洞总结中指出大部分的double-fetch漏洞主要出现在驱动中，当然在文件系统和网络子系统中也有许多。

double-fetch漏洞究竟会带来什么问题：

1. 信息的泄露，真的有办法可以因为size的问题fetch到内核空间的一些信息。
2. 越过限制
3. DOS攻击

那么对策呢？

1. 在第二次要值的时候直接使用第一次的值覆盖；
2. 如果检测到第二次值变了就终止程序的运行；
3. 有意不在重叠的内存区域中fetch；
4. 想办法将代码改造为只fetch一次；

**反正double-fetch的本质就是操作没有原子性。**作者提出可以使用transaction memory



###### limitation

1. 还存在部分文件是无法匹配llvm的编译的，这些文件是无法参与检测的。
2. 对于代码路径的构建，有长度限制，对于有循环的一些情况比较复杂，目前无法很好的解决，分支情况结合循环情况可能包含一些特殊例子要发掘。
3. 对于代码的构造，deadline无法处理一些内嵌的汇编代码，对于enclose函数的假设其实有漏洞。