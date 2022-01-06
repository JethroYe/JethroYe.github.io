---
layout: post
title: "Xcode 中编译选项的变化可能会帮你避免内存泄漏"
updated: 2021-12-31
tag: 
- 原创
comments: true
---

前段时间，在研究内存泄漏检测的的过程中，我写了一个测试case 用于测试内存泄漏。并且使用这个case 来研究OOMDetector [^1] 的机制和原理
```
//测试case
- (void)helloWorld { 
     void* pointer = malloc(500*1024*1024);
     memset(pointer, 255, 500*1024*1024);
 }
```
这个case 非常简单：在一个栈变量pointer 申请了500MB内存，并且将其涂写。
在iOS 中，申请并且使用过的内存就是dirty 的内存了，dirty内存是没办法被系统回收的。随着函数退出，栈变量pointer被回收，这500MB的内存将没有指针指向它，

所以，这块内存泄漏了。

我非常愉快的使用这个case 进行研究和调试。

首先，我将这段代码以源码的形式放在Demo里，在 Xcode Memory  中可以看到内存的上涨，并且 OOMDetector 工作正常，抓到了申请这块内存的堆栈。
接下来，我将这个函数放在一个framework 中，编译优化选项设置为 None，将framework以二进制的形式引入测试工程。这时，出现了有趣的事情：
1. 如果framework 的编译优化选项设置为 None，OOMDetector 可以抓到泄漏堆栈
2. 如果framework 的编译优化选项设置为O1，OOMDetector 什么都抓不到。。。

这时候，我十分惊讶了。难道OOMDetector 无法检查出 **O1** 优化下的二进制组件的内存泄漏？？？要知道几乎所有的iOS App 都会有二进制接入的组件，难道 OOMDetector 会犯这么简单的错误么？

> **此处补充一下OOMDetector 的工作原理**  
> 在iOS 中，内存申请无论OC 层面的alloc， C++ 中的new， new[] 等等都是通过libmalloc 这个库进行的。libmalloc 提供了一个回调函数 malloc_logger， 每次内存的申请/释放都会回调 malloc_logger  
> OOMDetector 中会记录每次申请/释放的内存地址。检查时会挂起全部线程，然后在遍历所有存活的内存地址看看是否在堆，栈，寄存器中有指针指向它。如果发现无人指向的地址，就认为它泄漏了。

此时我的注意力完全在二进制库上，难道被编译优化过的二进制库申请内存不会回调malloc_logger ？? 短暂的迷茫之后，我转念思考，是否 **O1** 优化条件下，压根没有发生内存泄漏？？

经过在Xcode 下的多次试验后，发现了O1 优化条件下，多次调用泄漏函数，内存并没有上涨。所以压根就没有申请/释放过内存。于是我修改了一下case。对比上面的case，新增了一个静态指针
```
//静态指针，指向申请到的内存
static void * testdasdas = NULL;

- (void)helloWorld {
    void* pointer = malloc(500*1024*1024);
    memset(pointer, 255, 500*1024*1024);
    testdasdas = pointer;
}
```

优化选项None 无需测试了，我们看看O1 下会发生什么
调用3次，内存上涨了1.5G, 检测到2个泄漏，符合预期。可以证明OOMDetector的可用性不会被二进制库的编译条件影响。
研究走到这里，就引出了两个问题：
1. **O1的优化选项究竟对我的代码做了什么？是简单的不给申请内存，还是从根本上优化掉了 helloworld 函数呢？**
2. **LLVM 的优化具体做了什么？**

LLVM的 代码过于庞大，直接刚源码不现实。于是我反向操作，先从编译后的汇编代码开始看起。

