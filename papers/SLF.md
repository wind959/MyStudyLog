# SLF: Fuzzing without Valid Seed Inputs

## Background

还是以 AFL 为例，以前我们研究 AFL 主要想让它能够走到更深的分支，可以有更高的代码覆盖率，从而测试出更多的 bug。但是现在的问题往往是这样，在实际使用的软件中，有很多软件输入往往是很有结构的。比如文中提到的一个 LibZip 它的输入是一个压缩包。

文中考虑的问题是没有合法的输入，在运用像 AFL 这样的随机修改策略时，如果没有办法构造合适的输入，就会非常限制 Fuzzer 的效率。

> 前提是在没有合法的输入的情况下，往往 Fuzzer 会陷入通过暴力修改而浪费时间在搜索输入上。

**那么作者的想法是使用程序中的一些对文件的判断结果，自然地推断出背后数据结构应该是怎样的，然后根据这个去构造输入。**

这个方法当然可以借助一些代码分析工具，但是对于 Fuzzer 而言，这种方法是不可取的，因为它的目的就是尽量不使用这些工具。但是很容易联想到很多 Fuzzer 工具有借助类似符号执行的方法，比如提到的 KLEE，它能根据程序执行情况生成输入。但是这样只是对比较简单的程序，复杂的程序很容易陷入困难分支而无法解析出输入的。

因此作者的目的是希望在没有合法输入的情况下，如何根据初始输入和 Fuzzer 的反馈信息来构造合法的输入。

## Overview

SLF 的想法是，首先给一个初始输入，作者在文中给出初始的输入是一个很短的输入，比如 4 个字节的随机数。然后交给 Fuzzer 执行后，根据反馈结果，将输入的各个字节位置分成若干个组；并且分析对于每一个输入位置，程序是如何修改的；但是注意一些比较复杂的修改流程可能有内在的关联，所以需要识别内在关联，指定修改策略；最后作者提出了一种梯度多目标搜索算法来给一些字节寻找合适的输入。

SLF 并不需要程序的源码，它让可执行程序泡在 QEMU 上来分析 cmp 和 test 指令的左参数和右参数。

## Detail

### 输入检测分类

首先我们执行初始的输入，然后我们来看一下对于左参和右参的反馈。

接下去在这一部分，将会对初始输入的**各个字节**进行翻转操作，然后再执行查看反馈。这时如果对于两个不同的字节，有着相同的反馈，我们将它们归为一类(意思是改变这些字节，会产生相同的结果)；否则它们就不是一类的，可能触发了新的分支。

### 输入判断分类

这一部分的目的是为了判断在程序中对某个数据的判断是什么类型的，那么在下面针对每个类型，就会有不同的方法来修复。

在这里一共分成了四类：

* Arithmetic Check：意思是取出了这个数字，直接就进行数学类型的操作了，比如直接比大小这一类。
* Index/Offset Check：在某个元素上加上一个偏移，并且和另一个元素进行比较。
* Count Check：本质就是有关结构的长度的比较。
* ITE Check：if语句

当然它也承认，确实一个语句可能符合多个类型的判断。

### 检测程序中的判断

因为我们已经将输入的每个字节都分成了多个组，根据我们对输入的分组情况，需要对每个分组确认它们属于哪一类。

* 检测第一类：给第一类的参数随机加上一个数，如果最后比较的结果发生了变化，则说明属于第一类，这是因为一般这种比较都是一对一的映射。(mod 除外) 

* 检测第二类：给数据的最前面额外添一个字节，如果比较结果变了，就说明是第二类。
* 检测第三类：重复被计算长度的数据结构，不过这个去探索该数据结构的边界。

### 识别这些判断的内部依赖

在之前一步的给每个输入的分组进行判断类型的时候，那么你会发现会有多个类型对同一个域有影响，这就说明有着内部依赖，说明可能出现前后判断语句是有依赖关系的。

### 梯度多目标搜索算法

在这一步中作者会去寻找修改的方法，来找出合适的结果。它所做的需要从当前失败的这个判断分支出发，但是又不能违反之前已经修改好的分支情况。

作者本质的想法呢是根据一个判断语句的**左参数**和**右参数**的判断类型，这对各种组合情况制定各种修改方法。

比如说作者提出的如果左参数的判断属于第一类，而右参数是一个常数，那么先要**计算出当前分支的一个梯度值**，然后计算与他**内在与相关的分支的梯度值**，随后选择一个值属于 fd−(fc.lhs−fc.rhs)/gradient 到 fd − (rc.lhs − rc.rhs)/relgrdt 之间，如果没有这样对应的值，那就只需要对值进行调整。

那么当然对于左参数和右参数类型的不同状况，还有其他的策略，不过文中的意思都与梯度相关。

## Evaluation

这么说吧，和很多类型的 fuzzer 比了，纯粹的灰盒测试也有比如 AFL；通过符号执行的也有，比如 KLEE；还有就是混合类型的 fuzzer，比如 Driller。结果测试下来都比它们好，就因为 SLF 它能够生成可以被执行的种子，所以省却了很多无用的时间，自然性能更好。

## Discussion

首先看这篇文章很符合当下研究的内容，正好想看看如何从程序本身去生成样例。这篇文章中给出了一些想法，但是由于这篇论文中并没有提供源代码，所以对于文中一些实现细节就很难去深究了，不过还是提供了一些很好的思路。

还有就是作者提出的方法可扩展性太差，似乎就是一个个找规律，然后需要罗列各种情况一个个去解决，这样对于问题再复杂一点的情况将很难处理，可能如果要改进的话需要重新建立一些模型。

这篇文章还有一点就是让我必须要去看一下梯度下降算法是怎么个意思了。
