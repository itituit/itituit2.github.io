---
layout:     post
title:      "Intel MPX Explained"
subtitle:   ""
author:     "blankaiwang"
header-img: "img/bg-1.png"
header-mask:  0.5
catalog: true
tags:

    - Intel MPX
    - Translation
---




# Intel MPX Explained

An Empirical Study of Intel MPX and Software-based Bounds Checking Approaches
[https://Intel-MPX.github.io](https://Intel-MPX.github.io)

Oleksii Oleksenko, Dmitrii Kuvaiskii, Pramod Bhatotia, Pascal Felber, and Christof Fetzer

## Abstract

内存安全漏洞已经成为使用不安全语言如C/C++开发的软件系统的可靠性和安全性缺陷的主要原因。但不幸的是，已有的基于软件的解决方案的高额外开销阻止了这类防护的商业化大面积应用。为了解决这个问题，Intel最近公布了一个新的指令扩展 Intel MPX，一个硬件协助的全栈内存安全防护方案。
在本篇文章中，我们对Intel MPX的架构进行深入的分析，展示它的优缺点。我们的研究基于以下三个方面

1. 性能开销
2. 安全性保证
3. 可用性问题

为了保证我们的评估结果客观，我们将Intel MPX与三类优秀的基于软件的防护方案进行对比

1. AddressSanitizer
2. SAFECode
3. SoftBound

我们的主要结论是Intel MPX是一项有前途的技术，但是在现阶段还不适合进行大范围的应用。Intel MPX的开销依旧很高（平均约50%），并且支撑它的基础架构存在bug，可能导致编译或运行异常。更重要的是，MPX不能检测~~时间错误~~（temporal errors），对多线程代码可能存在漏报和误报，对一些应用程序而言，它对内存分布的约束可能需要大量的代码改写。

## Introduction

系统软件的大多数都是使用低级语言如C或C++编写的。这类语言允许对内存分布的绝对控制，这也对系统开发是至关重要的。但是不幸的是，允许对内存的直接控制经常导致内存安全性问题，如对未授权区域的非法访问。
尤其的，内存安全性问题以 _空间错误_ 和 _时间错误_ 两种形式展现出来。空间错误——也被称为缓冲区溢出或越界访问：在应用程序对软件开发者预期之外的内存区域的读或者写操作时产生。时间错误——野指针——在被访问目标被创建前或删除后进行操作时产生。
这些内存安全性问题可能导致程序崩溃、内存丢失或者其他的bug。更重要的是，这些内存漏洞也可能会被进行内存攻击——攻击者访问非法内存区域并且控制系统或窃取数据信息。这个攻击向量在低等级语言中更是广泛分布——仅在2016年，美国国家漏洞数据库公布了将近1200个内存漏洞。
针对非安全编程语言的内存安全性问题，研究者提出了大量的解决方案，从静态分析到语言扩展。在本文中，我们聚焦于 _动态边界检查_，这也被广泛认为是唯一可以防护所有内存攻击的解决方案。
边界检查技术为原有的应用程序增加元数据（动态对象的边界或允许访问的内存区间）并且为每次内存访问添加对该元数据的检查。当一个边界检查失败的时候，程序被终止，进而内存攻击也被阻止。但不幸的是，先进的边界检查技术依旧有着高额的时间开销（50-150%），使得它们依旧停留在开发阶段。
为了降低运行开销，Intel提出了一个新的指令扩展——Intel MPX。MPX的基础思想是提供硬件的支持，包括新的指令和寄存器，使得它比基于软件的边界检查更高效。
到目前为止，据我们所知，并没有对Intel MPX的全面评估——来自学术界或Intel。因此，这篇工作的主要目标就是从性能、安全性和可用性三方面对Intel MPX进行分析。_性能分析_ 至关重要，因为只有低时间开销（至高10-20%）的解决方案才能有机会被用于实际应用。同时，也必须要分析额外开销的原因来对进一步的性能提升做准备。_安全性分析_ 使用现实世界的漏洞对MPX对外声称的安全性保护进行测试。 _可用性分析_ 让我们深入了解MPX的产品质量，更为重要的是，可以帮我们发现在Intel MPX框架下，应用产生的问题，这些问题需要被人工修复。
为了全面分析Intel MPX的优缺点，我们将Intel MPX与三种优秀的基于软件的解决方案进行对比来保证评估的客观性

1. Address Sanitizer
2. SAFECode
3. SoftBound

我们的评估认为Intel MPX有很大的潜力，但是现在还不适合进行大规模应用，我们得到的结论有：

* MPX指令并没有预期中的那么快，在最坏的情况下有4倍的额外时间开销，尽管编译器在编译阶段分担了一部分开销，平均的运行时间开销还是在50%左右
* 支持MPX的基础设施（编译器和运行库）并不足够成熟，还存在bug，约3-10%的应用程序无法编译/运行
* 不同于其他的解决方案，Intel MPX没有为时间错误类内存漏洞有任何保护
* 对于多线程代码，MPX可能存在漏报和误报
* 默认情况下，MPX对分配的内存区域进行检查，约8-13%的应用程序在不进行大面积代码替换的情况下无法正常运行。另外的，我们不得不对18%的应用程序进行人工修改。

尽管前三项问题可以在今后的版本中进行修复，后两项则被认为是基础设计带来的问题。我们认为添加多线程的支持必然会对性能开销带来不利的影响，宽松内存检查的策略也与MPX的设计理念相违背。

## Background

所有的空间和时间bug，以及在这类缺陷上开展的内存攻击，都是由于对非法内存区域的访问引起的。为了解决这类缺陷，程序的内存安全问题必须得到解决，如下述规定必须被执行：内存访问必须被限制在初始设定（指向）的对象范围内。
内存安全可以使用多种方式实现，包括纯粹的静态分析，基于硬件的检查，基于概率的策略和对C/C++语言的扩展。在本文中，我们着重对基于对原有程序插桩的运行时边界检查技术进行分析。这类技术提供了最高等级的安全性保证，同时，所需人工对程序进行调整的工作量又很少。

现有的运行时检查技术可以被大致分为~~阈值控制~~（trip-wire），基于对象的和基于指针的方法。基本的，这三类方法都对程序的初始数据的元数据的边界信息进行了创建、跟踪和检查的工作。阈值控制的方法为整个程序内存创建了”影子内存”，用于存放元数据；基于指针的方法为每个指针创建边界元数据；基于对象的方法为每个对象创建边界元数据。
为了和MPX进行对比，我们为上述提到的每一类选出一个对象，分别是AddressSanitizer，SAFECode和SoftBound。Figure 1给出了他们之间的区别。

![Figure 1](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%201.png)

**阈值控制方法：AddressSanitizer：** 这类方法为每个对象使用被标记（投毒）的内存进行包围，这类内存被称为 _redzones_，这样任何的溢出都会对 _redzone_ 中的内存信息进行修改，如果没有溢出， _redzone_ 中的信息就保持不变。_redzone_ 的完整性会被不断的检测。特别的，AddressSanitizer保留了整个虚拟内存空间大小的1/8用于构建 _影子内存_，这类内存只能被插桩代码访问。当一个新的对象被创建或释放的时候，AddressSanitizer对影子内存中的信息进行更新，并且对访问该对象的内存操作的前面插入对影子内存的检查。这个检查如下：

```C
ShadowAddr = MemToShadow(ptr)
if (ShadowIsPoisoned(shadowAddr))
  ReportError()
```

另外，AddressSanitizer通过提供 _隔离区_ 的方式实现对时间错误的检查：如果一个内存区域被释放，AddressSanitizer在该内存区域被允许重用前的一段时间内依旧保留其元数据。
AddressSanitizer的设计初衷是为了debug而不是安全性（尽管也可以被用于这个领域），例如它不能检测非连续的越界问题。但最基本的，它能够检测出很多时间错误缺陷从而显著提高攻击者攻击的难度。AddressSanitizer也是在阈值控制方法领域内应用最为广泛的技术，相比于Light-weight Bounds Checking, Purify和Valgrind。
**基于对象的方法：SAFECode：** 这类方法的主要思想是保持初始指向的正确性，如保证对指针的操作不会改变指针指向的对象。在SAFECode中，这条策略更加粗粒度：每个对象被分配在一个在编译时通过指针分析定义好的粗粒度区域 _pools_ 内；指针范围必须总是落在这个事先定义好的pool内。这项技术极大的优化了检查策略并且简化了运行时检查的工作量：

```C
poolAddr = MaskLowBits(ptr)
if (poolAddr not in predefinedPoolAddrs)
  ReportError()
```

但是另一方面，SAFECode的安全性保证不如AddressSanitizer——在pool内的对该对象的缓冲区溢出就无法检测。
我们也考虑了其他的基于对象的方法，但最终没有采用。CRED带来高额的性能开销，mudflap在新版本的GCC中无法使用，Baggy Bounds Checking没有开源。
**基于指针的方法：SoftBound：** 这类方法对指针的边界进行检查，并且在每次对内存的读写时对访问的范围进行检查。注意SoftBound不是对对象而是对对象的一个指针进行元数据处理。通过对指针的 _边界压缩_ ，使得检查在同一对象内的溢出成为了可能（在同一数据结构内，一个域溢出到了另一个域）。
Intel MPX同SoftBound很像；另一方面，一个被称为WatchdogLite的SoftBound的硬件实现也同MPX有很多共同点。在我们的对比中，我们使用了SoftBound+CETS的组合来保证指针的元数据在一个二级索引中，就像MPX的bound table一样。同时也引入了对时间错误的检查机制。这类检查逻辑如下：

```C
LoBound,UpBound,key,lock = TrieLookup(ptr)
if (ptr < LoBound or ptr > UpBound or key != *lock)
  ReportError()
```

对于其他的基于指针的方法，MemSafe没有开源，CCured和Cyclone需要对程序的人工修改。

## Intel Memory Protection Extensions

Intel MPX在2013年被首先提出，并且在接下来的2015年被宣布成为Skylake微架构的一部分。Intel MPX的初衷是为传统的C/C++程序添加透明的边界检查。考虑在Figure 2a中的代码片段，原始程序分配了一个`obj`类型的包含10个指针的数组`a [10]`。接着，程序便利了前`M`项来计算对象的长度。在C语言中，这个循环如下所示：

```C
for (i = 0; i < M; i++)
  total += a[i] -> len;
```

`M`是一个变量，当出现bug或一个恶意行为将设为大于`obj`大小的值就会引起缓冲区溢出。同样的，注意在第四行中使用指针 `ai` 访问数组 `a [i]`和在第六行中使用`lenptr`访问子域的行为。
Figure 2b展现了MPX保护后的代码。首先，在第三行中创建了数组`a [10]`的边界（这个数组包含了10个8 byte的指针，因此上界的偏移量是79）。接下来在循环中，在第八行的数组访问前插入了两个MPX边界检查指令来检查`a [i]`是否发生了溢出。注意由于这个保护的读操作从内存中读取了8字节的指针，也需要对`ai+7`的上界进行检查。

![Figure 2](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%202.png)

现在对象的指针使用`objptr`进行加载，程序想要访问名为obj.len的子域。在设计上，Intel MPX必须通过检查`objptr`的边界来保护这第二次读操作。这些边界从哪里计算而来呢？在MPX中，内存中的每个指针都有他们的边界，边界被存储在特殊的内存区域，这个内存区域只能使用MPX指令`bndstx`和`bndldx`访问。这样当指针`objptr`指向了内存地址`ai`，它对应的边界也被`bndldx`被恢复成为了同样的地址。最后，这两条边界检查差指令被插入到了对长度值的检查之前。
MPX需要在硬件-软件栈的每一个层级上进行相应的修改：

* 在 _硬件层_ 上，新加了一套指令和一套128位长的寄存器。同时，也引入了由这些新指令抛出的边界越界异常（#BR）。
* 在 _OS层_ 上，添加了#BR异常的处理逻辑，包含以下两个主要功能：（1）按需分配边界的存储空间和（2）无论何时检测到了边界越界，程序的标志位置位。
* 在 _编译器层_ 上，新的MPX转化pass被用来向程序插入MPX指令，实现对边界的创建、传递、存储和检查。补充的 _运行库_ 提供了初始化和终止化的程序，静态和调试信息以及对C标准库函数的打包功能。
* 在 _应用层_ 上，MPX保护的程序可能需要人工的调整，这是由于不标准的C编码形式、多线程问题或与其他微指令扩展的潜在问题引起的。（在某些情况下，并不推荐使用Intel MPX）。

在接下来，我们详细介绍在硬件-软件栈的每一层级上的MPX支持。

### Hardware

在这个层面上，Intel MPX提供了7条新的指令和一套128位长的边界寄存器。现阶段的Intel Skylake架构提供了4个寄存器，被称为bnd0-bnd3。每个边界寄存器都可以在0-63位存储64位的边界下界以及在64-127位存储边界上界。
**指令集.** 新的MPX指令包括：`bndmk`用于新建边界，`bndcl`和`bndcu/bndcn`用于比较边界寄存器`bnd`同代码指针的下界和上界，`bndmov`可以将边界值从一个边界寄存器移到另一个边界寄存器或将边界值从边界寄存器移到栈上，`bndldx`和`bndstx`用于向Bound Table读或写。注意`bndcu`有另外一个完全相同的版本`bndcn`，因此在接下来我们只使用`bndcu`。Figure 2b给出了这些指令中大多数的用法，没有展示的`bndmov`主要用于内部寄存器或栈的重排。
MPX也改变了x86-64的调用方式。简而言之，对应的指针参数在函数被调用前就被放在寄存器`bnd0-bnd3`中，函数的返回指针的边界在函数返回前被放在寄存器`bnd0`中。
我们将基于硬件实现的方法带来的优势同基于软件的实现方法SoftBound做一个对比。首先，Intel MPX引入了边界寄存器来降低了通用寄存器的负担，这也是基于软件的方法无法做到的。其次，基于软件的方式无法改变函数调用方式，当函数参数包含指针的边界时，只能求助于函数克隆（function cloning）。这使得函数的调用/被调用关系更为复杂，并且使得传统的未插桩库文件的通用性变差。最后，专用的`bndcl`和`bndcu`指令替换掉了基于软件实现的“比较-分支”的指令序列，减少了一个逻辑循环并且对分支预测器不会带来额外的压力。
Intel MPX的亮点在于其后向适用性和对未改写代码的通用性。在一方面，MPX插桩后的代码可以在传统硬件上运行，因为MPX指令在旧版本的架构上会被编译为nop，这样简化了二进制文件——相同的启用MPX的程序/库文件可以在不同的平台上使用。在另一方面，MPX为未改写代码有全面的通用性支持：（1）标志位`BNDPRESERVE`允许传送由未改写代码创建的不包含边界值信息的指针；（2）当未改写代码改变了内存中的一个指针时，对该指针的`bndldx`操作会注意到这个变化并赋值一个恒真的边界值`INIT`。在这两种情况下，由未改写代码创建/修改的指针会被认为是“无边界”的：这给MPX带来了通用性，但同时也对MPX的防护带来了漏洞。
**内存中边界值的存储.** 现阶段版本的MPX只包含4个边界寄存器，这对现实世界的应用程序而言显然不够，当有5个指针时，边界寄存器就不足。因此，所有的边界值必须在内存中进行存储，就如同将数据存储到通用寄存器一样。一个简单并且快速的解决方案是使用`bndmov`将他们存储到编译器定义的内存区域中。但是在这只在单一的栈帧内可行：如果有一个指针接下来在另一个函数中被重用，它的边界值也就丢失了。为了解决这个问题，MPX引入了两个指令`bndstx`和`bndldx`。这两条指令根据指针地址自身向指定内存区域写入或读取边界值，尽管实现略微复杂，但是使得在不需要任何额外信息的情况下，定位指针边界值的行为简化。
如同虚拟地址转化一样，`bndstx`和`bndldx`也是对存储在内存中的一个二级地址转化结构的边界值进行操作。每个指针都在Bound Table（BT）中有一个入口，同页表一样也是动态分配的。BT的地址被存储在Bounds Directory（BD）中，就如同页索引一样。对于一个指针，它在BD和BT中的入口地址就来源于它在内存中的地址。
注意我们在这里将边界值表同页表的对比仅限在概念上，实际的实现上有显著的不同。首先，MMU并不介入地址的转化过程，所有的操作都由CPU完成；其次，也是最重要的，MPX不具备一个专用的缓存（如TLB缓存那样），它必须同应用的数据共用缓存。在某些情况下，这会导致进程的快速置换问题，并带来高额的性能开销。
地址转化是一个多阶段的过程。考虑读取指针边界的情况（Figure 3）。在第一阶段，需要读取对应的BD入口。CPU需要完成步骤1-3，第二阶段，CPU需要完成步骤4-7：

![Figure 3](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%203.png)

1. 读取指针地址的20-47位来提取BD入口的偏移量，并将这个偏移量平移3位（这是由于所有的BD入口都是8位长）
2. 从寄存器`BNDCFGx`读取BD的基地址
3. 将基地址同偏移量相加，并根据相加结果读取BD入口（BT基地址）
4. 根据指针地址的3-19位提取BT入口，并将结果平移5位（这是由于所有BT入口都是32位长）
5. 将读取到的BT入口地址平移3位，从而清除前3位的元数据
6. 将基地址同偏移量相加
7. 最终根据指针地址得到了BT入口

注意BT入口有一个额外的“指针”域，如果实际指针值和该指针域中的指针值不匹配，MPX就会将边界值标记为恒真（`INIT`)。这是由于对未改写代码的通用性支持引起的，并且只在该代码对指针进行操作时会产生这种情况。
根据指针地址读取BT入口的操作的开销是高昂的，需要大约3次寄存器-寄存器的移动，3次平移以及2次内存读操作。除此之外，由于这两次读操作是不连续的，因此被保护的程序的缓存优先级会更低。
**同其他微指令扩展的交互性.** Intel MPX在同其他微指令扩展共同使用时，可能会产生问题，如Intel TSX和Intel SGX。在Intel TSX的硬件事务中应用MPX可能会在某些特殊情况下引起事务性中止。同样的，由于Bound Table和#BR异常由OS进行管理，MPX不能被应用在类似于SGX enclave的环境中。事实上，恶意的OS可能会影响这些数据结构并进而篡改MPX的正常执行流程。为了阻止这种情况的发生，MPX允许将它的功能放到SGX的enclave中去并进而验证每次OS的行为。最后，我们不考虑可能在enclave中使用MPX的侧信道攻击情况。
**微基准测试.** 作为我们评估的第一部分，我们分析MPX指令的延时和吞吐量。Agner-Fog是评估CPU指令性能的重要标准。为了进行该测试，我们扩展了建立Agner-Fog指令表的代码。对于每次运行，我们为所有的`bnd`寄存器使用假值进行初始化，从而避免由于失败的边界检查带来的中断。
Table 1展示了延时和吞吐量的测试结果，Figure 5展示了MPX指令使用的执行端口。如同我们预期的那样，多数操作的延时为1个周期，如最为常用的`bndcl`和`bndcu`。而性能的平静主要在于对边界的存储和读操作，也就是`bndstx`和`bndldx`。如同我们在刚才介绍的那样，这两条指令在访问Bound Table时使用了一套复杂的算法。