### 实验一
对源代码 foo.c 使用不同的优化方案生成汇编
```
//使用clang命令，用不同的优化等级生成汇编代码
clang -arch arm -O0 -S foo.c -o fooO0.s
clang -arch arm -O1 -S foo.c -o fooO1.s
```
foo.c 源码
```
int leakMemory() {
  char* pointer = malloc(500*1024);
  memset(pointer, 'F', 500*1024);
  return 0;
}

int main() {
  leakMemory();
  return 0;
}
```
结果对比
![](https://s2.loli.net/2022/01/03/wRtjPU78Xz1H9lD.png)

做完之后，第1个问题得到了解答。**O1 的优化选项优化掉了导致泄漏的函数的内部实现逻辑，泄漏函数是空的，刚好帮我兜住了内存泄漏。**

那么，为了搞明白LLVM的 O1优化和O0之前究竟有什么区别呢？我去看了下LLVM的源码[^2]。

经过简单的commend-f 搜索我找到了下面这些说明 
```
Code Generation Options
.. option:: -O0, -O1, -O2, -O3, -Ofast, -Os, -Oz, -Og, -O, -O4
  Specify which optimization level to use:

    :option:`-O0` Means "no optimization": this level compiles the fastest and
    generates the most debuggable code.
    -O0 : 完全不优化，编译出速度最快，完全可以调试的代码
             
    :option:`-O1` Somewhere between :option:`-O0` and :option:`-O2`.
    -O1: 在 -O0 和 -O2 中间的某个神秘地带。。
    
    :option:`-O2` Moderate level of optimization which enables most
    optimizations.
    -O2: 中等等级的优化，执行了大部分的优化选项

    :option:`-O3` Like :option:`-O2`, except that it enables optimizations that
    take longer to perform or that may generate larger code (in an attempt to
    make the program run faster).
    -O3: -O3`类似于-O2，不同之处在于它启用了其他优化功能：为了提高运行速度，可能会话费更长的编译时间或使程序更大

    :option:`-Ofast` Enables all the optimizations from :option:`-O3` along
    with other aggressive optimizations that may violate strict compliance with
    language standards.
    -OFast：启用O3 中所有的优化以及一些激进的优化策略，这些策略可能会破坏严格的语言层面的约定
    
    :option:`-Os` Like :option:`-O2` with extra optimizations to reduce code
    size.
    -OS: 像O2 一样，但是缩小了代码体积

    :option:`-Oz` Like :option:`-Os` (and thus :option:`-O2`), but reduces code
    size further.
    -Oz：像O2一样，进一步减小代码体积

    :option:`-Og` Like :option:`-O1`. In future versions, this option might
    disable different optimizations in order to improve debuggability.
    -Og：像O1一样，未来这个选项可能可以提高可调式能力

    :option:`-O` Equivalent to :option:`-O1`.
    和O1 一样

    :option:`-O4` and higher
      Currently equivalent to :option:`-O3`
      现在和O3 一样

```

我的问题出现在-O0，-O1 中间，所以，基本可以确定是某些编译优化生效了，导致内存泄漏被兜住了。

> **先来了解一下LLVM的编译流程**  
> 在iOS上，编译工具链使用的是大名鼎鼎的LLVM。LLVM和传统的编译器一样，也分前中后三个端。  
> **前端** 负责将各种各样的语言进行词法分析，语法分析，生成AST（抽象语法树），而后转换成为LLVM的中间语言 IR （intermediate representation）LLVM 中的前端编译器是clang。clang 是 LLVM 的子项目，是 C，C++ 和 Objective-C 编译器（Swift：clang-swift）  
> **中端** 负责对IR 进行处理，通过一系列的pass 对IR进行优化，类似于流水线的操作  
> **后端** 负责将优化好的IR，生成对应的机器码

![](https://s2.loli.net/2022/01/02/hc17LIPdW59vRyr.png)

LLVM的优点在于，中间表示IR代码编写良好，而且不同的前端语言最终都转换成同一种的IR。

![](https://s2.loli.net/2022/01/02/yzIYP5jKvFS4Tkp.png)

从流程上看，优化最有可能出现的地方就是 "**IR 优化**" 这个阶段了

先来看一个简单的例子了解一下IR语言长什么样子

### 实验2 
使用下面的示例代码，生成IR文件
```
unsigned add1(unsigned a, unsigned b) {
 return a+b;
}
unsigned add2(unsigned a, unsigned b) {
 if (a == 0) return b;
 return add2(a-1, b+1);
}
//命令 -- clang -emit-llvm -S IRSource.c -o IRSource.ll
//函数属性所表达的含义，可以在 https://llvm.org/docs/LangRef.html 中查到
```
生成的IR文件如下

```
; ModuleID = 'IRSource.c'
source_filename = "IRSource.c"
target datalayout = "e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx12.0.0"
; 上面是参与编译文件的基本信息

; Function Attrs: noinline nounwind optnone ssp uwtable （add1 函数）
define i32 @add1(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add i32 %5, %6
  ret i32 %7
}

; Function Attrs: noinline nounwind optnone ssp uwtable （add2 函数）
define i32 @add2(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i32, align 4
  store i32 %0, i32* %4, align 4
  store i32 %1, i32* %5, align 4
  %6 = load i32, i32* %4, align 4
  %7 = icmp eq i32 %6, 0
  br i1 %7, label %8, label %10

8:                                                ; preds = %2
  %9 = load i32, i32* %5, align 4
  store i32 %9, i32* %3, align 4
  br label %16

10:                                               ; preds = %2
  %11 = load i32, i32* %4, align 4
  %12 = sub i32 %11, 1
  %13 = load i32, i32* %5, align 4
  %14 = add i32 %13, 1
  %15 = call i32 @add2(i32 %12, i32 %14)
  store i32 %15, i32* %3, align 4
  br label %16

16:                                               ; preds = %10, %8
  %17 = load i32, i32* %3, align 4
  ret i32 %17
}

attributes #0 = { noinline nounwind optnone ssp uwtable "darwin-stkchk-strong-link" "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "probe-stack"="___chkstk_darwin" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+cx8,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0, !1, !2}
!llvm.ident = !{!3}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 12, i32 0]}
!1 = !{i32 1, !"wchar_size", i32 4}
!2 = !{i32 7, !"PIC Level", i32 2}
!3 = !{!"Apple clang version 13.0.0 (clang-1300.0.29.3)"}

