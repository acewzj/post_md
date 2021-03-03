---
layout: post
title:  "HelloWorld的运行原理"
date:   2019-10-07 10:16:18 +0800
categories:
- CPP
tags:
- Linux
- C
- compile
- link
- 计算机底层
---



这篇文章主要介绍了HelloWorld的运行原理。


<!--more-->






# HelloWorld的运行原理（不断更新）
一直都想搞明白printf("HelloWorld！\n")是怎么在屏幕上打印出来的，所以趁着中秋节尽可能的深挖一下。这篇文章会保持持续的更新。

## 1. 编译链接阶段

### 1.1 预处理

首先，我们编写如下程序并命名为main.c。

```c
#include <stdio.h>
int main(int argc,char **argv()){
	printf("HelloWorld!\n");
	return 0;
}

```

![](https://i.loli.net/2019/09/13/WKjeIC2MX3Av7Zc.png)

输入`gcc -E main.c -o main.i` 进行预处理工作，在此处，选项"-o"是指输出目标文件为main.i

预编译的处理规则：

> 将所有的 “#define” 删除，并展开所有的宏定义
> 处理所有的条件预编译指令，比如：" #if #ifdef #elif #else #endif "
> 处理所有的 “#include” 预编译指令
> 删除所有的注释 “//” 、 “/* */”
> 添加行号和文件名标识，以便编译时产生的行号信息以及用于编译错误或警告时能够显示行号
> 保留所有的 “#pragma” 编译器指令

我们可以打开main.i看看，里面的头文件stdio.h已经被展开了，包括一些类型定义函数定义等等，截取片段如下:

![](https://i.loli.net/2019/09/13/rY6jE8gQv25zSfi.png)

这里找到了printf的外部声明如下：

`extern int printf (const char *__restrict __format, ...);`

printf()函数的调用格式为:`printf("格式化字符串",输出表列)`。

参考例子为`printf("%2c-%2c-%2c-%2c\n",'D','e','m','o');`

### 1.2 编译(生成汇编代码 main.s)

编译过程是编译器gcc把预处理完的文件进行词法分析、语法分析、语义分析及优化后生成相应的汇编代码文件。使用命令`gcc -S main.i -o main.s`将前面预处理的main.i文件编译成汇编语言文件main.s

```asm
        .file   "main.c";
        .text
        .section        .rodata
.LC0:
        .string "HelloWorld!"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp;
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        subq    $16, %rsp
        movl    %edi, -4(%rbp)
        movq    %rsi, -16(%rbp)
        leaq    .LC0(%rip), %rdi
        call    puts@PLT
        movl    $0, %eax
        leave
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (Ubuntu 7.4.0-1ubuntu1~18.04.1) 7.4.0"
        .section        .note.GNU-stack,"",@progbits
```

我们发现printf函数调用被转化为call puts指令，而不是call printf指令，这好像有点出乎意料。不过不用担心，这是编译器对printf的一种优化。实践证明，对于printf的参数如果是以'\n'结束的纯字符串，printf会被优化为puts函数，而字符串的结尾'\n'符号被消除。除此之外，都会正常生成call printf指令。

puts()函数有两个特点:

- puts()在显示字符串时会自动在其末尾添加一个换行符。
- puts()遇到空字符时就停止输出，所以必须确保有空字符。

注意汇编程序由三个不同的元素组成：

> **指示（Directives）** 以点号开始，用来指示对编译器，连接器，调试器有用的结构信息。指示本身不是汇编指令。例如，.file 只是记录原始源文件名。.data表示数据段(section)的开始地址, 而 .text 表示实际程序代码的起始。.string 表示数据段中的字符串常量。 .globl main指明标签main是一个可以在其它模块的代码中被访问的全局符号 。至于其它的指示你可以忽略。
>
> **标签（Labels）** 以冒号结尾，用来把标签名和标签出现的位置关联起来。例如，标签.LC0:表示紧接着的字符串的名称是 .LC0. 标签main:表示指令 pushq %rbp是main函数的第一个指令。按照惯例， 以点号开始的标签都是编译器生成的临时局部标签，其它标签则是用户可见的函数和全局变量名称。
>
> **指令（Instructions）** 实际的汇编代码 (pushq %rbp), 一般都会缩进，以便和指示及标签区分开来。

> <u>小贴士: AT&T 语法和 Intel 语法</u>
> <u>注意GNU工具使用传统的AT&T语法。类Unix操作系统上，AT&T语法被用在各种处理器上。Intel语法则一般用在DOS和Windows系统上。下面是AT&T语法的指令：</u>
> <u>movl %esp, %ebp</u>
> <u>movl是指令名称。%则表明esp和ebp是寄存器.在AT&T语法中, 第一个是源操作数,第二个是目的操作数。</u>
> <u>在其他地方，例如interl手册，你会看到是没有%的intel语法, 它的操作数顺序刚好相反。下面是Intel语法:</u>
> <u>MOVQ EBP, ESP</u>
> <u>当在网页上阅读手册的时候，你可以根据是否有%来确定是AT&T 还是 Intel 语法。</u>

每个寄存器都有其特殊用途，并不是所有指令都可以应用到每一个寄存器。随着设计的进展，新的指令和寻址模式被添加进来，使得很多寄存器变成了等同的。少数留下来的指令，特别是和字符串处理相关的，要求使用%rsi 和%rdi。另外，两个寄存器被保留下来分别作为栈指针 (%rsp) 和基址指针 (%rbp)。最后的8个寄存器是编号的并且没有特殊限制。多年来，体系结构从8位扩展到16位，32位，因此每个寄存器都有一些内部结构：

![](https://i.loli.net/2019/09/13/IusO7Gy4AYmxJP1.png)

%rax的低8位是8位寄存器%al, 仅靠的8位是%ah。低16位是 %ax， 低32位是 %eax，整个64位是%rax。

寄存器%r8-%r15也有相同结构，但命名方式稍有不同：

![](https://i.loli.net/2019/09/13/ijQqmTs2zOL8bk1.png)

### 1.3 汇编(生成main.o文件)

汇编是汇编器把汇编代码转变成中间目标文件。汇编过程可以使用如下命令：

`gcc -c main.s -o main.o`

main.o已经是二进制文件了，直接打开会发现乱码一片，我们可以用objdump或者gdb的反汇编指令打开obj文件，如下:

<img src="https://i.loli.net/2019/09/13/FSuZDNRlPWMy49f.png" style="zoom:67%;" />

？？目标文件是什么？

--目标文件是指编译器编译源代码后生成的二进制文件，再通过链接器和资源文件链接就成可执行文件了。OBJ只给出了程序的【相对地址】，而可执行文件是【绝对地址】。CPP对应的二进制代码格式obj，是未经重定位的！以下摘自《程序员的自我修养》

> 现在PC平台流行的可执行文件格式(Executable)，主要是Windows下的PE（Portable Executable）和linux的ELF （Executable Linkable Format）,他们都是COFF（Common File Format）格式的变种。COFF是由Unix System VRelease 3首先提出并且使用的文件规范，后来微软公司基于COFF格式，制定了PE格式标准，并将其用于当时的Windows NT系统。System VRelease 4在COFF的基础上引入了ELF格式，目前流行的Linux系统也是以ELF作为基本的可执行文件格式。这也能解释为什么目前PE和ELF如此相似的主要原因，因为他们都是来源于同一种可执行文件格式COFF。目标文件就是源代码编译后为进行链接的那些中间文件（Windows下面为.obj文件；Linux下面为.o文件），它和可执行文件的内容和结构很相似，所以一般和可执行文件采用同一种格式进行存储。从广义上来讲，目标文件与可执行文件的格式其实几乎是一模一样的，所以，我们可以广义的将目标文件和可执行文件看成是同一种类型的文件。在Windows下，我们把目标文件和可执行文件都统一称为PE-COFF文件，在Linux下，我们把它们统称为ELF文件。
> 当然，事情没有这么简单!不光是可执行文件（Windows下面的.exe和Linux下面的ELF文件）按照可执行文件格式存储。动态链接库（DLL,dynamic linking library）[Windows下面的.dll文件和Linux下面的.a文件]以及静态链接库（Static linking Library）[Windows下面的.lib文件和Linux下面的.a文件]都是按照可执行文件格式存储的。只不过，在Windows平台下，他们按照PE-COFF格式存储，而在Linux平台下按照ELF格式进行储存。
> ELF文件标准里面把系统中采用ELF格式的文件归为以下四类：
>
> <img src="https://i.loli.net/2019/09/14/qtUFRAal3iIfpOs.png" style="zoom:67%;" />
>
> <img src="https://i.loli.net/2019/09/14/cRPbO1lq8dgYEAB.png" style="zoom:67%;" />
>
> 假设上图的可执行文件格式是ELF，从图中可以看到，ELF文件的开头是一个“文件头”，他描述了整个文件的文件属性，包括文件是否可执行、是静态链接还是动态链接以及入口地址（如果是可执行文件）、目标硬件、目标操作系统等信息。头文件包含一个段表（Section Table），段表事实是一个描述文件中各个段的数组。段表描述了文件中各个段在文件中的偏移位置及段的属性，从段表里面可以得到每个段的所有信息。文件头后面就是各个段的内容，比如代码段保存的就是程序的指令，数据段里面保存的就是程序的静态变量等。

### 1.4 链接（生成可执行程序）

链接器 ld：负责将程序的目标文件与所需的所有附加的目标文件连接起来，附加的目标文件包括静态连接库和动态连接库,链接是链接器ld把中间目标文件和相应的库一起链接成为可执行文件。

`gcc main.o -o main`

![](https://i.loli.net/2019/09/14/63LkiSUlcRowJrD.png)

如果前面使用的是$ gcc main.c命令，默认会产生一个a.out 的可执行文件，使用命令./a.out执行该可执行文件

？？为什么会使用a.out作为名字？

-- 《Expert C Programming》中提到它是`assembler output(汇编程序输出)`的缩写，默认使用a.out的名字是UNIX“没什么理由，但是我们就是这么做的”思维的一个例子。

## 2. 装载运行阶段

在之前我写程序一般到`./main`就已经结束了。看着`HelloWorld`被输出之后就不再往下挖了，但是今天我们继续往它的祖坟上刨刨，看看到底能挖多深？

### 2.1 C语言内存模型及运行时内存布局





### 2.2 内核模式和用户模式

  首先我们要解释一个概念——进程（Process）。简单来说，一个可执行程序就是一个进程，前面我们使用C语言编译生成的程序，运行后就是一个进程。进程最显著的特点就是拥有独立的地址空间。

严格来说，程序是存储在磁盘上的一个文件，是指令和数据的集合，是一个静态的概念；进程是程序加载到内存运行后一系列的活动，是一个动态的概念。

前面我们在讲解地址空间时，一直说“程序的地址空间”，这其实是不严谨的，应该说“进程的地址空间”。一个进程对应一个地址空间，而一个程序可能会创建多个进程。  

内核空间存放的是操作系统内核代码和数据，是被所有程序共享的，在程序中修改内核空间中的数据不仅会影响操作系统本身的稳定性，还会影响其他程序，这是非常危险的行为，所以操作系统禁止用户程序直接访问内核空间。

要想访问内核空间，必须借助操作系统提供的 API 函数，执行内核提供的代码，让内核自己来访问，这样才能保证内核空间的数据不会被随意修改，才能保证操作系统本身和其他程序的稳定性。

用户程序调用系统 API 函数称为系统调用（System Call）；发生系统调用时会暂停用户程序，转而执行内核代码（内核也是程序），访问内核空间，这称为内核模式（Kernel Mode）。

用户空间保存的是应用程序的代码和数据，是程序私有的，其他程序一般无法访问。当执行应用程序自己的代码时，称为用户模式（User Mode）。

计算机会经常在内核模式和用户模式之间切换：

- 当运行在用户模式的应用程序需要输入输出、申请内存等比较底层的操作时，就必须调用操作系统提供的 API 函数，从而进入内核模式；
- 操作完成后，继续执行应用程序的代码，就又回到了用户模式。

总结：用户模式就是执行应用程度代码，访问用户空间；内核模式就是执行内核代码，访问内核空间（当然也有权限访问用户空间）。

#### ？？为什么要区分两种模式

-- 我们知道，内核最主要的任务是管理硬件，包括显示器、键盘、鼠标、内存、硬盘等，并且内核也提供了接口（也就是函数），供上层程序使用。当程序要进行输入输出、分配内存、响应鼠标等与硬件有关的操作时，必须要使用内核提供的接口。但是用户程序是非常不安全的，内核对用户程序也是充分不信任的，当程序调用内核接口时，内核要做各种校验，以防止出错。

从 Intel 80386 开始，出于安全性和稳定性的考虑，CPU 可以运行在 ring0 ~ ring3 四个不同的权限级别，也对数据提供相应的四个保护级别。不过 Linux 和 Windows 只利用了其中的两个运行级别：

- 一个是内核模式，对应 ring0 级，操作系统的核心部分和设备驱动都运行在该模式下。
- 另一个是用户模式，对应 ring3 级，操作系统的用户接口部分（例如 Windows API）以及所有的用户程序都运行在该级别。

#### ？？为什么内核和用户程序要共用地址空间

--既然内核也是一个应用程序，为何不让它拥有独立的4GB地址空间，而是要和用户程序共享、占用有限的内存呢？

让内核拥有完全独立的地址空间，就是让内核处于一个独立的进程中，这样每次进行系统调用都需要切换进程。切换进程的消耗是巨大的，不仅需要寄存器进栈出栈，还会使CPU中的数据缓存失效、MMU中的页表缓存失效，这将导致内存的访问在一段时间内相当低效。

而让内核和用户程序共享地址空间，发生系统调用时进行的是模式切换，模式切换仅仅需要寄存器进栈出栈，不会导致缓存失效；现代CPU也都提供了快速进出内核模式的指令，与进程切换比起来，效率大大提高了。

## 3. 进程运行时

当我们在终端输入

> ./main

这时，终端会调用fork创建一个子进程，fork函数是一个系统调用，这时会嵌入内核，调用的是clone函数，这个clone函数接着会调用do_fork函数，这个函数做了大部分创建工作，例如创建task_struct，内核栈，thread_info等结构，因为fork函数是采用写时复制技术，因此此时子进程task_struct大部分的属性还是和父进程task_struct一样，主要就是没有为子进程开辟内存空间；当子进程内核结构创建好之后，这时进程调度系统会优先调度子进程，因为一般情况下，子进程会直接调用exec函数避免写时拷贝开销．

> 这里提下，一次fork调用为什么会返回两个值；因为当fork在内核调用成功，要返回用户态时，如果此时调度子进程执行，那么会把0放入rax寄存器中，等fork返回用户态执行子进程时，从rax得到的就是0；当内核调度的是父进程时，这时会把子进程的id号放入rax寄存器中，等返回到用户态执行父进程时，此时从rax获得的就是子进程的id号；

到这里，还是只是创建了子进程内核的一些结构，接下来，在终端fork的子进程中，会调用exec系列函数

> execl("./hello","hello","",NULL);

这个函数会会为子进程hello单独开辟一块内存(之前是和父进程共用内存空间)，其实最主要就是为mm_struct结构以及页表重新赋值；具体怎么做了，最主要是调用mmap函数；我们可以把上述图的左边看成是躺在磁盘中的可执行文件，右边对应的是进程在内存中布局；当内核要将可执行文件的代码段映射到进程空间时，内核会先把code segment读取进内核的cache中，然后给hello进程的code段分配一块vma虚拟内存，并把这块虚拟内存映射到在cache中code segment，并把这块vma放入mm_struct的红黑树和链表中．链表适合当需要遍历所有vma内存区域时，而红黑树适合快速获取某个特定内存区域；我们经常查看/proc/<pid>/maps某个进程的内存布局，其实就遍历这个进程的vma链表即可．

### 3.1 进程是如何执行的?

进程的执行其实就是cpu获取指令以及数据，并进行计算，这里cpu各个寄存器的使用和虚拟内存与物理内存转换等等；而main这个进程非常简单，就是调用C库函数printf输出一句话．但是其中涉及的过程还是相当复杂的．

首先当执行到printf时，因为main.c中没有定义这个函数，所以进程会去C库的动态链接库查找，当找到printf之后，进程会跳转到printf函数执行；在printf函数内部，会执行系统调用

> write(1, "hello world!\n", 13)

其中1是STDOUT_FILENO表示标准输出，对应的就是输出到显示器；到这里，我们可以聊聊系统调用；当执行这个write函数时，因为write是个系统调用，在执行这个write之前，会将参数放入寄存器中，例如1放在rdi,＂hello world!\n"字符串指针放入rsi,13放入rdx寄存器中．linux在执行系统调用时，会触发一次int80软中断，并把系统调用号放入rax寄存器中；这时cpu切换到内核软中断处理函数中，怎么找到这个软中断函数？这个说起来，话又很多了(IDTR寄存器和中断描述符表)．在中断函数中，找到rax寄存器对应的系统调用，write对应的是sys_write函数，开始执行sys_write函数．在sys_write函数中，会通过fd找到file结构，inode结构等等，最后输出到显示器．

### 3.2 进程是如何退出的

当进程执行结束之后，会调用exit函数，而这个函数调用的系统调用函数_exit()会嵌入内核，进行清除工作．例如释放进程打开的文件，释放进程mm_struct对应的内存(如果没有共享内存)等等，最后只剩下task_struct,内核栈和thread_info三个结构．子进程会给父进程发送一个SIGCHLD信号，表示进程退出；父进程在收到这个信号之后，会调用wait或waitpid函数回收子进程的资源，task_struct,内核栈以及thread_info．到这里，main进程的生命就算走完了．











[1]: https://blog.csdn.net/pro_technician/article/details/78173777
[2]: https://blog.csdn.net/qq_32385873/article/details/83304396
[3]: http://luodw.cc/2016/07/02/helloworld/
[4]: https://blog.csdn.net/zhengqijun_/article/details/72454714







/etc/profile : 在登录时,操作系统定制用户环境时使用的第一个文件 ,此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行。

```shell
export PATH="/home/acewzj/Software/5.11.1/gcc_64/bin:$PATH"

export PATH="/home/acewzj/Software/Tools/QtCreator/bin:$PATH"

export M2_HOME=/usr/local/apache-maven-3.6.2

export PATH=${M2_HOME}/bin:$PATH

```

/etc /environment : 在登录时操作系统使用的第二个文件, 系统在读取你自己的profile前,设置环境文件的环境变量

```shell
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64/bin/java"

```

~/.profile :  在登录时用到的第三个文件 是.profile文件,每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次!默认情况下,他设置一些环境变量,执行用户的.bashrc文件。

```shell
# ~/.profile: executed by the command interpreter for login shells.
# This file is not read by bash(1), if ~/.bash_profile or ~/.bash_login
# exists.
# see /usr/share/doc/bash/examples/startup-files for examples.
# the files are located in the bash-doc package.

# the default umask is set in /etc/profile; for setting the umask
# for ssh logins, install and configure the libpam-umask package.
#umask 022

# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
	. "$HOME/.bashrc"
    fi
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi

```

/etc/bashrc : 为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取.

```shell
acewzj@acewzj:/etc$ cat bash.bashrc 
# System-wide .bashrc file for interactive bash(1) shells.

# To enable the settings / commands in this file for login shells as well,
# this file has to be sourced in /etc/profile.

# If not running interactively, don't do anything
[ -z "$PS1" ] && return

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, overwrite the one in /etc/profile)
# but only if not SUDOing and have SUDO_PS1 set; then assume smart user.
if ! [ -n "${SUDO_USER}" -a -n "${SUDO_PS1}" ]; then
  PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi

# Commented out, don't overwrite xterm -T "title" -n "icontitle" by default.
# If this is an xterm set the title to user@host:dir
#case "$TERM" in
#xterm*|rxvt*)
#    PROMPT_COMMAND='echo -ne "\033]0;${USER}@${HOSTNAME}: ${PWD}\007"'
#    ;;
#*)
#    ;;
#esac

# enable bash completion in interactive shells
#if ! shopt -oq posix; then
#  if [ -f /usr/share/bash-completion/bash_completion ]; then
#    . /usr/share/bash-completion/bash_completion
#  elif [ -f /etc/bash_completion ]; then
#    . /etc/bash_completion
#  fi
#fi

# sudo hint
if [ ! -e "$HOME/.sudo_as_admin_successful" ] && [ ! -e "$HOME/.hushlogin" ] ; then
    case " $(groups) " in *\ admin\ *|*\ sudo\ *)
    if [ -x /usr/bin/sudo ]; then
	cat <<-EOF
	To run a command as administrator (user "root"), use "sudo <command>".
	See "man sudo_root" for details.
	
	EOF
    fi
    esac
fi

# if the command-not-found package is installed, use it
if [ -x /usr/lib/command-not-found -o -x /usr/share/command-not-found/command-not-found ]; then
	function command_not_found_handle {
	        # check because c-n-f could've been removed in the meantime
                if [ -x /usr/lib/command-not-found ]; then
		   /usr/lib/command-not-found -- "$1"
                   return $?
                elif [ -x /usr/share/command-not-found/command-not-found ]; then
		   /usr/share/command-not-found/command-not-found -- "$1"
                   return $?
		else
		   printf "%s: command not found\n" "$1" >&2
		   return 127
		fi
	}
fi
PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/lib/pkgconfig
export PKG_CONFIG_PATH

```



~/.bashrc : 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取。

```shell
acewzj@acewzj:/etc$ cat ~/.bashrc
# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac

# don't put duplicate lines or lines starting with space in the history.
# See bash(1) for more options
HISTCONTROL=ignoreboth

# append to the history file, don't overwrite it
shopt -s histappend

# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# If set, the pattern "**" used in a pathname expansion context will
# match all files and zero or more directories and subdirectories.
#shopt -s globstar

# make less more friendly for non-text input files, see lesspipe(1)
[ -x /usr/bin/lesspipe ] && eval "$(SHELL=/bin/sh lesspipe)"

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, unless we know we "want" color)
case "$TERM" in
    xterm-color|*-256color) color_prompt=yes;;
esac

# uncomment for a colored prompt, if the terminal has the capability; turned
# off by default to not distract the user: the focus in a terminal window
# should be on the output of commands, not on the prompt
#force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
	# We have color support; assume it's compliant with Ecma-48
	# (ISO/IEC-6429). (Lack of such support is extremely rare, and such
	# a case would tend to support setf rather than setaf.)
	color_prompt=yes
    else
	color_prompt=
    fi
fi

if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi
unset color_prompt force_color_prompt

# If this is an xterm set the title to user@host:dir
case "$TERM" in
xterm*|rxvt*)
    PS1="\[\e]0;${debian_chroot:+($debian_chroot)}\u@\h: \w\a\]$PS1"
    ;;
*)
    ;;
esac

# enable color support of ls and also add handy aliases
if [ -x /usr/bin/dircolors ]; then
    test -r ~/.dircolors && eval "$(dircolors -b ~/.dircolors)" || eval "$(dircolors -b)"
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'

    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
fi

# colored GCC warnings and errors
#export GCC_COLORS='error=01;31:warning=01;35:note=01;36:caret=01;32:locus=01:quote=01'

# some more ls aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias mm='tldr'
# Add an "alert" alias for long running commands.  Use like so:
#   sleep 10; alert
alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'

# Alias definitions.
# You may want to put all your additions into a separate file like
# ~/.bash_aliases, instead of adding them here directly.
# See /usr/share/doc/bash-doc/examples in the bash-doc package.

if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

# enable programmable completion features (you don't need to enable
# this, if it's already enabled in /etc/bash.bashrc and /etc/profile
# sources /etc/bash.bashrc).
if ! shopt -oq posix; then
  if [ -f /usr/share/bash-completion/bash_completion ]; then
    . /usr/share/bash-completion/bash_completion
  elif [ -f /etc/bash_completion ]; then
    . /etc/bash_completion
  fi
fi

```
