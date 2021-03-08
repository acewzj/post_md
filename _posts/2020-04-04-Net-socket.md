---
title: socket编程
date: 2020-04-02 10:27:17
tags:
  - CPP 
  - Net
categories:
  - socket
---

这篇文章记述了关于 socket 的知识点。

<!--more-->

## 不同socket地址结构对比

![image-20200401224452953](https://i.loli.net/2020/04/01/8ZcPnrdFHtVwW5D.png)



## accept

`clntfd = accept(serfd,(struct sockaddr*)&servaddr,sizeof(clntaddr));`

`int accept(int sock,struct sockaddr* addr,socklen_t* addrlen);`

成功时返回创建的套接字文件描述符，失败时返回-1

为什么这里会再创建一个fd呢？

这是因为accept函数受理连接请求队列里面待处理的客户端的连接请求。函数调用成功后，accept函数内部会生成用于数据I/O的套接字，并返回其文件描述符。以下这幅图很形象的描述了这个过程：也就是说你客户端connect到我服务器listen的队列里，服务器有空了就从队列里取队头的客户端进行accept创建套接字进行连接。

它之所以被命名为套接字，就是说得配套，客户端人家是有connect给自动创建出来的fd，服务器先开始通过socket函数返回了文件描述符server_fd，这个是告诉进程可以通过像读写文件那样来进行网络通信

这里第二个参数是保存连接过来的客户端的IP地址，给它填充上127.0.0.1

![image-20200401101317898](https://i.loli.net/2020/04/01/1gzbMQ6qr5VkNY7.png)



## connection refused

服务器正常启动，但是客户端一启动就在`connect`函数退出了，错误显示`connection refused`，单步调试也看不出什么来。

上网一看，有的人也碰到了类似问题，他们怀疑出现这个状况最大的可能是服务器正在监听程序，但是客户端并没有按照规矩的IP和端口号来给服务器发程序：这不有的人就是因为把服务器的`serv_addr.sin_port = htons(atoi(argv[1]));`写错为`serv_addr.sin_port = htonl(atoi(argv[1]));`导致端口号分配出错了。

![image-20200401092200981](https://i.loli.net/2020/04/01/2UY1yrgvwzsNjD8.png)

当然我也好不到哪里去？在使用telnet localhost 9000发现服务器正常好用之后，我坚信我的客户端程序出了问题：一行一行代码排查，终于发现`serv_addr.sin_port = htons(argv[1]);` 我特么忘了加atoi，这端口号肯定是错了啊【初始端口为9000，16进制为0x2328经过hostToNetShort后变成0x2823–>10275】

![image-20200401100250744](https://i.loli.net/2020/04/01/OcltiYzaLw89gv3.png)

![image-20200401100638777](https://i.loli.net/2020/04/01/L6Qp4ORXPeFi5UY.png)











## 如何判断机器的大端与小端？

### 常见的字节序

一般操作系统都是小端，而通讯协议是大端的。

![image-20200401213715103](https://i.loli.net/2020/04/01/Nxpg5lwZLejzRTA.png)

```c
#define BigtoLittle16(A)  ((((uint16)(A) & 0xff00) >> 8) | 
							(((uint16)(A) & 0x00ff) << 8))
  
 
#define BigtoLittle32(A)  ((((uint32)(A) & 0xff000000) >> 24) |                                            						(((uint32)(A) & 0x00ff0000) >> 8) | \
                   (((uint32)(A) & 0x0000ff00) << 8) | \
                   (((uint32)(A) & 0x000000ff) << 24))
```

## IP地址怎么存入int里面

$127.0.0.1$

$1 << 24 = 1 * 2^{24}->16777216 + 127 = 16777343$



因为通讯协议是大端，也就是高位127在低地址处0

 输入

```
10.0.3.193
167969729
```

输出

```
167773121
10.3.3.193
```

```c
#include<iostream>
#include<stack>
#include<string>
using namespace std;

int main()
{
unsigned int a,b,c,d;
char ch;
while(cin>>a>>ch>>b>>ch>>c>>ch>>d)
    {
    cout<<((a<<24)|(b<<16)|(c<<8)|d)<<endl;
    cin>>a;
    cout<<((a&0xff000000)>>24)<<"."<<((a & 0x00ff0000)>>16)<<"."<<((a&0x0000ff00)>>8)<<"."<<(a & 0x000000ff)<<endl;
    }
return 0;

}
```



## Select

文件描述符的监视范围与第一个参数有关，实际上，select函数通过第一个参数来传递监视对象文件描述符的数量。因此，需要得到注册在fd_set变量中的文件描述符数量。但是每次新建文件描述符时，其值都会加1，故只需要将最大的文件描述符值加1再传递到select函数即可。加1是因为文件描述符的值从0开始。

```c
if((fd_num = select(fd_max+1,&cpy_reads,0,0,&timeout)) == -1)
    
for(int i = 0;i<fd_max+1;i++)
        {
            if(FD_ISSET(i,&cpy_reads))
            {
                if(i == servfd)
                {
                    clntfd = 
                     accept(i,(struct sockaddr*)&servaddr,sizeof(clntaddr));
                    FD_SET(clntfd,&reads);//reset 1
                    if(fd_max < clntfd)
                        fd_max = clntfd;//last fd occur
                    printf("connected! client :%d\n",clntfd);
                }
                else
                {
                    str_len = read(i,buf,BUF_SIZE);
                    if(str_len == 0)
                    {
                        FD_CLR(i,&reads);
                        close(i);
                        printf("closed client:%d \n",i);
                    }
                    else
                    {
                        write(i,buf,BUF_SIZE);
                    }                   
                    
                }
```

```
socklen_t clntsocklen = sizeof(clntaddr);
                    clntfd = accept(i,(struct sockaddr*)&servaddr,&clntsocklen);
```

请注意这两个代码段里面关于accept函数最后一个参数，它要求的是

`int accept(int __fd, struct sockaddr *__restrict__ __addr, socklen_t *__restrict__ __addr_len)`

其实是一个指向一个长度整数的地址。先开始我给传的是sizeof(clntaddr),但是调试时在这里返回-1.

![image-20200429224802055](https://i.loli.net/2020/04/30/XoJF9HvuCW1Vqsl.png)

## Epoll

**创建epoll对象**

如下图所示，当某个进程调用epoll_create方法时，内核会创建一个eventpoll对象（也就是程序中epfd所代表的对象）。eventpoll对象也是文件系统中的一员，和socket一样，它也会有等待队列。创建一个代表该epoll的eventpoll对象是必须的，因为内核要维护“就绪列表”等数据，“就绪列表”可以作为eventpoll的成员。

![image-20201102201252909](https://i.loli.net/2020/11/02/5p8BZNdc3AeStbJ.png)

![image-20201102200800299](https://i.loli.net/2020/11/02/EIMrH8jazZnSuCp.png)

**维护监视列表**

创建epoll对象后，可以用epoll_ctl添加或删除所要监听的socket。以添加socket为例，如下图，如果通过epoll_ctl添加sock1、sock2和sock3的监视，内核会将eventpoll添加到这三个socket的等待队列中。

![image-20201102201327381](https://i.loli.net/2020/11/02/bBaNEwn3RgCYLtP.png)

当socket收到数据后，中断程序会操作eventpoll对象，而不是直接操作进程。

**接收数据**

当socket收到数据后，中断程序会给eventpoll的“就绪列表”添加socket引用。如下图展示的是sock2和sock3收到数据后，中断程序让rdlist引用这两个socket。

![image-20201102201359620](https://i.loli.net/2020/11/02/rF46AjcTGeXLwZS.png)

eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。

当程序执行到epoll_wait时，如果rdlist已经引用了socket，那么epoll_wait直接返回，如果rdlist为空，阻塞进程。

**阻塞和唤醒进程**

假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了epoll_wait语句。如下图所示，内核会将进程A放入eventpoll的等待队列中，阻塞进程。

![image-20201102201107158](https://i.loli.net/2020/11/02/XUSg75REOsb2wmZ.png)

当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。

![image-20201102201434741](https://i.loli.net/2020/11/02/Bbps2w4lVZknWoU.png)



![image-20200328232049698](https://i.loli.net/2020/03/30/4FkcyDpC9ZxMiTu.png)

## epoll_create

`int fd = epoll_create(int size)`

这个函数创建文件描述符，它创建的fd主要是用来区分epoll例程；size参数已废弃

比如`servfd = socket(PF_INET,SOCK_STREAM,0);`创建的`servfd`是5；

`epfd = epoll_create(EPOLL_SIZE);`创建的`epfd`就是6,实际上也是一个文件描述符；

## epoll_ctl

```c
    //create fd    
    epfd = epoll_create(EPOLL_SIZE);
    ep_events = malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

    event.events = EPOLLIN;
    event.data.fd=servfd;
    epoll_ctl(epfd,EPOLL_CTL_ADD,servfd,&event);//将servfd注册到epollpoll里面
```



## epoll_wait

`int epoll_wait(int epfd,struct epoll_event *event,int maxevents,int timeout)`

`event_cnt = epoll_wait(epfd,ep_events,EPOLL_SIZE,-1);`



返回值是发生事件的文件描述符数量，如果有5个客户端连接，就是返回5；

同时在第二个参数所指向的缓冲中保存发生事件的文件描述符集合。因此无需像select那样插入针对所有文件描述符的循环。

`int maxevents` ：第二个参数中可以保存的最大事件数；

timeout:以1ms为单位的等待时间，传递-1时，一直等待直到事件发生；

![image-20200420224546779](https://i.loli.net/2020/04/20/n3NkFtTXyU5DzRO.png)

epoll:产生的事件数；及消息；使用注意

epoll 全称 eventpoll，是 linux 内核实现IO多路复用（IO multiplexing）的一个实现

epoll 监听的fd(File Descriptor)集合是常驻内核的,它有3个系统调用(*epoll_create*, *epoll_wait*, epoll_ctl)通过 *epoll_wait* 可以多次监听同一个 fd 集合，只返回可读写那部分

select 只有一个系统调用，每次要监听都要将其从用户态传到内核，有事件时返回整个集合。

从性能上看，如果 fd 集合很大，用户态和内核态之间数据复制的花销是很大的，所以 select 一般限制 fd 集合最大1024。

从使用上看，epoll 返回的是可用的 fd 子集，select 返回的是全部，哪些可用需要用户遍历判断。

尽管如此，epoll 的性能并不必然比 select 高，对于 fd 数量较少并且 fd IO 都非常繁忙的情况 select 在性能上有优势。

## 数字电路里的边沿触发和电平触发

边沿触发包括上升边沿触发和下降边沿触发，边沿触发检测的是电平变化，高电平转低电平或低电平转高电平时，触发一次中断。

边沿触发通过D触发器来锁存中断信号，即：若CPU来不及响应中断，外部中断信号撤消后，由于D触发器的记忆作用，消失的中断信号仍然有效，直到中断被响应并进入中断ISR，记忆的中断信号才会由硬件自动清除。

-----

电平触发分为高电平触发和低电平触发；电平触发需要手动清除中断信号

电平触发根据硬件设计的不同，分为即时触发和信号锁存触发；

（1）即时的电平触发，当外部中断信号撤消时，中断申请信号随之消失。如果在外部中断信号申请期间，CPU来不及响应此中断，那么有可能这次中断申请就漏掉了。

即时的电平触发是一个时间段，需要一直触发中断的，就用电平触发。比如高电平触发，只要检测到是高电平就触发中断。

（2）信号锁存的电平触发，当检测到高电平或低电平信号，该触发信号也会被锁存，类似于边沿触发，但是触发信号需要进行手动清除；（注意前面的边沿触发会由硬件自动清除）

 

3、边沿触发及电平触发的区别

如果是采用边沿检测外部中断，检测到电平变化会中断，但是如果中断检测口一直保持某一电平，则无法产生下次中断，需要等下次检测到电平变化才会中断。中断得到响应后由硬件自动清除。

如果是采用电平检测外部中断，检测到低/高电平会中断，但是如果中断检测口一直保持低电平，中断处理完成后，会继续产生下次中断，需要检测到高电平才会停止中断产生。中断得到响应后由硬件手动清除。

![image-20200328232423802](https://i.loli.net/2020/03/30/e7WKMlwZguRIyE4.png)



## 条件触发程序

在条件触发方式中，只要输入缓冲有数据就会一直通知该事件。

当我的服务器接收缓冲区只有4的时候，客户端发了一个“Hello？”

这时服务器会从wait返回，读取的时候只能读入Hell四个字符

剩下的字符o?会再次触发，再次从wait返回。

## 边缘触发程序

边缘触发事件中，接收数据时仅注册一次该事件

所以一旦发生输入相关事件，就应该读取输入缓冲中的全部数据。因此需要验证输入缓冲是否为空。

`read函数返回 -1， 变量errno中的值为EAGAIN时，说明没有数据可读`

既然如此，为何还需要将套接字变成非阻塞模式？

边缘触发下，以阻塞方式工作的read & write函数有可能引起服务器端的长时间停顿。

> 这里的长时间停顿是指：当读或者写大量数据时，比如1KB 数据时，肯定会长时间停顿。

因此边缘触发方式中一定要采用非阻塞的read & write函数。

## 慢系统调用阻塞状态下来了个信号（Vital）

select函数、accept函数等都是慢系统调用函数，就是它得等系统网络IO准备好了才能执行，没有准备好就进入阻塞状态下，如果这个时候来了一个信号（因为我们知道信号是模仿的中断的嘛），当执行完信号处理函数返回后，PC指针就会跑到select函数的下一条指令去执行去了。。。。。。这不完蛋了吗：我等待的事件还没来呢我就被信号处理函数给跳过去了。

所以我们需要在select函数的返回值上做文章，如果select函数失败会返回-1，如果是去执行信号处理函数去了，error = EINTR

## 非阻塞编程fcntl

对于一个给定的描述符，有两种为其指定非阻塞IO的方法；

1、如果调用 open 获得描述符，则可指定 O_NONBLOCK 标志；

2、对于已经打开的一个描述符，则可调用 fcntl ，由该函数打开 O_NONBLOCK 标志

```c
void setnonblockingmode(int fd)
{
    int flag = fcntl(fd, F_GETFL, 0);
    fcntl(fd,F_SETFL, flag|O_NONBLOCK);
}
```

3、

```c
static int setnonblocking( int fd )
{
    int old_option = fcntl( fd, F_GETFL );//2
    int new_option = old_option | O_NONBLOCK;//O_NONBLOCK = 04000(八进制) 2048十进制 或之后变成2050
    fcntl( fd, F_SETFL, new_option );
    return old_option;
}
```





POSIX.1 要求，对于一个非阻塞的描述符如果无数据可读，则 read 返回 -1，errno 被置为 EAGAIN；

让服务器不会阻塞在send，accept等函数之上，防止有人恶意攻击

![image-20200322111836020](https://i.loli.net/2020/03/30/4txL8PY9qgjCdbJ.png)

![image-20200322112447217](https://i.loli.net/2020/03/30/JuRfAcSBvnHa1qr.png)

![image-20200322112833866](https://i.loli.net/2020/03/30/fHRLFJg8sGY6t4b.png)

![image-20200322113420519](https://i.loli.net/2020/03/30/M1fJEvI9H56tsWQ.png)

## 用户进程缓冲区

前面提到，用户进程通过系统调用访问系统资源的时候，需要切换到内核态，而这对应一些特殊的堆栈和内存环境，必须在系统调用前建立好。而在系统调用结束后，cpu会从核心模式切回到用户模式，而堆栈又必须恢复成用户进程的上下文。而这种切换就会有大量的耗时。

你看一些程序在读取文件时，会先申请一块内存数组，称为buffer，然后每次调用read，读取设定字节长度的数据，写入buffer。（用较小的次数填满buffer）。之后的程序都是从buffer中获取数据，当buffer使用完后，在进行下一次调用，填充buffer。

所以说：**用户缓冲区的目的是为了减少系统调用次数，从而降低操作系统在用户态与核心态切换所耗费的时间。**

## 内核缓冲区

除了在进程中设计缓冲区，内核也有自己的缓冲区。

当一个用户进程要从磁盘读取数据时，内核一般不直接读磁盘，而是将内核缓冲区中的数据复制到进程缓冲区中。

但若是内核缓冲区中没有数据，内核会把对数据块的请求，加入到请求队列，然后把进程挂起，为其它进程提供服务。

等到数据已经读取到内核缓冲区时，把内核缓冲区中的数据读取到用户进程中，才会通知进程，当然不同的io模型，在调度和使用内核缓冲区的方式上有所不同，下一小结介绍。

你可以认为，read是把数据从内核缓冲区复制到进程缓冲区。write是把进程缓冲区复制到内核缓冲区。

当然，write并不一定导致内核的写动作，比如os可能会把内核缓冲区的数据积累到一定量后，再一次写入。这也就是为什么断电有时会导致数据丢失。

所以说`内核缓冲区，是为了在OS级别，提高磁盘IO效率，优化磁盘写操作。`

1. 用户进程向 CPU 发起 read 系统调用读取数据，由用户态切换为内核态，然后一直阻塞等待数据的返回。
2. CPU 在接收到指令以后对磁盘发起 I/O 请求，将磁盘数据先放入磁盘控制器缓冲区。
3. 数据准备完成以后，磁盘向 CPU 发起 I/O 中断。
4. CPU 收到 I/O 中断以后将磁盘缓冲区中的数据拷贝到内核缓冲区，然后再从内核缓冲区拷贝到用户缓冲区。
5. 用户进程由内核态切换回用户态，解除阻塞状态，然后等待 CPU 的下一个执行时间钟。

## socket编程

我们知道两个进程如果需要进行通讯最基本的一个前提能能够唯一的标示一个进程，在本地进程通讯中我们可以使用PID来唯一标示一个进程，但PID只在本地唯一，网络中的两个进程PID冲突几率很大，这时候我们需要另辟它径了，我们知道IP层的ip地址可以唯一标示主机，而TCP层协议和端口号可以唯一标示主机的一个进程，这样我们可以利用**ip地址＋协议＋端口号**唯一标示网络中的一个进程。

![image-20201102102819557](https://i.loli.net/2020/11/02/Sz9ytWGHPNIVhao.png)

Socket是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的\*TCP/IP\*协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。一般由操作系统或者JVM自己实现。java.net中的socket其实就是对底层的抽象调用。有一点需要注意，运行在同一主机上的其他应用程序可能也会通过底层套接字抽象来使用网络，因此会与java socket实例竞争资源，如端口。对于“套接字结构”，是指底层实现（包括JVM和TCP/IP，但通常是后者）的数据结构集，包含了特定Socket所关联的信息。套接字和套接字数据结构是不一样的概念。套接字结构包含：

1.该套接字所关联的本地和远程互联网地址和端口。

2.一个FIFO队列用于存放接收到的等待分配的数据（RecvQ），以及一个用于存放等待传输数据的的队列（SendQ）。

3.对于TCP套接字，还包含了与打开关闭TCP握手相关的额外协议状态信息。

![image-20201102103000373](https://i.loli.net/2020/11/02/cMkU3TQhuF21p9e.png)

netstat可以查看本地和远程IP地址和端口的连接状态和sendQ和RecvQ中的字节数。

![image-20201102103033993](https://i.loli.net/2020/11/02/w1b3ryzXiD8C56H.png)

TCP是一种可信赖的字节流服务，任何写入socket输出流的数据副本必须保留（保留到本地缓冲区），直到另一端成功的接收。向输出流写入信息并不意味着数据实际上已经被发送，他们只是被复制到了本地缓冲区。就算调用flush（）也不会保证能立即发送到信道。

###  缓冲与数据传输

​    ！！！不能假设从一端写入输出流的数据和在另一端从输入流读出数据之间有任何的一致性。尤其是在发送端由单个输出流的write()方法传输数据，可能要经过另一端的多个read()方法获取，而一个read（）方法可以返回多个write（）写入的内容。为了展示这种情况，给出如下程序：

![image-20201102103112577](https://i.loli.net/2020/11/02/hlFsijIpH2gtQBm.png)

> 这个TCP连接向接收端传输8000字节，在接收端这些字节的分组方式，取决于read和write调用的时间差，以及提供给in.read的缓冲区大小。可以认为TCP连接上发送的字节序列在某一个瞬间分成了3个FIFO序列。
>
>   1.sendQ：在发送端底层实现中缓存的字节，这些字节已经写入网络流，还没有被接收端收到。
>
>   2.RecvQ：在接收端底层实现中缓存的字节，等待分配到应用程序——即从输入流中读取数据.
>
>  3.Delivered：接收着从输入流中已经读取到的字节。

![image-20201102103247262](https://i.loli.net/2020/11/02/9fw26apximBDQL4.png)

![image-20201102103256072](https://i.loli.net/2020/11/02/rjTo8KVteanlGu7.png)

## 从网卡到socket

![image-20201102185133763](https://i.loli.net/2020/11/02/BMduky69veCSYaZ.png)

## 使用nc发送\r\n

```
\r是把光标移动到首字母前 CR Carriage Return 0x0D

\n是换行 LF Line Feed 0x0A
```

![image-20201105204006210](https://i.loli.net/2020/11/05/J3mjZ2GFWwxCDXL.png)

如果直接在nc中输入hello \r\n会出现上图结果

![image-20201105211508789](https://i.loli.net/2020/11/05/ghRxuq4sJEVyzkf.png)