| 指令         | 描述                             | 延迟 | 吞吐量 |
| ------------ | -------------------------------- | ---- | ------ |
| `bndmk b,m`  | 创建指针边界                     | 1    | 2      |
| `bndcl b,m`  | 根据内存操作数地址检查指针下界   | 1    | 1      |
| `bndcl b,r`  | 根据寄存器操作数地址检查指针下界 | 1    | 2      |
| `bndcu b,m`  | 根据内存操作数地址检查指针上界   | 1    | 1      |
| `bndcu b,r`  | 根据寄存器操作数地址检查指针上界 | 1    | 2      |
| `bndmov b,m` | 从内存中取出指针边界             | 1    | 1      |
| `bndmov b,b` | 将指针边界移动到其他的寄存器     | 1    | 2      |
| `bndmov m,b` | 将指针边界移动到内存             | 2    | 0.5    |
| `bndldx b,m` | 从BT中读取指针边界               | 4-6  | 0.4    |
| `bndstx m,b` | 向BT中存储指针边界               | 4-6  | 0.3    |



Table 1：MPX的延迟和吞吐量；`b`——MPX边界寄存器；`m`——内存操作数；`r`——通用寄存器操作数

![Figure 5](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%205.png)

在我们的试验中，我们注意到Intel MPX的保护没有增加程序的IPC（指令/周期），这在其他的内存安全保护方案中是常见的（见Figure 11）。这也是出乎我们意料的：我们本以为MPX可能会利用它所拥有的CPU资源来提高较低的IPC。为了理解造成这个瓶颈的原因，我们评估了典型的MPX检查语句的吞吐量。
我们的评估发现，瓶颈在于`bndcl/u b,m`指令在端口1上的竞争。在没有检查的情况下（Figure 6 a），我们的初始基准可以同时进行两个读操作，实现2IPC的吞吐量（注意读取的数据一致保持在内存调用缓冲区中）。在添加了`bndcl/u b,r`检查后（Figure 6 b），IPC增加到了3：一次读操作，一次边界下界检查，一次边界上界检查。对于`bndcl/u b,m`检查（Figure 6 c），IPC比初始更 _低_ 了：2次读操作和4次检查被分配在4个周期中完成，因此IPC变为1.5。最终，IPC是在约1.5-3的范围内（初始IPC为2），因此 _MPX程序的 IPC 与原始程序相当_ 。

