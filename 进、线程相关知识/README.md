# 进、线程相关知识

&emsp;&emsp;进程是资源调度的基本单位，运行一个可执行程序会创建一个或多个进程，进程就是运行起来的可执行程序。线程是程序执行的基本单位，是轻量级的进程。每个进程中都有唯一的主线程，且只能有一个，主线程和进程是相互依存的关系，主线程结束进程也会结束。

&emsp;&emsp;线程是轻量级的进程，启动速度快、且系统开销小，线程使用有一定的难度，需要处理数据一致性的问题，同一线程共享的有堆、全局变量、静态变量、指针、引用、文件等，自己独享栈，且有唯一的线程id。

&emsp;&emsp;理论上，一个进程的可用虚拟空间是2G,默认情况下，线程的栈的大小是1MB，所以理论上最多只能创建2048个线程，如果要创建多于2048的话，需要修改编译器设置。因此，一个进程可用创建的线程数由可用虚拟空间和线程栈的大小共同决定，只要虚拟空间足够新线程就能创建成功。如果要创建2K以上的线程，需要减小线程栈的大小就能实现，但过多的线程会导致大量的时间浪费在线程切换上，降低效率。一般也不会创建如此多的线程。

**进程调度算法**

1、先来先服务

非抢占式调度算法，按照请求的顺序进行调度
有利于长作业，不利于短作业，因为短作业必须一直等待前面的长作业执行完毕才能执行，而长作业又需要执行很长的时间，造成短作业会等很久。

2、短作业优先

非抢占式调度算法，按估计运行时间最短的顺序进行调度
长作业有可能会饿死，处于一直等待短作业执行完毕的状态。因为如果一直有短作业到来，那么长作业永远得不到调度。

3、最短剩余时间优先

最短作业优先的抢占式版本，按照剩余运行时间的顺序进行调度，当一个新的作业到达时，其整个运行时间与当前进程的剩余时间作比较。
如果新进程需要的时间更少，则挂起当前进程，运行新的进程，否则新的进程等待。

4、时间片轮转

将所有就绪进程按FCFS的原则排成一个队列，每次调度时，把CPU时间分配给首进程，该进程可以执行一个时间片。
当时间片用完时，由计时器发送时钟中断，调度程序便停止该进程的执行，并将它送往就绪队列的末尾，同时继续把CPU时间分配给队首的进程。
时间片轮转的效率和时间片大小有关：
时间片太小，进程切换太频繁，在进程切换上花费太多时间。
时间片太长，实时性无法保证。

5、优先级调度

为每个进程分配一个优先级，按优先级进行调度。
为了防止低优先级的进程永远得不到调度，可以随着时间的推移增加等待进程的优先级。

6、多级反馈队列

一个进程需要执行100个时间片，如果采用时间片轮转，就需要100次交换。
多级队列是为这种需要连续执行多个时间片的进程考虑，它设置了多个队列，每个队列时间片大小都不同，例如1，2，4，8...进程在第一个队列没执行完，就会被移到下一个队列。
这种方式下，之前的进程只需要被交换7次，每个队列优先权也不同，最上面的优先权最高。因此只有上一个队列没有进程在排队，才能调度当前队列上的进程。
它是4和5的结合。


Linux下进程通信的方式

**管道**：

无名管道：管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程之间使用。进程的亲缘关系是指父子（兄弟）进程关系。

有名管道（FIFO文件，借助文件系统）：有名管道也是半双工通信，但是允许在没有亲缘关系的进程之间使用，管道是先进先出的通信方式。

**共享内存**：

共享内存就是映射一段能被其它进程所访问的内存，这段共享内存由一个进程创建，但可以由多个进程访问，共享内存是最快的IPC方式，它是针对其它进程间通信方式运行效率低而专门设计的。它往往与信号量，配合使用来实现进程间的同步和通信。

**消息队列**：消息队列是有消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能只能承载无格式字节流以及缓冲区大小受限等缺点。

**套接字**：适用于不同机器间进程通信，在本地也可作为两个进程通信的方式。