```

### IR介绍
IR语言大致由下面四个部分构成
- Module: 理解为一个完整的编译单元，通常对应的就是一个源文件 .c
- Function: 函数，通常可以理解为一个函数定义或者函数声明
- BaseBlock: 基本块，表示一段没有控制流逻辑的基本流程
- Instruction: 指令

对于函数来说，通常使用 **define** 进行定义，使用 **declare** 进行声明。我们在一个编译单元（模块）下，可以使用别的模块的函数，这时候就需要在本模块先声明这个函数，才能保证编译时不出错，从而在链接时正确将声明的函数与别的模块下其定义进行链接。
一个函数会包含多个基本块，每个基本块又是由许多指令组成的。基本块只能从头部进入，从末尾退出。基本块的最后一条指令必须是终止指令，类似 ret br switch 等等。而每一个函数必须由一个入口块组成，这个入口块不能含有任何前置块

了解过IR之后，再来看LLVM中IR优化的流程。在LLVM 中，负责IR 优化的部分，叫做 **opt**。opt 中通过一个一个的 **Pass** ，构成了一个处理管道。每一个Pass代表了一个处理步骤。比如内联函数的替换, 表达式重写，循环优化。一个LLVM 优化器是由很多pass组成的**管道流（pipeline）**。
LLVM优化器提供了很多pass，它们拥有公共的父类，最后这些pass会被编译为动态库供LLVM使用。LLVM编译器使用这些库文件进行优化。这些pass之间是松耦合的，尽量不依赖其他的pass。
通过指定LLVM编译器优化用的参数，可以设定优化过程中要执行那些pass。 **-O0** 表示**不使用任何pass即没有优化**, **-O3** 使用 **67个passes**进行优化(LLVM 2.8)。

看到这里，就可以对上文中O0，O1，O2 有了更深的理解。**“不同的优化等级对应了LLVM opt 中pass的不同集合”**，针对问题2，已经猜出大概

> _“O1 比 O0多的某个pass，帮我优化了代码”_

为了验证猜想，我尝试用下面的两个指令生成不同优化等级下的IR

```
# 实验3. foo.c 使用不同的优化方案生成IR
clang -emit-llvm -S -O0 foo.c -o fooO0.ll
clang -emit-llvm -S -O1 foo.c -o fooO1.ll
```

实验源码：
```
#include <string.h>
#include <stdlib.h>
int leakMemory() {
  char* pointer = malloc(500*1024);
  memset(pointer, 'F', 500*1024);
  return 0;
}