![Figure 6](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%206.png)

如Figure 9和Figure10所示，这带来了显著的性能下降。如果下一代CPU为这类内存地址运算提供了新的端口，这样，这些检查就可以并行执行，MPX的性能也会由此得到提升，从而解决这个问题。我们推测GCC-MPX同AddressSanitizer采用了同样水平的解决方案，因为他们的插桩时间大致相当。对应的，ICC版本的性能可能会更好，额外时间开销降至20%以内。但是我们必须提出我们没有任何证据可以证实我们的推测。

### Operating System

在MPX的设计中，操作系统有两项主要的任务：处理边界异常和管理BT，如创建和删除BT。这些行为都被连接到一类新的异常处理：#BR。#BR专门为了Intel MPX引入，并且同页错误有相似之处，尽管#BR还有一些其他的功能。
**边界异常处理.** 如果一个启用了MPX的CPU检测到了边界异常，如一个指针被发现指向了边界之外，就会抛出一个#BR异常，并且处理器陷入内核态（在Linux下）。内核对指令进行解码来获取产生异常的地址和边界，并将它们存储到一个名为`siginfo`的数据结构中。接下来，它将`siginfo`中记录的异常信息同`SIGSEGV`信号一同发送给应用程序。此时，程序开发者有两个选择：提供一个自行的信号处理机制来恢复进程或选择一项默认的策略：崩溃，打印错误信息并继续执行或忽略掉这个异常。
**管理Bounds tables.** 两级边界地址转化使用不同的管理策略：BD在程序启动伊始由运行库创建，而BT只能被动态按需创建。对BT的创建是OS的任务。在Figure 4中展示了这个过程。

![Figure 4](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%204.png)

1. 应用程序尝试进行指针边界值的存储
2. CPU读取对应的BD入口并检查是否是合法入口
3. 如果检查失败，CPU抛出#BR异常并陷入内核
4. 内核建立一个新的BT
5. 在BD入口中存储BT的地址
6. 返回至用户空间
7. CPU在新建立的BT中存储代码指针的边界，并继续执行该应用程序

由于对BT的操作对应用程序透明，因此OS也要负责对BT的释放。在Linux下，这项“垃圾回收”操作在内存对象被释放（回收）时进行。OS遍历内存对象的元素，并删除所有对应的BT入口。如果BT为空，OS释放整个BT，并对应的在BD中删除该BT的入口。
**微基准测试.** 为了评估分配和回收BT的额外开销，我们为最坏的情况准备了两套微基准测试。第一种情况：大量指针，并且每个指针的内存地址使得它们都独立存在于BT中，这种情况下，需要建立大量的BT；第二种情况类似，但在BT被分配以后立即释放所有内存空间，以此触发BT回收机制。我们的评估结果在Table 2中（在评估中我们关闭了所有的编译器优化，从而独立评估BT的分配和回收对OS的影响）。在多数情况下，MPX保护的版本同原始版本的运行参数（如缓存优先级、分支预测失败数等）相当。但是，运行速度的减慢是值得注意的——超过两倍。这是由于单一因素引起的——在内核态执行的指令数。也就是意味着额外的时间开销单纯由内核态的BT管理带来。由此我们可以得到OS会导致MPX程序的运行会变慢至高2.3倍，尽管这种情况相当罕见。

| 类型      | 时间开销 | 指令增加数%(用户空间) | 指令增加数%(内核空间) |
| --------- | -------- | --------------------- | --------------------- |
| 分配      | 2.33×    | 7.5                   | 160                   |
| 分配+回收 | 2.25×    | 10                    | 139                   |

Table 2：MPX对OS的最坏影响
在本节中，我们仅讨论了Linux下的实现。但是，在Windows下也采用的是相同的BT管理机制。唯一显著的区别在于Windows下的MPX支持是由守护进程完成的，而在Linux下是由内核实现相应的功能。

### Compiler and Runtime Library

MPX在新的指令和寄存器的硬件支持上显著降低了每次 _独立的_ 边界检查操作的性能开销。但是，性能、正确性、和整个应用程序的完整边界检查主要依赖于编译器及其相关的运行环境。

**编译器支持.** 至今为止，只有GCC 5.0+和ICC 15.0+编译器支持Intel MPX。为了支持MPX对应用程序的保护，GCC和ICC都引入了新的被称为指针检查的编译器pass。通过向正常的编译过程添加一系列选项，就可以启用MPX支持。

```bash
#gcc -fcheck-pointer-bounds -mmpx test.c
#icc -check-pointers-mpx=rw test.c
```

简单的，指针检查pass按照下述规则对原始程序进行插桩

1. 为全局变量分配静态边界以及在栈上变量前插入`bndmk`语句
2. 为每次指针的读写操作前插入`bndcl`和`bndcu`边界检查指令
3. 在旧指针的基础上建立新指针时，使用`bndmov`将旧指针在`bnd`寄存器中的信息移动到新的寄存器中
4. 在`bnd`寄存器不足时，使用`bndmov`指令将最少使用的边界存储到栈上
5. 在指针从内存中读/写操作时，使用`bndldx`和`bndstx`也对指针的边界值进行读/写操作

