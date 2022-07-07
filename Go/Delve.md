#! https://zhuanlan.zhihu.com/p/538725747
# Delve的内部架构与实现

## 前言
本文内容系本人基于Alessandro Arzilli在GopherCon EU 2018上的talk整理加工而成，转载请注明出处。原视频及ppt链接在文章末尾。
### 目的
- 对delve的架构做一个总体概述
- 解释为什么其他debugger在调试Go程序时会遇到困难

### 什么是Delve？
Delve是一款专用于Go编程语言的debugger，旨在为Go提供一个简单、功能齐全的调试工具。

github主页：https://github.com/derekparker/delve

### 为什么会有Delve？
这就要谈到Go语言的底层实现了。简单来说，有两点：

1. Go语言中的堆栈概念与我们平时理解的C/C++那种“程序堆栈”的概念不太一样。传统意义上的“栈”被Go语言的运行时消耗掉了，用于调度器、GC等；而对于用户代码而言，它们所消耗的“堆”和“栈”实际上都是Go运行时向OS申请的堆内存！这也就意味着，Go语言在程序逻辑上的“栈”是可以被移动的，也正是因为这一特点，对普通指针所作的算术运算不能奏效，除非你使用unsafe包。

2. 由于Go的“栈”是动态的，goroutine可以在创建的时候只使用很小的一块内存空间（默认为2KB），这也是创建goroutine的代价比创建线程要小的原因。而当goroutine的栈已经不足以储存它所需要存储的数据时，就会进行扩容，如果当前栈的下一块内存空间已经被占用了，那么整个栈将会被拷贝到内存的另一块位置。

传统的调试工具如gdb，是基于“栈不会在内存中发生移动”这一假设而设计的，因此它们无法正确地处理Go语言中栈发生移动的情形，Go语言需要一款自己的调试工具，Delve也就应运而生了。

## 架构
Go语言程序是编译成汇编代码再去执行的，而汇编指令需要进行大量的寄存器访问，debugger的工作主要关注两个特殊的寄存器：程序计数器(PC)和堆栈指针(SP)。

Delve的整体架构如下，分为三层：

- UI Layer，直接与用户交互；
- Symbolic Layer，拥有代码行号、数据类型、变量名等信息；
- Target Layer，控制目标进程，不关心用户源码。