**信号**：用于通知接收某个事件已经发生，比如按下CTRL+C就是信号

**信号量**：信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，实现进程、线程的对临界区的同步及互斥访问。

**进程同步的四种方法:**

1、临界区
对临界资源进行访问的那段代码称为临界区。
为了互斥访问临界资源，每个进程在进入临界区之前，需要先进行检查。

2、同步和互斥
同步：多个进程因为合作产生的直接制约关系，使得进程有一定的先后执行关系
互斥：多个进程在同一时刻只有一个进程能进入临界区。

3、信号量

信号量是一个整型变量，可以对其执行down和up操作，也就是常见的P、V操作。
Down：如果信号量大于0，执行-1操作，如果信号量等于0，进程睡眠，等待信号量大于0
Up：对信号量执行+1操作，唤醒睡眠的进程让其完成down操作。
Down和up操作需要被设计成原语，不可分割，通常做法是在执行这些操作的时候屏蔽中断。
如果信号量的取值只能为0或1，那么就成为互斥量（Mutex），0表示临界区已经加锁，1表示临界区解锁。

**使用信号量实现生产者-消费者问题**

>问题描述使用一个缓冲区来保存物品，只有缓冲区没有满，生产者才可以放入物品;只有缓冲区不为空;消费者才能拿走物品。

因为临界区属于临界资源

4、管程

使用信号量机制实现的生产者消费者问题需要客户端代码做很多控制，而管程把控制的代码独立出来，不仅不容易出错，也使得客户端代码调用更加容易。

管程有一个重要特性：在一个时刻只能有一个进程使用管程。进程在无法继续执行的时候不能一直占用管程，否则其它进程永远不能使用管程。

管程引入了条件变量以及相关的操作：wait()和signal()来实现同步操作。对条件变量执行wait()操作会导致调用进程阻塞，把管程让出来给另一个进程持有。signal()操作用于唤醒被阻塞的进程。

线程通信方式：
 
* Linux:
信号：类似进程间的信号处理
锁机制：互斥锁、读写锁、自旋锁
条件变量：使用通知的方式解锁，与互斥锁配合使用
信号量：包括无名线程信号量和命名线程信号量

* Windows

全局变量：需要多个线程来访问一个全局变量时，通常我们会在这个全局变量前加上volatile声明，以防编译器对此变量进行优化。

Message消息机制：常用的Message通信的接口主要有两个：PostMessage 和 PostThreadMessage，PostMessage为线程向主窗口发送消息。而PostThreadMessage是任意两个线程之间的通信接口。

CEvent对象：CEvent为MFC中的一个对象，可以通过对CEvent的触发状态进行改变，从而实现线程间的通信和同步，这个主要是实现线程直接同步的一种方法。

虚拟内存的目的是什么？

**虚拟内存的目的是为了让物理内存扩充为更大的逻辑内存，从而让程序获得更多的可用内存。**

为了更好的管理内存，操作系统将内存抽象成地址空间。每个程序拥有自己的地址空间，这个地址空间被分割为多个块，每一块称为一页。

这些页被映射到物理内存，但不需要映射到连续的物理内存，也不需要所有页都必须在物理内存中。当程序引用到不在物理内存中的页时，由硬件执行必要的映射，将缺失的部分装入物理内存并重新执行失败的指令。

从上面的描述中可以看出，虚拟内存允许不用将地址空间中的每一页都映射到物理内存，也就是说一个程序不需要全部全部调入内存就可以允许，这使得有限的内存运行大程序成为可能。











[vim操作](https://blog.csdn.net/CltCj/article/details/123596776)

[gcc编译](https://blog.csdn.net/CltCj/article/details/123603499)

[makefile](https://blog.csdn.net/CltCj/article/details/123613421)

[gdb调式](https://blog.csdn.net/CltCj/article/details/123616833)

[文件操作相关函数](https://blog.csdn.net/CltCj/article/details/123623500)

[进程间通信1](https://blog.csdn.net/CltCj/article/details/123686100)


[进程间通信2](https://blog.csdn.net/CltCj/article/details/123719614)