相比于AddressSanitizer和SAFECode，MPX的一大优势在于他的设计上就支持 _对数据结构的边界值进行压缩_ 。考虑Figure 2中的结构`obj`，它包含两个域：100B的缓冲区`buf`和在其后的整型`len`。显然的，对于仅有1位的`obj.buf`缓冲区溢出而言，紧随其后的`obj.len`会被覆盖并破坏。AddressSanitizer和SAFECode在设计上就不能检测这类在同一对象内的溢出（尽管AddressSanitizer可以检测这类溢出的一个子集）。相比而言，MPX可以通过边界压缩从而对数据结构的某个域进行保护，如在Figure 2b的第10行。此时，编译器将`objptr_b`的边界压缩为4字节，并在11-12行对压缩后的边界进行比较，而不是与整个对象的边界进行比较。边界压缩可能需要对源代码进行一些调整，并且是默认开启的。
默认情况下，MPX pass对内存读和写都进行插桩：这使得对缓冲区的越界写和越界读都有所防护。但这种设计想法是多余的。首先，只对写操作进行插桩可以显著降低MPX的性能开销（从GCC的2.5倍降低到1.3倍）；其次，高危bug集中在对内存区的越界写操作（经典的，通过缓冲区溢出获取远程计算机的权限），仅对写操作进行保护就可以提供等效的高安全性保证。
至少在GCC的实现下，pass可以通过附加编译选项进行微调。在我们的经验中，这些编译选项不提供任何的性能、安全性或可用性的优势。对于所有提供的变意思选项，可以参照MPX的官方文档。
性能上，编译器必须尽量优化冗长的MPX代码。通常，在GCC和ICC下有两种优化方式（也被用于Baggy Bounds的优化）。（1）移除编译器可以通过静态判断内存访问安全性的边界检查（如在已知偏移量的情况下对数组内部的访问）；（2）将循环中的边界检查提取到循环外部。如在Figure 2b中的情形所示，如果已知M<=10，则这类优化可以（1）移除6-7行的恒真检查；或者第2种优化方式可以将这些检查提取到循环的外面，这样在每次执行中都可以减少两条指令的执行。
**运行库.** 作为使能MPX进程的最后一步，应用程序必须同两个MPX专用库进行连接：`libmpx`和`libmpxwrappers`（在ICC下是`libchkp`).
`libmpx`用于在应用启动之初对MPX进行初始化：启动硬件和OS支持并且进行MPX运行环境的相应配置（传递环境变量）。这些选项更多集中在对调试和日志方面，但是其中的两个关乎到安全性保证。`CHKP_RT_MODE`必须被设置为“stop”，这样应用程序在检测到边界越界后会立即停止执行；只有在调试的时候可以将其设置为“count”。另外，`CHKP_RT_BNDRESERVE`定义了允许应用程序调用未改写的外部库文件中的传统函数的情况，如果整个函数都被MPX保护，这个参数必须被使能。
默认情况下，`libmpx`占用了一个变量，这个变量既不会中止程序运行也不会写入调试信息（取决于运行选项）。但是，这个变量可以被用户自定义的变量进行覆写。这在程序必须退出或需要进行自检时可能会有用。
另外一个有趣的事情是用户可以定义`libmpx`来拒绝由OS创建BT。在这种情况下，#BR异常会直接交给程序进行处理，由程序自行创建BT。这在用户完全不信任OS的情况下会用得到（如在SGX的enclave中）。
GCC的`libmpxwrappers`（ICC的`libchkp`）对C标准库中的函数进行打包。类似于AddressSanitizer，没有对libc进行插桩而是对libc中的所有函数提供了一个包含边界检查的副本。
**讨论.** 对于GCC和ICC，编译器和运行时支持都有一系列问题，列在了Table 3中。

| 编译器和运行问题              | GCC   | ICC  |
| ----------------------------- | ----- | ---- |
| MPX pass优化问题              | 22/38 | 3/38 |
| MPX编译器问题                 |       |      |
| 函数调用的不正确边界          | -     | 2/38 |
| 和自动向量化pass的冲突        | -     | 3/38 |
| 由于C99 VLA向量引起对栈的破坏 | -     | 3/38 |
| 未知的编译器内在错误          | 1/38  | -    |
| 运行库文件中的问题            |       |      |
| 对libc函数的打包缺失          | all   | all  |
| 对`memcpy`打包的指针边界清零  | all   | -    |
| 对`memcpy`打包的性能bug       | -     | all  |

Table 3：编译器pass以及运行库文件中对MPX支持的问题。
考虑到性能问题，现阶段的GCC和ICC采用了不同的方式优化MPX代码。GCC相对更为保守，更多的考虑初始程序的稳定性，牺牲了一部分性能。在很多情况下，我们注意到GCC MPX pass  _禁止_ 了其它的优化，如解循环和自动向量化。同时，相比于ICC，GCC也更少将循环中的MPX语句提到循环之外。ICC的优化更为激进，同时，ICC并不阻止其他对MPX的激进优化选项。但是这也使得ICC的pass不那么稳定：我们检测到由错误优化带来的三类编译器bug。
同时，我们注意到运行打包库的问题。首先，只有最流行的libc函数被包括进去，如`malloc`, `memcpy`, `strlen`等，这也使得在调用其他函数时可能会带来未检测的bug。为了产业化应用，现有的库文件必须被扩展至 _所有_ 的libc库。其次，大多数的打包遵循了“检查边界然后调用实际的函数”的处理原则，但是实际上有更为复杂的应用场景。如函数`memcpy`不能只将指定的内容从内存区域复制到另一个内存区域，还要将对应的BT中的指针信息进行复制。GCC库使用了一个快速的算法完成这项操作，而ICC的`libchkp`存在着性能瓶颈。
**微基准测试.** 为了分析不同的编译器选项和优化策略，我们编写了四个微基准测试，每一项对应MPX的一项特性。两项微基准测试`arraywrite`和`arrayread`对内存进行写入/读取操作，同时对`bndcl`和`bndcu`进行压力测试。微基准测试`struct`在数据结构内部创建一个数组，从而对边界压缩特性以及实现该特性的`bndmk`和`bndmov`进行压力测试。微基准测试`ptrcreation`持续创建新的指针，对边界增加的特性以及实现该特性的`bndstx`指令进行压力测试。Figure 7展示了相比于初始版本的性能开销。

![Figure 7](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%207.png)

我们可以注意到几个有意思的细节。首先，对于`arraywrite`和`arrayread`，所有的开销都是由边界检查指令带来的，约50%。由于需要创建并将边界存储到栈上或从栈上读取创建的边界，`struct`的开销更高，约2.1-2.8倍。而新建边界带来的开销是MPX操作中最高的，`ptrcreation`的开销约为5倍。这样高额的开销使得在执行具有大量边界读取和存储的指针密集型应用时，额外开销是不可接受的。
其次，在`arraywrite`中，GCC和ICC有约25%的差异。这就是优化带来的不同：GCC的MPX pass禁止了解循环而ICC充分利用了解循环带来的性能优势（有趣的是，在`arrayread`中也有类似的情况，但是由于ICC自身的优化做的过于优秀，以至于相比而言，ICC MPX pass的性能开销显得更高）。
再次，在采用只对写操作进行检查的MPX版本中，`arrayread`的性能开销几乎可以忽略：在这个微基准测试中，读操作没有进行插桩。最后，同样的情形在`struct`中也存在：禁用了边界压缩也就大量移除了产生大量开销的`bndmk`和`bndmov`指令，进而大幅降低了整个程序的开销。

### Application

在应用层，我们发现了MPX的两个主要问题。首先，MPX不支持C语言的几种常用风格（有些是由于设计问题，有些是由于实施选择问题），这使得应用程序不可执行；其次，也是最为重要的一点，就是MPX不支持多线程程序。
**不支持的C语言风格.** 如同我们刚刚介绍的那样，MPX的一项主要特性——边界压缩——能够显著增加程序的安全性。因为边界压缩使得明确指向一个复杂数据结构的某一域的指针不会指向其他的域。但是，我们的评估发现边界压缩也会破坏许多程序。产生这一问题的原因在于C/C++程序经常偏离标准的内存模型。
在C99之前，一个通用的C语言风格是以单一指针大小为变动单元的灵活的指针域，如`arr[1]`。在实际应用中，这样类型的对象拥有一个动态的大小，且通常 _大于_ 单个指针变量的大小，但是MPX在编译时对此无从得知。因此，MPX在`arr`被访问时，将它的边界压缩到单个变量的大小，这也就带来了误报。类似的，还包括不支持Intel MPX的由变量定义大小的数组，如`arr[]`。这些编程风格在当前的程序中依旧流行，如Table 4的第一行所示。注意现阶段的C99标准，`arr[0]`已经可以被正确处理，不会带来程序的崩溃。

| 应用级的问题                           | GCC  | ICC  |
| -------------------------------------- | ---- | ---- |
| 不定或变量大小的数组（`arr[1]/arr[]`） | 7/38 | 7/38 |
| 根据数据结构的一个域访问整个数据结构   | 1/38 | 3/38 |
| 自定义内存管理                         | 2/38 | 2/38 |