![Image](https://pic4.zhimg.com/80/v2-eeadea258b7d84c967248d3b8c46f0a9.png)


### Target Layer

下面首先详细介绍一下Target Layer的功能：

- 附加到目标进程或与目标进程（也就是正在debug的进程）脱离
- 枚举目标进程中的线程
- 可以启动/停止单个线程（或整个进程）。
- 接收“调试事件(debug event)”（线程的创建/死亡，最重要的是线程在断点上的停止）。
- 可以读/写目标进程的内存
- 可以读/写一个停止的线程的CPU寄存器

### Symbolic Layer

编译器会向可执行文件中写入一些用于调试的信息，我们称为debug symbol，Symbolic Layer所做的事情就是从可执行文件中读取这些符号。Go语言采用的debug symbol规范是DWARFv4(2018年)，我们可以看看DWARF有哪些section:

![Image](https://pic4.zhimg.com/80/v2-a211aa1f141426550b0a61ad18d4fc0e.png)

emmmm看起来也不少，不过好在Delve只需要关心其中的三种：

![Image](https://pic4.zhimg.com/80/v2-4caf2dd5dc083227a087c4e02f644c19.png)

- debug_line：这是一个表，它将指令地址映射到文件:行号。
- debug_frame：堆栈解压信息。
- debug_info：描述程序中的所有函数、类型和变量。

debug_line太过简单，一看就明白什么意思，debug_frame实现比较复杂，Alessandro Arzilli在大会上也未作说明，这里我们重点关注一下debug_info。

以下面这段代码为例：
```go
package main 

type Point struct {
    X, Y int 
} 

func NewPoint(x, y int) Point {
    p := Point{ x, y }
    return p 
}
```

它的debug_info长这样：
![Image](https://pic4.zhimg.com/80/v2-ed15a5ab7af5ee510efc2352a8d9a26d.png)
NewPoint函数作为一个Subprogram节点，它拥有三个子节点，其中两个是形参，一个是局部变量，变量均通过一个reference指向具体的类型，对于结构体类型而言，它又通过child指向每个成员。

那么，Delve是如何打印出下面这样的堆栈信息的呢？

![Image](https://pic4.zhimg.com/80/v2-558f8175907315bc3c9a8e84a39947f0.png)

上面出现的信息一共有三种：
- 指令地址
- 函数名
- 文件:行号

Delve通过debug_info找到函数名，通过debug_line找到某条指令对应的代码在哪个文件的哪一行，这都很好理解，稍微复杂一点的是debug_frame获取指令地址的过程：
![Image](https://pic4.zhimg.com/80/v2-1a7ea6c23d69493114c64428e78f5a7b.png)

开始时，$PC_0$和$SP_0$分别取PC寄存器和SP寄存器的值，然后通过在debug_frame中查找$PC_i$，获取对应的栈帧大小，再由此计算出$PC_{i+1}$和$SP_{i+1}$的值，以此类推…

### Delve的真实架构
之前所说的三层模型其实是一种更为简单的抽象，在Delve的实际实现中，UI层与后两层是分离的，通过一个Service层进行交互，如下图所示。
![Image](https://pic4.zhimg.com/80/v2-b4b702322cb07b3e1fd8d4f79a95a4b7.png)

Delve这么设计的目的在于用户能够更方便地定制自己想要的UI，已经比较成熟的UI如下：
![Image](https://pic4.zhimg.com/80/v2-b9284709f5a87a44c4db9c483ef267eb.png)

## 一些Delve功能的具体实现
### Variable Evaluation

当我们在delve中输入`print a`命令，UI层将其翻译成`EvalExpression(”a”)`，由Symbolic层通过debug_info确定变量a的地址和大小（int是8个字节），再由Target Layer读取指定内存地址的内容。
![Image](https://pic4.zhimg.com/80/v2-454615c910c4b21dd5371f03a657d88a.png)
Target层将读取的8个字节向上传递至Symbolic层，为其添加地址、变量名、类型信息以后，再交给UI层，最终将`a = int(1)`返回给用户。
![Image](https://pic4.zhimg.com/80/v2-8233e07c70ef461a055c7e832e36fe72.png)

因为Delve是专为Go语言设计的，在打印变量信息方面，具有gdb所不具备的优势，对比如下：
![Image](https://pic4.zhimg.com/80/v2-af34349900a6838ef22aa5d2bf6f2194.png)

从上图可以看到，对于error接口类型和channel类型，gdb只能打印出变量的地址，而delve能够打印出对用户更友好的信息。

### Creating Breakpoints
![Image](https://pic4.zhimg.com/80/v2-476f10e021dc5568e5f314fcb2dd846b.png)
当用户输入`break main.f`时，UI层将其翻译为`SetBreakpoint(FunctionName: “main.f”)`，Symbolic层在debug_info中查找`main.f`的地址，再由Target层向该地址写入断点。Target层写入断点的操作实际上是将原先处于这一地址的指令覆盖掉，替换成一条新的指令：当执行该指令时，线程停止运行并让OS通知debugger（debug event）。

在给程序打断点这件事上，delve也比gdb更为优秀，这同样与Go语言的设计有关：
![Image](https://pic4.zhimg.com/80/v2-5998bd925e94495e89cd430659895ec4.png)

橙色部分代码的作用是在函数执行前进行判断，该函数是否需要更多的栈空间，如果需要，则调用`runtime.morestack`函数进行扩栈操作。`break main.f` 会在函数f的起始位置打断点，如果使用gdb的话，那么断点对应的汇编代码地址是`0x452eb0`，如果发生了扩栈，那么这个断点将会被hit两次，给用户的感觉就像是函数执行了两次一样。而delve针对Go语言的特点做了优化，它会越过前三行汇编指令，将断点打在`0x452ebf`处。

***— The End***

**码字不易，如果对你有帮助，动动手指点个赞吧！**

原视频链接：https://www.youtube.com/watch?v=IKnTr7Zms1k

ppt可在此获取：https://speakerdeck.com/aarzilli/internal-architecture-of-delve