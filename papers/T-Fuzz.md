# T-Fuzz: fuzzing by program transformation

###### BACKGROUND

现在一般的fuzz方法都是生成一些比较随机的输入来做代码覆盖，总的来说希望覆盖到更多的执行路径从而发现更多的执行上的bugs，但是在操作系统中存在这样一种情况，如果程序的中间要求检验一些魔数，校验和，hash等就很难被覆盖到，因此作者思考是否能够有一种更好的方法能够提高代码覆盖。

fuzzing工具一共可以从输入方式上分为两类，一类是随机生成输入，一类是修改输入，主流的还是修改输入，随机生成输入的话因为要告诉输入的格式，所以还需要人力去构造。而修改输入则根据分析结果随机修改输入数据。但是目前的修改输入的方法还是无法绕过所谓sanity check真实检测的问题。这是因为有大部分的fuzzer在修改的时候还是会忽略输入的格式。

现在绝大多数的研究都是希望让输入可以尽可能地修改来发现更多的bugs，但是其实作者认为也可以让应用程序做出改变，这就是这篇文章的一个想法。

在这里要提一下所谓sanity check，这里具体可以分为两类：NCC：指的是程序中的逻辑检查需要用这个进行过滤；CC：这个和执行的函数有着密切的关系。这样NCC其实是可以直接被移除的，因为它们其实对于bug的阻碍是没有作用的，并且如果我通过这种方法产生了一种bug，那么这个bug必然是可以复现的；但是CC如果被移除就可能造成误差，那么就需要后续的方法避免这种误差的出现。

###### OVERVIEW

T-Fuzz分为三部分：使用一个市场上普通的coverage-based fuzzer，用来进行fuzz的同时记录生成的输入；一个程序转换器，如果发现fuzzer停滞了，则启动程序转换器，看是否需要做程序的转换。还需要一个crash分析器，用来检测这个崩溃是否是一个错判。

###### DETAILED DESIGN

一共分三步具体实现：

1. 检测处所有NCC候选：这里作者使用了一个轻量级的检测方式。作者只检测代码中的某些check部分，使得fuzzer使用当前任意的输入都无法继续运行。我们可以认为在一个控制流图中，一个check代码块会有输出口，分别指向了对应的false段代码和true段代码，那么这里就要关注fuzzer是否在这个代码处只走了一边。这里需要定义一个叫做边缘边的概念，这也是作为NCC候选的前提：这个边不在fuzzer所生成的边集中，但是边e的源结点却在这个执行的点集中。这样T-fuzz对于每一个输入，都会触发一个tracer帮助生成执行边序列和执行点集。然后保证在总的控制流图中，如果其中的一条边不在这个生成的边序列中但是它的源结点在执行点序列中，则认为这条边是一条候选NCC
2. 当罗列出所有的候选NCC后，就要剔除一些无用的NCC。有一种情况是如果移除了这种NCC后就只有错误句柄了，那么程序会不停报错。因此这里作者要求保证每个错误句柄前有一定的路径长度。
3. 代码转换：这里选择了静态重写的方法，因为动态二进制指令和简单调整跳转指令会带来额外开销。在这里T-fuzz的做法是将原来的NCC候选用一个否定跳转来代替。这样可以尽可能保证源程序变化非常小。每个被修改的指令的地址会作为输出记录下来。
4. 过滤：过滤器会检测如果修改过程序后经过fuzz生成的漏洞和之前未修改过的一样，则说明是一个错改；否则它会重新生成一个输入让源程序能够fuzz出这个错误。它使用了一种技术，跟踪了被修改程序的导致crash的路径来收集源程序的路径限制语句。如果结果是可满足条件的就说明是一个正例。具体是指当trace被修改过的程序的同时也会trace源程序，一开始会对输入进行预处理以保证确实会crash。如果修改程序的跟踪过程中出现了一个否定跳转标记，就把否定的跳转指令的取非加入源程序集合；否则对于一个跳转，将正常跳转加入源程序集合。如果这个跳转记录在源程序中可以被满足条件，则说明这是一个真的bug。

###### EVALUATION

* 使用了DARPA CGC数据集，这个数据集具有很多有漏洞的程序。这个和AFL，Driller进行了比较。
* 使用了LAVA数据集，这个数据集也包含了一系列的漏洞程序。
* 也使用了真实的程序。

反正效果肯定很好....，然后也对比了用了过滤器的效果。

当然最后还有有一些错误，比如出现将一个真实的错误误报成一个错报。主要的原因应该是由于循环的问题，这也是一些静态分析存在的问题所在。