Table 4:破坏MPX内存模型假设的应用数量
另外一种常见的编程风格是使用数据结构的一个域（通常是这个数据结构的第一个域）访问数据结构的其他域。同样的，这也破坏了MPX的初衷，并且会带来运行时的#BR异常（见Table 4的第二行）。由于这种问题的通用性，GCC为这种情况进行了例外处理，但ICC没有为此进行额外的规定。
最后，一些程序可能会为了性能问题，进行可疑的内存操作，这破坏了C语言的内存完整性。如在SPEC2006中就有两个例子：`gcc`自身有一套对任意类型变量的内存管理机制，并且会进行指针内的位操作；`soplex`有一套通过为每个指针添加偏移量，实现快速将一个内存区域的内容移到另一个内存区域的机制（Table 4的第三行）。这两种情况都会引起MPX的误报。
最终，所有这些不符合常理的情形都应当被修复（事实上，我们修复了MPX不支持灵活的指针域/变量定义大小数组的问题）但是，在一些情况下，用户对修改原始代码有强烈的抵触心理。这个时候，用户可以选择略微牺牲安全性，通过使用`fno-chkpnarrow-bounds`选项来禁止边界压缩。另外一种非介入式的解决方案是通过添加一个特殊的编译器选项，标记不进行边界压缩的对象。
**多线程问题.** 当前的Intel MPX实现可能会对多线程应用带来漏报和误报问题。这是由于MPX使用`bndldx`和`bndstx`进行指针边界的读取/写入的操作方式所带来的。当一个指针从内存中被读取时，这个指针对应的边界值也需要从BT中进行读取。
在理想化的情况下，对指针的读取和其边界的读操作印个当具备 _原子性_（对于写入操作也是一样的）。但是，现阶段的硬件实现和GCC/ICC都没有进行原子化的实现。这种多线程支持问题会导致（1）由于误报引起的程序崩溃或（2）具备缺陷的程序在MPX的保护下，缺陷依旧可以被利用。
考虑Figure 8中的情形：对数组指针`arr`的数据存在竞争。后台线程可以选择以第一个或第二个变量进行填充。与此同时，主线程访问当前数组项所指向的对象。注意此时依据常量`offset`的值，这段程序可能是“正确”或“充满错误”的：如果`offset`为0，主线程就会一直访问正确的对象；否则，主线程就会一直访问一个错误并且与之相邻的对象。如果在现实中的代码中出现了第二种情况，这个漏洞就可能被攻击者所利用

![Figure 8](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%208.png)

在MPX的保护下，在第二行代码前会根据第一个对象插入`bndstx`指令，存储对应的边界值（对于第三行代码和第二个对象同样如此）。另外，在第5行代码前会插入`bndldx`来对`arr[i]`访问的变量的边界进行读取。在第5行代码前也会插入对边界的检查指令`bndcl`以及`bndcu`。现在，就会产生下述的竞争情况。主线程在后台线程先占的情况下，读取了第一个变量的边界值，但是后台线程对数组内容使用第二个变量的数据进行填充，同时赋予了对应的边界值。此时主线程的指针指向了第一个变量，而边界值是第二个变量的边界。
我们使用C语言编写了对应的测试用例并使用GCC和ICC分别进行编译，如我们预期，这样的MPX程序同时存在了漏报和误报。
对于一个正确的源程序（`offset=0`），程序在第5行访问变量时会产生误报。事实上，指向这个变量的指针时正确的，但是边界值被后台线程使用第二个变量的边界值进行了覆写，因此MPX产生了误报。对于终端用户而言，调试这样的bug显然不是什么令人开心的事情。
对于一个错误的源程序（如`offset=1`），情况更为复杂。事实上，MPX被用于检测所有的越界访问，但是在此时会产生漏报的情况。此时，指向第一个变量的指针在添加偏移量之后恰好落在第二个变量的范围内，但是由于MPX恰好错误的使用第二个变量的边界值进行检查，因此此时MPX不会抛出越界异常。我们相信这种在多线程情况下的 _潜在_ 越界漏报问题会使开发者对MPX的兴趣大大降低。我们同样相信，对于一个有经验的攻击者而言，可以利用这一特点构造攻击代码，以很大概率实现对MPX防护的绕过。
我们注意到在Phoenix和PARSEC多线程测试套件中没有发现类似的问题——可能我们运气足够好，没有碰到会由MPX产生崩溃的程序。
为了在多线程应用中的安全使用，MPX的插桩必须需对指针及其边界值的读取/存储操作保持其原子性。对于软件（编译器）层级而言，可以使用对`mov-bndldx/bndstx`的同步原语、细粒度锁、硬件事物性内存或原子化来实现。无论选择哪种处理方式，我们都可以预见这会使MPX的性能显著下降。
对于微架构层级，一种解决方案是可以对`mov-bndldx/bndstx`进行合并来保证他们执行的原子性。指令解码器在探测到`bndldx`后，在指令序列中找到对应的`mov`操作，并且规定其余的操作都按照这样的逻辑执行。但是我们认为这样的解决方案需要对CPU的前端进行修改。另外，这也会限制编译器可以进行优化的潜力。

## Measurement Study

在本章中我们回答以下问题：

* MPX的性能缺陷是多少？、
  * 程序会变的多慢？
  * 内存消耗会怎么变化？
  * MPX保护如何影响多线程应用？
* MPX提供什么层级的安全性保证？
* MPX在应用中存在什么样的可用性问题？

### Experimental Setup

所有的实验环境使用Fex测试框架搭建，并且对所需的编译类型、测试工具以及某些实验步骤需要进行一定的修改。
**实验平台.** 所有的实验都是在以下的配置上运行

* _硬件：_
  * Intel Xeon CPU E3-1230 v5 @3.40GHz
  * 1套接口，8个超线程，4物理内核
  * CPU缓存：L1d = 32KB, L1i = 32KB, L2 = 256KB, 共享L3 = 8MB
  * 64GB内存
* _网络：_ 为了进行验证性实验，我们使用的两台机器之间的网络带宽使用iperf测得为938Mbits/sec
* _软件配置：_
  * 内核：4.4.0
  * GLibC：2.21
  * Binutils：2.26.1
* _编译器：_

 GCC 6.1.0，使用以下配置

```bash
--enable-languages=c,c++ --enable-libmpx --enable-multilib --with-system-zlib
```

 ICC 17.0.0
 Clang/LLVM 3.8.0（AddressSanitizer），使用以下配置

```bash
-G "Unix Makefiles"
-DCMAKE_BUILD_TYPE = "Release"
-DLLVM_TARGETS_TO_BUILD = "X86"
```

 Clang/LLVM 3.2.0（SAFECode），使用以下配置

```bash
-G "Unix Makefiles"
-DCMAKE_BUILD_TYPE = "Release"
-DLLVM_TARGETS_TO_BUILD = "X86"
```

 Clang/LLVM 3.4.0（SoftBound），使用以下配置

```bash
--enable-optimized --disable-bindings
```

**测试工具.** 我们使用下述工具进行测试：
_perf stat._ 是我们用于检测所有CPU相关的参数的工具，完整的检测参数列表包括：

```bash
-e cycles,instructions,instructions:u,instructions-k
-e branch-instructions,branch-misses
-e dTLB-loads,dTLB-load-misses
-e dTLB-stores,dTLB-store-misses
-e L1-dcache-loads,L1-dcache-load-misses
-e L1-dcache-stores,L1-dcache-store-misses
-e LLC-loads,LLC-load-misses
-e LLC-store-misses,LLC-stores
```

_时间._ 由于perf不提供测量进程物理内存消耗的功能，我们使用`-verbose`来收集常驻内存模块的最大大小
_Intel Pin._ 用于采集MPX的插桩信息。
**测试集.** 在我们的测试中， 我们使用三套测试基准：PARSEC 3.0，Phoenix 2.0以及SPEC CPU2006。为了修复一些已经发现的bug，我们对SPEC测试套打了一个补丁，并且在我们的测试中，我们也对这三套测试套件修复了一定的bug。
我们将这些测试套件连同他们的依赖一同进行编译。
**编译选项.**
_GCC-MPX_
编译器选项：

```bash
-fcheck-pointer-bounds -mmpx
```

连接器选项：

```bash
-lmpx -lmpxwrappers
```

环境变量：

```bash
CHKP_RT_BNDPRESERVE = "0"
CHKP_RT_MODE = "stop"
CHKP_RT_VERBOSE = "0"
CHKP_RT_PRINT_SUMMARY = "0"
```

子选项：
禁止边界压缩：

```bash
-fno-chkp-narrow-bounds
```

只保护内存写，不保护内存读：

```bash
-fno-chkp-check-read
```

_ICC-MPX_
编译器选项：

```bash
-check-pointers-mpx=rw
```

连接器选项：

```bash
-lmpx
```

环境变量：

```bash
CHKP_RT_BNDPRESERVE = "0"
CHKP_RT_MODE = "stop"
CHKP_RT_VERBOSE = "0"
CHKP_RT_PRINT_SUMMARY = "0"
```

子选项：
禁止边界压缩：

```bash
-no-check-pointers-narrowing
```

只保护内存写，不保护内存读：使用

```bash
-check-pointers-mpx=write
```

替换掉原有的CFLAG：`-check-pointers-mpx=rw`
_AddressSanitizer（GCC和 Clang）_
编译选项：

```bash
-fsanitize=address
```

环境变量：

```bash
ASAN_OPTIONS="verbosity =0:\
detect_leaks=false:\
print_summary=true:\
halt_on_error=true:\
poison_heap=true:\
alloc_dealloc_mismatch=0:\
new_delete_type_mismatch=0"
```

子选项：
只保护内存写，不保护内存读：

```bash
--param asan-instrument-reads=0
```

_SoftBound_
编译器选项：

```bash
-fsoftboundcets -flto -fno-vectorize
```

连接器选项：

```bash
-lm -lrt
```

_SAFECode_
编译器选项：

```bash
-fmemsafety -g
-fmemsafety -terminate -stack-protector=1
```

**实验选项.** 每个程序都执行十次，结果取算算术平均值。对同一测试套里面的不同测试程序的平均值使用几何平均数。三套测试套的最终的平均值也使用几何平均数取得。
我们进行以下形式的实验：

* 正常情况：对一个单线程使用定值输入
* 多线程：进行2, 4, 8个线程程序的实验
* 变量输入：使用大小逐级递增的输入（5次实验，每次实验的输入大小是上一次实验输入大小的2倍）

