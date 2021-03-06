# kAFL : Hardware-Assisted Feedback Fuzzing for OS Kernels

###### BACKGROUND

模糊测试具体要做什么：

1. 它需要准备一份插入程序中的正确的文件；
2. 它用随机的数据部分来替换文件的某些部分；
3. 然后使用该程序去打开文件；
4. 运行后看哪儿被破坏了。

这是一款基于反馈式的模糊测试工具，什么是反馈式的呢？ **反馈式模糊测试是指利用前面模糊测试的效果反馈来指导改进后面的模糊测试（uses mechanisms to learn which inputs are interesting and which are not）。**

内核fuzz的难点主要在哪？ 它的运行更加具有不确定性，比如会发生中断，发生内核线程的调度等等。另外很容易想到的是，如果使用一个进程专门去fuzz内核，那么当内核奔溃时会影响fuzz的行为，因为这个时候系统会重启。

那么这里作者提出的是一个和OS无关的一个使用硬件支撑的工具。这里有一个很有意思的问题，这个工具确实和OS无关，但是却和架构有关，并且需要特别的硬件进行支撑。

###### BASIC KNOWLEDGE

* Intel的CPU将特权级别分为4个级别：RING0,RING1,RING2,RING3。Windows只使用RING0和RING3，RING0只给操作系统用，RING3谁都能用。
* Intel VT-x 这个是intel推出的一款虚拟技术，这个如果在刚买来计算机的时候使用vmware会跟你说你必须打开这个选项。那么这个技术的目的就是将一个物理CPU抽象成多个逻辑CPU来使用，就像是进程调度一样，其实一个物理CPU上只能跑一个逻辑CPU，但是对于每一个逻辑CPU，不同时刻要的资源可能是不同的，那就让用户感觉到是同时在跑。那么在每个逻辑CPU上就可以建立虚拟机。一个虚拟机又由VM和VMM（虚拟机监视器），这个虚拟机监视器又可以叫做hypervisor，它对物理CPU有控制权但是对物理资源的访问是有限制的。如果你在虚拟机里有什么行为，则会出发一个叫做VM-EXIT的事件，这会将控制权交给hypervisor，这样就实现了在虚拟机里运行程序。
* Intel PT： 第五代intel处理器后，这个技术支持了执行与分支追踪信息的记录。这个是在i5/i7 5000以上型号上加入的功能，由于它是硬件级的特性，相比Qemu或Boch，在性能上和代码工作量会占有一定优势。它会生成两种信息：执行时信息和控制流包。那么对于这个控制流包，需要通过解码才能知道里面的数据，intel定义了五种控制流命令在这个包信息中使用，以下三种会使用到：
  * Taken-Not-Taken (TNT):  条件转移
  * Target IP(TIP) : 间接跳转（near ret or far transfer）
  * Flow Update Packets (FUP):  发生中断或者陷阱
* 系统支持一种过滤器，可以过滤指令执行。本文使用了CR3过滤器，这个过滤器帮助我们确定代码的执行是在ring0还是ring3

###### OVERVIEW

KAFL分为三个组件：fuzzing逻辑件，虚拟机设施，用户代理。fuzzing逻辑件运行在ring3，VM设施分为两部分：QEMU-PT和KVM-PT。这么设计的目的是为了让运行在用户空间的fuzz能够获取追踪到的运行情况。KAFL的运行流程是这样的：

1. 用户代理loader会使用HC_SUBMIT_PANIC方法去给QEMU-PT内核panic的句柄地址。这一步是在VM崩溃之前能更快的获取信息（覆写了崩溃句柄）。
2. 用户代理loader使用HC_GET_PROGRAM 请求实际的用户代理启动，这一步会让fuzzing完成初始化。
3. 用户代理出发一个HC_SUBMIT_CR3给KVM-PT来使得VM能够将当前CR3的值交给QEMU-PT；最后触发HC_SUBMIT_BUFFER告诉主机输入的位置，然后完成fuzzer的初始化。
4. 用户代理通过HC_GET_INPUT来获取输入。
5. 然后fuzzer会生成一个新的输入给QEMU-PT，然后QEMU-PT将输入放入指定的地址空间，随后调用VM-ENTRY让VM执行；
6. VM-ENTRY也会触发内部的PT追踪
7. 再fuzzing过程种，由QEMU-PT来解析追踪信息，一旦将控制流交回用户代理，则发出 HC_FINISHED，然后触发VM-EXIT结束跟踪。
8. 最后将解析结果反馈给之后的处理。

对于fuzzing逻辑而言，它只是管理了一系列输入，生成了变异的输入，然后评估他们，它就是使用了AFL的算法。AFL中的做法是开多个线程，每个线程独立的fuzz；而KAFL会让所有线程都运行在最感兴趣的输入上。

用户代理运行在VM中的用户层上，它是负责和客户机交互并且获取新的输入信息的。

虚拟机设施有两个部分，fuzzing逻辑是直接和QEMU-PT交互的，然后再由它和KVM-PT交互。KVM-PT负责追踪vCPU而不是逻辑CPU，而QEMU-PT的目的就是配置和切换用户空间的intel-PT然后从输出缓存中获取数据并解析。

对于有上下文依赖的代码和非确定性代码，首先通过解析的数据包中的TIP获知出现了一个中断，如果TIP中包含FUP标记则说明是一个异步事件。这样解析器会放弃接下去所有的代码块直到碰到一个iret。第二个机制是将所有非确定性的代码块置入黑名单。

###### DETAILS

KVM-PT: 作用是为了弥补之前的PT只能监视逻辑CPU而不能监视vCPU的缺陷 。逻辑CPU会收集所有的满足条件的指令，那么为了防止它接受一些不想要的数据，作者使用了MSR自动挂载技术，这可以给MSR灌输一些初始的值。当我们在追踪数据时，会将一些数据输出到一个内存缓冲区内，那么这些缓冲区的物理地址入口由一个叫ToPA的数据结构维护。当这个缓冲区满了就会产生中断，随后处理这个缓冲区中的数据并且重置缓冲区。但是由于中断的不确定性还是会导致缓冲区溢出的问题，本文中的意思是在原缓冲区下再加上一个溢出缓冲区。这是一个可变长度的缓冲区，这样当第二个缓冲区的数据依旧溢出时会通知系统增长缓冲区长度。

QEMU-PT: 这个的作用时一个PT的配置和开关。它的还有一个功能就是解析数据并且声称一个类AFL的bitmap。作者是实现了自己的解码器，并且将它转换为AFL-BitMap

AFL使用BITMAP来记录在追踪过程中遇到了哪些代码块，每一个基础的代码块都有一个特定的ID号，并且从A到B在位图中分配了一个偏移。KAFL直接使用了代码块地址，通过位图来存储某些代码块是否被运行，这样需要保存一个全局的位图。

###### LIMITATION

1. 只对CPU支持以上技术的可能使用该工具；
2. 对于即时性代码，解码器将不能进行解码；
3. 