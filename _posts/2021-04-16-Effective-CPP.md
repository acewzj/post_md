---
title: Effective CPP
tags:
  - null
  - null
categories:
  - null
  - null
date: 2021-04-16 09:22:24
---

这篇文章记述了关于 Effective CPP 的知识点。

<!--more-->

1、将构造函数声明为 explicit，可以阻止它们被用来执行隐式类型转换。但它们仍然可以用来被执行显式类型转换。

> 什么叫隐式类型转换？比如你有一个对象类型为dog，你的参数是int，那么int和dog之间没有隐式类型转换。
>
> 借用标准里的话来说，就是当你只有一个类型T1，但是当前表达式需要类型为T2的值，如果这时候T1自动转换为了T2那么这就是隐式类型转换。
>
> int转成long是向上转换，通常不会有太大问题，而long到int则很可能导致数据丢失，因此要尽量避免后者。

2、拷贝构造函数被用来“以同型对象初始化自身对象”，拷贝运算符被用来“从另一个同型对象中拷贝其值到自我对象”。

> 比如你定义一个类叫Widget
>
> Widget w1;// 调用构造函数
>
> Widget w2(w1);//调用拷贝构造函数
>
> w1 = w2; //调用拷贝运算符函数
>
> ------
>
> Widget w3 = w2; //这里调用拷贝构造函数！
>
> 总结：
>
> 如果一个新的对象被定义（如 w3）,一定会有构造函数被调用，不可能调用赋值运算；如果没有新的对象被定义（如w1 = w2），就不会有构造函数被调用，而是调用赋值操作。

在进入本节前我们看一道经典的面试题：

```c++
std::string s = "hello c++";
```

请问创建了几个string呢？如果你脱口而出1个，那么面试官八成会狡黠一笑，让你回家等通知去了。

那么答案是什么呢？是1个或者2个。什么，你逗我呢？

先别急，我们分情况讨论。首先是c++11之前。

在c++11前题目里的表达式实际上会导致下面的行为：

1. 首先`"hello c++"`是`const char[N]`类型的，不过它在表达式中于是退化成`const char *`
2. 然后因为s实际上是处于“声明即定义”的表达式中，因此适用的只有复制构造函数，而不是重载的=
3. 因此等号的右半边必须也是`string`类型
4. 因为正好有从`const char *`到`string`的转换规则，因此把它转换成合适的类型
5. 转换完会返回一个新的`string`的临时量，它会作为参数调用复制构造函数
6. 复制构造函数调用完成后s也就创建完毕了。

在这里我们暂且忽略了string的写时复制等黑科技，整个过程创建了s和一个临时量，一共两个string。

很快c++11就出现了，同时还带来了移动语义，然而结果并没有改变：

1. 前面步骤相同，字符串字面量隐式转换成string，创建了一个临时量
2. 临时量是个右值，所以绑定给右值引用，因此移动构造函数被选择
3. 临时量里的数据移动到s里，s创建完成

移动语义减少了不必要的内部数据的复制，但是临时量还是会被创建的。

有进捣鼓编译器的朋友可能要说了，编译器是不生成这个临时量的。是这样的，编译器会用复制省略（copy elision）优化这段代码。

是的，复制省略在c++11里就已经被提到了，不过那时候它是可选的，并不强制编译器支持这一优化。因此你在GCC和clang上观察到的不一定能代表全部的c++编译器的情况，所以我们仍以标准为基础推演了理论上的行为。

到目前为止答案都是2，然而很快有意思的事情发生了——复制省略在c++17里成为了被标准化的行为。

在c++17里除非必要，否则临时量（现在叫做右值的结果对象，一个右值只有在实际需要存在一个临时变量的情况下才会创建一个临时变量，这个过程叫做实质化，创建出来的那个临时量就是该右值的结果对象）不会被创建，换而言之，`T obj = expr`这样的形式会以expr产生结果直接调用合适的构造函数，而不会进行临时量的创建和复制构造函数的调用，不过为了保证语义的完整性，复制构造函数仍然被要求是可访问的，毕竟类本身不允许复制构造的话复制初始化本身就是不正确的，不能因为复制省略而导致错误的代码被编译通过。

所以现在过程变成了下面这样子：

1. 编译器发现表达式是string的复制初始化
2. 右侧是表达式会隐式转换产生一个string的纯右值用于初始化同一类型的s
3. 判断复制构造函数是否可用，然后发现符合复制省略的条件
4. 寻找string里是否有符合要求的构造函数
5. 找到了`string::string(const char *)`，于是直接调用
6. s初始化完成

因此，在c++17下只会创建一个string对象，这比移动语义更加高效。这也是为什么我说题目的答案既可以是1也可以是2的原因。

同时我们还发现，在复制构造时的类型转换不管复制有没有被省略都是存在的，只不过换了一个形式，这就是我们后面要讲的内容。