int main() {
  leakMemory();
  return 0;
}

```
实验结果
![](https://s2.loli.net/2022/01/03/sywVU9LNm4aQFv3.png)


通过对比，发现O1优化条件下，两个函数的函数体中几乎没有指令了。这里刚好可以和前文的汇编结果对应上。致此，我的问题2解决了一半：**LLVM的opt 在生成IR的时候，把我的函数里"看起来没用的"指令去掉了。**

那究竟是什么pass？优化掉了我的代码呢？
找了2个看起来最有可能的pass来分析一下：
##### SimplifyCFGPass.cpp 
- SimplifyCFGPass.cpp pass 全名叫做simplify the CFG (control flow graph)，就是流程简化的意思，主要会做这几件事情
  - Removes basic blocks with no predecessors. 删除没有前任的基本块
  - Merges a basic block into its predecessor if there is only one and the predecessor only has one successor. 如果一个基本块的唯一前任只有它一个后继，就会把它合并在它的前任里
  - Eliminates PHI nodes for basic blocks with a single predecessor. 用单个前任消除基本块的PHI节点
  - Eliminates a basic block that only contains an unconditional branch. 消除仅包含无条件分支的基本块
  - .....

关于simplify CFG 的详细介绍，可以参考这篇讲义 [How LLVM Optimizes a Function](https://blog.regehr.org/archives/1603) [^3]

看到这里，我一度认为代码优化是在这里做的。但大致看一下**SimplifyCFGPass.cpp**的源码后就会发现：简化CFG 的操作是针对基本块来做的，而我的代码里每个函数只有一个逻辑路径，不存在if，switch等流程控制，所以每个函数也只有1个基本块。所以优化的点不在这里。事实上，本pass只会优化掉类似这样的基本块。

```
;<label>:13:
br label %14

;<label>:14:
.... ; label 14 的工作
br label %15

```

找错人了，不是这个pass。。。

![](https://s2.loli.net/2022/01/02/u9RTBLmS25ZHtPj.png)

##### mem2Reg.cpp
了解mem2Reg pass 之前，先简单学习一个概念: **SSA(静态单一赋值)**

SSA（静态单一赋值）：In compiler design, static single assignment form (often abbreviated as SSA form or simply SSA) is a property of an intermediate representation (IR), which requires that each variable is assigned exactly once, and every variable is defined before it is used. 
SSA 是一种属性，在这种属性下的IR文件，必须满足下面的两个要求：

1. **每个变量必须被赋值，并且只能被赋值1次**
2. **每个变量使用之前，必须声明**

SSA 通过简化程序中变量的特性，可以同时达到两种目的：第一，可以简化很多编译优化方法的过程；第二，对很多编译优化方法来说，可以获得更好的优化结果。
下面给出一个例子

```
; 没有满足SSA的IR
 y := 1
 y := 2
 x := y
```
```
; 满足SSA的IR
 y1 := 1
 y2 := 2
 x1 := y2 