实验结果使用以下原则进行评判：

* 应用成功编译
* 应用正常执行（以返回码0退出）
* 对于可确定的程序，对于同样的输入，与未使用MPX保护的程序有同样的输出

在Table5中展示了差异系数（~~离散系数~~）

### Performance

为了评估MPX带来的开销，我们选用了三套测试套件：Phoenix 2.0， PARSEC 3.0和SPEC CPU2006，我们不止评估了ICC和GCC下的MPX实现，同时也对AddressSanitizer，SAFECode和SoftBound进行了评估。
**运行开销.** 首先，我们注意到ICC-MPX的运行开销显著优于GCC-MPX。但是同时，ICC的可用性也更差：在38个测试程序中，只有30个程序（79%）被正确编译和执行，而在GCC的MPX环境下，有33个程序（87%）被正确编译和执行。
尽管AddressSanitizer是一个基于软件的实现，但是它同ICC-MPX的性能相当，比GCC-MPX更为优秀。这种预期之外的评估结果预示着由硬件带来的性能优势被MPX自身的复杂设计和低效率的指令所吞噬殆尽。尽管MPX的安全性比AddressSanitizer更高。
在Phoenix测试套件环境下，SAFECode和SoftBound的表现优秀，但是在PARSEC和SPEC的环境下，他们的性能和可用性都很差。首先，我们考虑在Phoenix下的SAFECode：由于Phoenix测试程序的设计简单，指针较少，SAFECode在此时的性能开销约5%。但是在PARSEC和SPEC测试套件环境下，SAFECode只能正确执行31个测试程序中的18个（58%），并且产生了很高的性能开销。SoftBound在PARSEC和SPEC下只能执行7个测试程序（23%）。并且，在测试中，SAFECode和SoftBound的性能并不稳定，在有些测试程序的性能开销有超过20倍。
**插桩开销.** 在多数情况下，性能开销只受到一个因素的影响，就是被保护程序增加的指令数量。如果我们分析Figure 9和Figure 10，我们就可以得出这一结论。

![Figure 9](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%209.png)

![Figure 10](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2010.png)

如同我们预计的那样，MPX由于其硬件实现，大大减少了指令数量（ICC-MPX的指令数量相比AddressSanitizer减少了约70%）。因此，一旦MPX的指令性能得到提升，MPX的性能开销就会大大降低。
MPX的性能开销的一部分来源于对BT的管理。我们的微基准测试结果显示，在最差的情况下，可能会带来100%以上的时间开销。但是，我们在现实应用程序中没有发现这样的问题。即使对于一些会创建数百个BT的应用程序，相比于其他因素，对BT的管理带来的时间开销几乎可以忽略不计。
**IPC.** 很多程序没有完全利用CPU的执行单元资源。例如，理论上我们机器的IPC约为5，但是许多程序只能得到约1-2的实际执行IPC（见Figure 11）。因此，进程安全技术受益于没有完全使用的CPU执行单元，并且部分掩盖了他们的性能开销。

![Figure 11](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2011.png)

但是我们最重要的发现就是MPX没有提升程序的IPC。我们的微基准测试显示这是由于MPX的边界检查指令对于一个执行端口的竞争导致。如果这个功能能够在更多端口上执行，MPX就可以并行执行指令，从而降低MPX的性能开销。
与此同时，基于软件的解决方案，尤其是AddressSanitizer和SoftBound显著提升了IPC，这也部分掩盖了他们的性能开销。
**缓存使用.** 一些应用对内存的消耗很大，并且会对CPU的缓存系统造成很大的压力。如果原始程序有很多L1或LLC缓存命中失败，在这种情况下，系统的内存管理系统就成为了程序性能的瓶颈。此时，软件安全策略就能够部分掩盖它们的性能开销。
Figure 12中使用ICC-MPX编译的 _wordcnt_ 就可以很好的说明这种情况。使用MPX保护后的指令数量为原始数量的4倍，IPC接近原始程序，并且使用了很多高开销的指令`bndldx`和`bndstx`。但是它的性能开销只有3倍。这就是因为原始版本的 _wordcnt_ 有大量的缓存命中失败。这些缓存命中失败带来了高额的性能开销并且部分掩盖了ICC-MPX带来的时间开销。

![Figure 12](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2012.png)

**MPX指令.** 对于Intel MPX而言，影响性能的最重要的因素之一是在插桩时使用的指令。特别的，存储`bndstx`和读取`bndldx`指令需要二级地址转化，这些指令的时间开销很高，并且也会破坏缓存的局部性。为了证明这一点，我们分析了MPX指令在整个程序中所占的比例（Figure 13）

![Figure 13](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2013.png)

如同我们预计的那样，MPX指令的大部分都是用于边界检查的`bndcl`和`bndcu`。除此之外，许多程序需要使用`bndmov`指令将边界值从一个寄存器移动到另一个寄存器（`bndmovreg`），或者将边界值从寄存器存储到栈上（`bndmovmem`）。最后，指针操作密集的程序需要使用高性能开销的指令`bndstx`和`bndldx`来对BT进行存/取操作。
`bndstx`和`bndldx`指令所占的比例同MPX的性能开销具有很强的相关性。例如，在ICC-MPX保护下的 _matrixmul_ 只添加了边界检查指令，因此，我们可以很容易的分析得到指令数量和性能开销之间的关系。但是在GCC-MPX保护下的版本的优化就没有这么彻底，还存在着一些`bndldx`指令，这也使得GCC-MPX版本的性能开销显著更高。
ICC-MPX版本的 _wordcnt_ 中，`bndldx/bndstx`指令的比例非常高。这是由于ICC的libchkp库中对于`memcpy`函数的简化算法的bug引起的。
**内存消耗.** 在某些情况下，内存消耗（更具体的讲，是常驻内存模块的消耗）可能是一项比较重要的评估因素。例如对于数据中心服务器上运行的需要协助定位并频繁进行迁移的程序而言就是如此。Figure 14展示了对于内存消耗的评估结果。

![Figure 14](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2014.png)

平均而言，ICC版本的MPX有2.1倍的内存消耗，GCC版本的MPX有1.9倍的内存消耗。相比于AddressSanitizer的2.8倍已经是一个不小的提升。这主要是由以下三点带来的：

1. AddressSanitizer通过为每个目标添加一个包围该目标的“redzone”，改变了内存分布
2. AddressSanitizer在主内存空间维护了一个“shadow zone”，并且“shadow zone”的大小随着程序工作集大小线性增加
3. AddressSanitizer有一个“隔离”的特性来限制对已释放内存的复用

相比而言，MPX仅为指针边界的元数据分配了内存空间，并且还有一个中间层BD实现了在长时间运行时的低内存开销的元数据管理功能。有趣的是，SAFECode通过对pool分配内存，它的内存消耗更低，但更低的内存开销并没有给它带来更好的性能。
**其余MPX特性的影响.** Intel MPX的两项特性同时影响到了性能和安全性。_边界压缩_ 提升了安全性等级，但是一定程度上降低了性能；_仅对写操作进行保护_ 通过禁用对内存读的保护提升了性能。
这些特性的对比在Figure 15和Figure 16中进行展示。边界压缩由于不影响检查的数目，因此对性能的影响很低。但与此同时，它会略微增加内存消耗因为需要存储更多的边界信息。仅对写操作进行保护对程序的影响相反：插入更少的代码降低了性能开销，但是对内存消耗的影响很低。

![Figure 15](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2015.png)

![Figure 16](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2016.png)

多线程.** 为了评估多线程的影响，我们评估所有测试套在2个和8个线程执行下的运行时间（见Figure 17）。注意仅有Phoenix和PARSEC是多线程的。同样的，SoftBound和SAFECode也不支持多线程，因此我们将他们移出本次评估的讨论。

![Figure 17](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2017.png)

如Figure 17中展示的那样，可扩展性的区别是很小的。对于MPX而言，这主要是由于多线程支持的缺失引起的，也就意味着对于多线程版本，程序没有执行额外的代码。对于AddressSanitizer而言，由于设计上对多线程进行了考虑，因此也不需要进行显式的同步。
对 _linearreg_ 和 _wordcnt_ 的GCC-MPX实现，8线程实现相比2线程更慢，这是由于BT入口的高速缓存线路共享引起的。
对于 _swaptions_ ，AddressSanitizer和MPX的实现都比原始程序显著更慢。这是由于相比于原始程序，剩余的IPC资源不足以同时支持8个线程（超线程带来的问题）。类似的，对于 _streamcluster_ ，MPX保护后的版本的性能也显著比原始版本和AddressSanitizer保护下的版本更慢，这也是由于超线程带来的问题：MPX指令在8线程的情况下饱和占用了IPC资源，因此使得程序比原始版本的可扩展性更差。
**可变输入大小.** 不同的输入大小（工作集）可能导致不同的缓存行为，进而引起性能开销的改变。为了观测这种影响，我们以小、中、大三种输入运行不同的测试集，但是测试结果并没有什么值得讨论的，通用的结论是输入的大小对性能开销的影响可以忽略不计。

### Security

