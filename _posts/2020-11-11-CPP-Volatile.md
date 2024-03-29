---
title: CPP-Volatile
tags:
  - CPP
  - Volatile
categories:
  - CPP
  - Volatile
date: 2020-11-11 15:35:57
---

这篇文章记述了关于 C++ Volatile 的知识点。

<!--more-->

![image-20201117102104340](https://i.loli.net/2020/11/17/mq1MUCPH37lSAFI.png)

## 阿里：violatile 的作用，是否具有原子性，对编译器有什么影响

volatile的本意是“易变的” 因为访问寄存器要比访问内存单元快的多,所以编译器一般都会作减少存取内存的优化，但有可能会读脏数据。当要求使用volatile声明变量值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。精确地说就是，遇到这个关键字声明的变量，编译器对访问该变量的代码就不再进行优化，从而可以提供对特殊地址的稳定访问；如果不使用valatile，则编译器将对所声明的语句进行优化。（简洁的说就是：volatile关键词影响编译器编译的结果，用volatile声明的变量表示该变量随时可能发生变化，与该变量有关的运算，不要进行编译优化，以免出错）

**violatile 不具有原子性**

volatile的本质：

1> 编译器的优化

在本次线程内, 当读取一个变量时，为提高存取速度，编译器优化时有时会先把变量读取到一个寄存器中；以后，再取变量值时，就直接从寄存器中取值；当变量值在本线程里改变时，会同时把变量的新值copy到该寄存器中，以便保持一致。

当变量在因别的线程等而改变了值，该寄存器的值不会相应改变，从而造成应用程序读取的值和实际的变量值不一致。

当该寄存器在因别的线程等而改变了值，原变量的值不会改变，从而造成应用程序读取的值和实际的变量值不一致。

2>volatile应该解释为“直接存取原始内存地址”比较合适，“易变的”这种解释简直有点误导人。

### 什么场景一定要使用 violatile

### violatile 能否和 const 一起用

const和volatile 也一样，所谓的const，只是编译器保证在C的“源代码”里面，没有对该变量进行修改的地方，而实际运行的时候则不是编译器所能管的了。同样，volatile的所谓“可能被修改”，是指“在运行期间”可能被修改。也就是告诉编译器，这个变量不是“只”会被这些C的“源代码”所操纵，其它地方也有操纵它们的地方。所以，C编译器就不能随便对它进行优化了。
　　const –>该变量为常量,不能在此程序中更改
　　volotile –>该变量为一个共享变量,也就是说会有除了本程序之外的其他途径对其值进行更改,如多线程,或是硬件，其他的运行程序.
const volatile表示该变量既不能被修改，又不能被优化到寄存器，即又是可能会被其他编译器不知道的方式修改的。比如一个`实时时钟`，我们不希望被程序做修改，所以要声明为const，但其他的线程、中断等（可能来自于库）又要修改此时钟的值，编译器不能把它作为const常量优化到寄存器，所以又要声明为volatile。再举个例子，`只读的状态寄存器`，它是volatile，因为它可能被意想不到地改变。它是const，因为程序不应该试图去修改它。在嵌入式系统当中这种修饰很常见，常用于修饰可能被硬件修改的寄存器，这些寄存器只是用来表示状态，并不能被写值（可能会导致错误）。比如我们在主循环中查询中断标志位。