```
在不满足SSA的IR代码中，我们需要“人肉分析”能看出来第1行的y = 1 赋值是没用的。而在满足SSA的IR代码中，通过对相同变量拆分成不同“版本”赋值的办法，可以更加直观的看出y1 是没用的。
对于采用非 SSA 形式 IR 的编译器来说，它需要做数据流分析（User-Define分析）来确定选取哪一行的 y 值。而LLVM中，通过IR优化最终产生的的IR是满足SSA的
**“LLVM 很可能在对IR代码优化使其满足SSA的时候，优化掉了明显不会被用到的变量”**

LLVM是怎样使IR满足SSA的呢？
LLVM 会采用一种alloca/load/store 的方法，把所有局部变量通过“alloca”指令进行分配。这样做的目的是将 构造 SSA 的工作从前端分离出来 。 简单来讲，就是对所有栈变量的使用，都需要包装为alloca/load/store。这里我们再来回顾一下前面生成的 fooO0.ll 
  - **mem2Reg.cpp** 这个pass 可以将采用了 **alloca/load/store**  量存取都需要访问技术生成的IR代码转换为SSA形式。
  - **mem2reg** 算法还是比较复杂的。在这里我们不深究mem2Reg算法的细节。读者可以通过这篇文章来更加深入了解mem2reg算法 [^4] 。这里只去简单的看下它的源码，在mem2reg的源码中，有一段关键注释可以解决我们的疑问
  
```
//函数 PromoteMem2Reg::run() mem2reg 算法的真正代码实现，该函数共200+行, 我们只找能回答问题的关键部分。
void PromoteMem2Reg::run() {
  Function &F = *DT.getRoot()->getParent();

  AllocaDbgDeclares.resize(Allocas.size());

  AllocaInfo Info;
  LargeBlockInfo LBI;
  ForwardIDFCalculator IDF(DT);

  /*for 循环依次处理每一个 AllocaInst ，在进行一些 assert 判断之后，调用函数 removeLifetimeIntrinsicUsers 在 LLVM IR 中删除该 AllocaInst 的 User 中除了 LoadInst 和 StoreInst 以外的 Instructions。如果某个 AllocaInst 没有 User，那么直接删除该 AllocaInst，并且将该 AllocaInst 从成员变量 std::vector<AllocaInst *> Allocas 中删除*/
  for (unsigned AllocaNum = 0; AllocaNum != Allocas.size(); ++AllocaNum) {
    AllocaInst *AI = Allocas[AllocaNum];

    assert(isAllocaPromotable(AI) && "Cannot promote non-promotable alloca!");
    assert(AI->getParent()->getParent() == &F &&
           "All allocas should be in the same function, which is same as DF!");

    removeLifetimeIntrinsicUsers(AI);

    if (AI->use_empty()) {
      // If there are no uses of the alloca, just delete it now.
      AI->eraseFromParent();

      // Remove the alloca from the Allocas list, since it has been processed
      RemoveFromAllocasList(AllocaNum);
      ++NumDeadAlloca;
      continue;
    }
    ....
}

```

### 总结一下
针对两个疑问 O1的优化选项究竟对我的代码做了什么？LLVM 的优化具体做了什么？已经有了个比较明确的答案：

**编译优化选项O1 的比O0 在opt 中多执行了一些pass，其中mem2Reg.cpp 的pass 中，将使用了alloc/load/store 技术的IR代码优化使其满足SSA，在这一过程中，泄漏函数变成了空实现，导致内存泄漏没有发生。**

在Apple的文档中（For Mac） ，也提到了推荐使用的优化等级
>As you near the end of your development cycle, though, the release build configuration can give you an indication of the size of your finished product, so the Fastest, Smallest option is appropriate.
Fastest, Smallest 对应的就是-OS选项，即执行大部分的优化选项，同时压缩了体积，这个level 是适合release使用的


### 引用 
[^1]: OOMDetector 是腾讯出品的一款内存泄漏检测工具。 [OOMDetector](https://github.com/Tencent/OOMDetector)
[^2]: [LLVM源码](https://github.com/llvm/llvm-project)
[^3]: [How LLVM Optimizes a Function](https://blog.regehr.org/archives/1603)
[^4]: [LLVM-TransformUtils-Mem2Reg]https://blog.csdn.net/yeshahayes/article/details/97018670