**RIPE测试平台.** 我们在RIPE安全测试平台上进行我们的安全性测试。RIPE是一个不断尝试攻击自己的C程序，可以实现在栈、堆、数据段、BSS段进行缓冲区溢出。RIPE可以模拟超过850种不同的攻击，其中包括shellcode，return-into-libc和ROP。在我们的评估中，我们的安全环境更为宽松，我们禁用了ASLR，stack canaries，fortify-source（~~一种源码检查标志~~）并且允许了栈执行，在GCC下，只有64次攻击可以绕过编译器的检查，而在ICC下只有34次攻击绕过编译器的检查，在Clang下，只有38次攻击绕过编译器的检查。
最后结果在Table 6中进行展示。默认版本的GCC-MPX的表现很糟，41次攻击成功执行（64%)，这也就意味着，默认的GCC-MPX的编译选项并不完善。我们在对`memcpy`的打包函数中找到了一个bug，这个bug会使所有的边界寄存器清零，这也就意味着对`memcpy`的边界检查都变得没有意义。这个bug在环境变量`BNDPRESERVE`被手动置为1的时候消失。另外，GCC-MPX并没有对数据结构的第一个域默认进行边界压缩，这比ICC采用的策略更为宽松。为了检测到在第一个域内发生的对象内溢出，我们需要为GCC设定`-fchkp-first-field-has-own-bounds`编译选项。当我们添加了这两个编译选项的时候，所有的攻击都被阻止了，表格中其余的测试项都添加了这两项编译选项。

| 测试方式               | 可用的攻击数 | 备注                         |
| ---------------------- | ------------ | ---------------------------- |
| MPX（GCC）默认配置     | 41/64        | 均为`memcpy`引起或对象内溢出 |
| MPX（GCC）             | 0/64         |                              |
| MPX（GCC）禁用边界压缩 | 14/64        | 均为对象内溢出               |
| MPX（ICC）             | 0/34         |                              |
| MPX（ICC）             | 14/34        | 均为对象内溢出               |
| AddressSanitizer(GCC)  | 12/64        | 均为对象内溢出               |
| SoftBound（Clang）     | 14/38        | 均为对象内溢出               |
| SAFECode（Clang）      | 14/38        | 均为对象内溢出               |

Table 6：RIPE安全测试集的测试结果
其他的测试结果同我们预期的结果相同。对于没有进行边界压缩的MPX保护，有14次对象内溢出，意味着一个可被攻击的缓冲区同被攻击对象在同一数据结构内，在AddressSanitizer，SoftBound和SAFECode这种也同样出现了这种攻击。但是又去的是，在AddressSanitizer中只出现了12次对象内溢出，比其它防护方案少了两次，尽管没有深入研究，但是AddressSanitizer可以阻止堆上的对象内溢出。
我们使用同样的评估方法评估了仅对写保护的MPX版本，而评估结果完全相同。这是由于RIPE仅尝试进行控制流劫持，而没有进行信息泄露攻击。
**其它已知bug.** 在我们的试验中，我们发现了六个可以指针越界的bug，其中5个是已知的，还有一个可以被GCC-MPX见测但是在之前并没有提出。
这些bug为：

1. 在 _ferret_ 中不正确的黑白图像输入会造成缓冲区溢出
2. 在 _h264ref_ 中错误的前置递增声明导致差位错误
3. 在 _perlbench_ 中的越界写操作
4. 在 _x264_ 中的正常的对象内缓冲区越界写操作
5. 在 _h264ref_ 中正常的对象内缓冲区越界读操作
6. 在 _perlbench_ 中对象内缓冲区越界写操作

在启用边界压缩的GCC-MPX中，这些bug都可以被检测到。可以预测的是，所有三个对象内bug和一个只读bug在被禁用了边界压缩功能和只对写操作进行保护的MPX实现中都不会被检测到。ICC-MPX仅检测到了这其中的三个bug：在其它情况下，程序由于MPX的原因无法执行。此时有一个有意思的现象：包含bug的程序也是在MPX下崩溃最为频繁的程序。
同我们期待的一致，AddressSanitizer只能发现其中的三个bug，它的检测粒度为单个对象，而不能对对象内的缓冲区溢出进行检测。SAFECode可以检测bug 2和3，对于其他的bug，要么是由于粒度问题无法检测，要么是程序无法被SAFECode编译。SoftBound无法检测bug 2，并且不支持多线程的 _ferret_ 和 _x264_ ，_perlbench_ 也无法正确运行。

### Usability

如我们在之前讨论的那样，一些程序在MPX下崩溃是因为他们使用了不支持的C语言风格或完全破坏了C语言的标准。但是也存在一些程序因为编译器的MPX pass的内在bug导致无法编译或运行（GCC下有一种情况，ICC下有8种情况）。
Figure 18展示了MPX的 _可用性_ ，如无法正确编译的使用MPX保护的程序或需要对代码进行大量修改的情况。对于可以进行简单修复的情况（Table 4）我们不认为程序是不可用的。MPX的 _安全性等级_ 由我们自行分级并对应着更严格的保护规则。level 0意味着没有采用保护的原始程序，而level 6意味着保护最为严格的MPX配置。我们的评估包括来自Phoenix、PARSEC和SPEC测试条件的38个程序。

![Figure 18](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2018.png)

约10%的程序在应用MPX最弱的level 1的保护时就产生了崩溃（没有边界压缩并且仅进行写保护），在执行最高的等级的安全level 6（启用了`BNDPRESERVE`）时，多数程序会崩溃。
对于其它的防护方案，在AddressSanitizer下，没有程序发生崩溃。在SAFECode下，约70%的程序可以正常执行（Phoenix下的所有程序，PARSEC下的半数程序和SPEC下的四分之三的程序）。SoftBound只能运行简单的程序（Phoenix下的所有程序，PARSEC下的1个程序和SPEC下的6个程序）。
**遇到的问题.** Figure 19展示了我们在实验中遇到的问题。
AddressSanitizer没有可用性问题——在设计之初就没有对C标准和内存模型进行假设，同时这也是最为稳定的版本，在每个版本的GCC和Clang中都会进行修复和更新。

![Figure 19](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2019.png)

相比而言，SoftBound和SAFECode更像是进行科研的验证性版本，它们在Phoenix下的简单程序下可以正常工作，但是不能在更为复杂的测试集PARSEC和SPEC上正确的编译和运行。另外，SoftBound不支持多线程，所有的多线程程序都会立刻崩溃。
尽管GCC-MPX和ICC-MPX在level 6下（`BNDPRESERVE=1`）使多数程序发生了崩溃，但这是由于`BNDPRESERVE`没有为来源于或访问未被保护的库文件的指针清除边界信息导致的。这也就意味着任何来自传统库文件（包括C标准库）或被传统库文件所修改的指针边界一定是错误的，此时，89%的GCC-MPX和76%的ICC-MPX程序会发生崩溃。
注意在Phoenix测试套件下，GCC-MPX多数情况下无法正确运行，而ICC-MPX运行正常。这是由于对`libc`打包的细微差别导致的：所有在GCC-MPX下崩溃的程序都使用了`mmap64`函数，这个在GCC下被忽略的函数，在ICC下被正确打包。因此，在GCC下，由于函数未被打包，此时函数的指针边界为空，在`BNDPRESERVE=1`的条件下，就会被认为是指针越界。
一些程序在C标准库的互用性被破坏的情况下依旧可以正常工作，这是因为一些程序如`kmeans`, `pca`和`lbm`几乎不需要除`malloc`, `memset`以及`free`等之外的外部函数，而这些函数由已打包的MPX库文件进行提供。
一些程序因为 _破坏了内存模型_ 而崩溃：

* _ferret_ 和 _raytrace_ 都存在使用一个数据结构的第一个域访问其他域的行为（一种不符合C标准的通用做法）。在启用了边界压缩的情况下，ICC-MPX禁止这种行为，而GCC-MPX默认情况下允许这种行为，并且提供了一个编译选项`-fno-chkp-first-field-has-own-bounds`用于对这种情况下的边界检查，我们认为启用了这种选项的MPX保护等级为 level 5.
* gcc有一套自己标准的复杂内存模型，可以进行位操作，类型转换以及一些其它被C标准弃用的行为。
* _soplex_ 可以手动将指向对象的指针从一个地址使用指针算数运算指向另一个地址，而没有对指针边界进行任何保护。在设计上，MPX不会允许这种破坏C标准的行为（同样的情形也发生在了 _mcf_ 中，不过只是一种测试输入的特殊情况）
* _xalancbmk_ 从数据结构的基部开始进行~~容器样式的差集~~（a container-style subtraction from the base of a struct)。这使得GCC-MPX和ICC-MPX在启用边界压缩时产生崩溃
* 我们手动修复了一些违背内存模型的情况，如以1为单位的可变数组（`arr[1]`）

在以下情形，可以检测到存在bug：

* 我们检测并修复了存在于 _ferret_,  _h264ref_ 以及 _perlbench_ 中的3个bug
* 存在于 _ferret_, _h264ref_ 以及 _perlbench_ 中的3个bug只能被GCC-MPX检测到，ICC-MPX没有检测到 _h264ref_ 和 _perlbench_ 中的bug。在调试过程中，我们注意到ICC-MPX对于边界压缩的控制没有GCC-MPX严格，因此漏过了bug。我们认为可能是由于GCC和ICC编译器的不同内存分布导致的。

在很罕见的情况下，我们遇到了GCC和ICC的编译器的bug

* GCC-MPX只有一个bug，存在于只进行写保护的 _xalancbmk_ 的“fatal internal GCC compiler error”
* ICC-MPX在保护 _vips_, _gobmk_ , _h264ref_ 和 _milc_ 的数个版本的时候会遇到自动向量化bug
* ICC-MPX在保护 _x264_ 和 _xalancbmk_ 的数个版本的时候会遇到“wrong-bounds through indirect call”
* ICC-MPX在保护 _dealII_ 时有一个我们无法识别的bug
* 我们手动修复了ICC-MPX的C99 VLA显示bug

## Case Studies

为了了解MPX对现实世界应用的影响，我们进行三项个例研究：Apache和Nginx web服务器以及Memcached内存管理系统。类似于上一届，我们从性能和内存开销，安全性以及可用性三个维度来进行评估。
我们将初始版本的程序同使用默认编译选项的GCC以及ICC下的MPX实现保护的程序进行比较，同时，我们也将AddressSanitizer加入我们的对比。我们在个例研究中不 sSanitizer的执行结果。
所有的实验环境都通之前的实验相同。其中一台机器作为服务器端而另一台作为客户端，它们之间使用一条1GB的以太网光纤进行连接，实际带宽为938Mbit/sec。我们在进行后续测试时，配置同时使用服务器的8个内核，其他的配置参数保持默认值。
所有三个程序都同他们的独立库文件静态连接。选择静态连接便于测量每个程序的整体开销。

### Apache Web Server

我们选择连接了OpenSSL 1.0.1f的Apache version 2.4.18进行测试。我们选取的OpenSSL版本收到Heartbleed的影响，Heartbleed的信息泄露可以使攻击者得到明文的敏感信息如密钥和用户密码。由于AddressSanitizer和MPX都不支持内联汇编，我们在Apache编译时禁用了这一点。为了充分利用服务器，我们选择了Apache的MPM事件模型的默认配置。
我们在客户端机器上运行ab测试集进行工作载荷收集。在使能了KeepAlive特性后，测试集不断通过HTTP连接接受一个静态的2.3K大小的网页。我们通过增加在单一时间点的请求数量来增加工作载荷。
但是在对Heartbleed进行测试时，我们发现ICC-MPX的实现在处理OpenSSL中的`x509_cb`函数时，遇到了一个英特尔的编译器bug，这个bug会导致Apache崩溃。由于这个bug仅在HTTPS连接下出现，我们依旧可以对ICC-MPX的性能进行评估。

![Figure 20](https://github.com/blankaiwang/blankaiwang.github.io/raw/master/_posts/2019-01-16-Intel-MPX-Explained.assets/Figure%2020.png)

**性能.** 在Figure 20a中,GCC-MPX, ICC-MPX和AddressSanitizer相比于原始的应用程序的性能开销都很低，分别为95.3%，95.7%和97.5%。在时延上的额外开销不超过5%，即在实际操作中，影响性能的是网络而不是CPU或内存。
考虑到内存开销（Table 7），AddressSanitizer有约3.5倍的额外开销，而MPX的额外开销约为12.8倍。这是由于Apache为每个客户端分配了一个额外的1MB大小的充满指针的数据信息，这导致大量BT的创建。

| 测试项           | Apache  | Nginx   | Memcached |
| ---------------- | ------- | ------- | --------- |
| 原始程序         | 9.4     | 4.3     | 73        |
| MPX              | **120** | 18      | **352**   |
| AddressSanitizer | 33      | **380** | 95        |

Table 7：在峰值吞吐量下的内存占用情况（GCC-MPX和ICC-MPX的测试结果相当）
**安全性.** 我们使用Heartbleed验证安全性保护。简而言之，在服务器端收到恶意构造的TLS心跳包后就会触发Heartbleed漏洞。服务器没有对收到的信息头部的参数长度进行规则检查，而直接使用`memcpy`将进程内存信息进行复制并回复信息，这样攻击者就可以读取到敏感内存信息。
AddressSanitizer和GCC-MPX都可以检测到Heartbleed。

### Nginx Web Server

我们使用Nginx 1.4.0进行实验，这是最新的已知有栈缓冲区溢出漏洞的版本。Nginx使用“autodetected”的工作进程数配置方式加载所有内核，并且使用与Apache相同的ab测试集进行评估，在客户端同样使用ab测试集。
为了在GCC-MPX下成功运行Nginx，我们将`ngx_hash_elt_t`数据结构中的变量长度数组`name[1]`改为`name[0]`。但是启用了边界压缩的ICC-MPX依旧无法正确运行，会在`ngx_http_merge_locations`函数处进行误报。产生这个问题的原因在于从一个较小变量转化到较大变量的过程中，边界没有随之变化。而在GCC-MPX中由于默认使用数据结构中的第一个域继承对象的边界，就不会遇到这个bug。因此在剩余的评估中，使用未启用边界压缩的ICC-MPX。
**性能.** 从性能角度而言（Figure 20b），Nginx同Apache相似。AddressSanitizer可以达到初始吞吐量的95%，而GCC-MPX和ICC-MPX分别达到了86%和89.5%。同Apache类似，网络性能是整个试验性能的约束。原始程序的CPU使用率是225%，MPX的CPU使用率是265%，而AddressSanitizer的CPU使用率是300%（从CPU的使用率证明基于硬件的方法降低了CPU的开销）。
Nginx的性能走势只在GCC下成立。Nginx的原始程序在ICC下只有GCC吞吐量的85%，Clang下只有GCC吞吐量的90%。更令人吃惊的是，ICC-MPX的性能比原始ICC有5%的 _性能提升_ 。类似的，使用AddressSanitizer的Clang比原始版本的Clang有10%的 _性能提升_ 。
对于内存开销（Table 7），情况与Apache相反：MPX的内存开销约为4.2倍，而AddressSanitizer的内存开销为88倍之多（同时还有625倍的页错误和13%的LLC缓存命中失败）。这是由于AddressSanitizer的“隔离”特性引起的。此外，AddressSanitizer将使用过的内存为“poisoned”，并向OS请求新的内存块（这可以解释大量页错误的原因）。初始版本的Nginx不断进行内存回收，而AddressSanitizer则带来了大量的内存消耗，当我们禁用了隔离特性后，AddressSanitizer仅占用24MB内存。
但是隔离性的问题并不会影响性能。首先，Nginx的性能开销的约束在于网络性能，主机有足够的资源掩盖内存占用问题；其次，分配新的内存空间的高额开销掩盖了向OS请求新的内存块的开销。
**安全性.** 为了评估安全性，我们选取了可以被用于构建ROP攻击的缓冲区溢出漏洞CVE-2013-2028。此时，一个恶意构造的HTTP请求使Nginx错误的将一个有符号整数识别为一个无符号整数，接下来，将可以导致发生缓冲区溢出的参数大小传给`recv`函数，从而触发这个bug。
令人吃惊的是，AddressSanitizer可以检测到这个bug，而MPX _不能_ 检测到这个bug。产生这个问题的原因在于运行打包库文件：AddressSanitizer打包了所有的C库函数，包括`recv`，因此检测到了这个bug。而在GCC-MPX和ICC-MPX中，只有最为广泛应用的函数被打包并且进行了边界检查，如`memcpy`和`strlen`。这也使得`recv`的缓冲区溢出没有被MPX检测到。
这也使得进行完整保护的重要性凸显。不止保护程序的自有代码，还要对所有程序使用到的未被保护的库文件打包进行保护。另一点是这个漏洞是只读漏洞，不能被仅写保护的保护策略所检测。

### Memcached Caching System

最后，我们对Memcached缓存管理系统version 1.4.15进行测试。这是受到DDoS攻击的最新版本。在我们的试验中，Memcached使用8线程的方式运行以充分利用服务器资源。在客户端我们使用来自libmemcached的memaslap测试基准，并使用默认配置（90%的读操作的大小平均为1700B，10%的写操作的平均大小为400B）。我们通过增加并发数增加负载。
经过在Nginx和Apache上的调试工作，我们高兴的发现Memcached在使用GCC-MPX和ICC-MPX时没有产生任何问题。
**性能.** 从性能上分析，Memcached上的MPX表现是最差的（Figure 20c），AddressSanitizer和原始版本的Memcached的吞吐量相当，而GCC-MPX和ICC-MPX都只能达到最大吞吐量的48%-50%。
在原始版本和AddressSanitizer版本的Memcached，性能约束在于网络性能，但这并不适用于MPX：Memcached只占用了70%的网络带宽。Table 7帮助我们理解MPX下的Memcached的性能约束。AddressSanitizer只产生了30%的额外内存开销，而MPX占用了350MB的内存（原始版本的4.8倍）。高额的内存消耗破坏了缓存序列并且带来5.4倍的页错误和10%-15%的LLC缓存未命中，使得MPX版本的Memcached的性能约束在于内存（事实上，CPU的使用率从未超过320%）。
**安全性.** 在安全性评估上，我们使用CVE-2011-4971进行评估。在这个拒绝服务攻击漏洞中，服务器端收到一个特殊构造的数据包并使用处理程序`conn_nread`尝试将数据包中的内容使用`memmove`函数复制到一个缓冲区中。但是由于在参数中存在整型符号错误，`memmove`试图对Gbyte级别的数据进行复制并引发段错误。所有的防护方案都可以检测到这个缓冲区溢出并终止程序的执行。