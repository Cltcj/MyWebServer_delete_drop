# 两种高效的事件处理模式

&emsp;&emsp;随着网络设计模式的兴起，Reactor和Proactor事件处理模式应运而生。同步I/O模式通常用于实现Reactor模式，异步I/O模型则用于实现Proactor模式。

&emsp;&emsp;异步I/O：内核执行读写操作并触发读写完成事件。程序没有阻塞阶段。








## Reactor模式

&emsp;&emsp;在web服务器开发中，有2种常见的架构，基于线程的架构和事件驱动的架构。初期使用一个连接用一个线程来处理，这样显然对于高并发连接而言会使得线程创建开销很大，而改进方法使用多进程来处理每个请求，这样单个请求出问题不会影响到其它请求，但进程切换很慢且内存消耗很大。为了优化线程数量以获得最佳的整体性能，同时为了避免线程创建/销毁的开销，通常在实际应用中 , 会在一个数量有限的阻塞队列上使用一个单独的线程用于分发（epoll），事件发生时，再从一个线程池（thread pool）中分发工作线程给一个连接。

## Reactor模式

![image](https://user-images.githubusercontent.com/81791654/169034418-29316458-6905-4442-9eb0-97ab740bc26e.png)
![image](https://user-images.githubusercontent.com/81791654/169034814-371a18b0-4c34-42e0-944a-d3af20097dad.png)

## Proactor模式

![image](https://user-images.githubusercontent.com/81791654/169035047-69ce1faf-621e-4765-83e5-d2de953aa555.png)

![image](https://user-images.githubusercontent.com/81791654/169035121-ed39e0e7-0b65-4fa2-9169-1c3448ffdf5f.png)














（1） Reactor模式组成及经典实现方案

Reactor是一种主流的网络编程模式，是事件驱动架构的一种实现.通常,它使用一个线程进行事件循环,阻塞在产生事件的资源上,当事件到来时分发给相关的handlers和callbacks.比如像公司里的接线员将电话转接给合适的人。
所以Reactor有两个组成部分

Reactor（分发器）
Reactor在独立的线程中运行,它的作用是对IO事件做出响应并将它们分发给合适的处理程序；比如公司里的接线员
Handlers（处理器）
Handler对I/O事件执行实际的工作，Handler类似于客户实际想打给的人。
reactor将I/O事件分发给合适的handler，Handlers执行非阻塞的操作
最终 Reactor 模式有这三种典型的实现方案（[参考资料1](https://juejin.cn/post/6844903855801679879)，[参考资料2]()）：

* **单 Reactor+单进程 / 线程** (没有进程间通信，没有进程竞争,但只有一个进程，无法发挥多核 CPU 的性能)

* **单 Reactor+多线程（线程池）**（单 Reator 多线程方案能够充分利用多核多 CPU 的处理能力，但Reactor 承担所有事件的监听和响应，只在主线程中运行，瞬间高并发时会成为性能瓶颈且多线程数据共享和访问比较复杂，多线程比多进程简单）

* **多 Reactor+多进程 / 线程**（Nginx 采用的是多 Reactor 多进程）


（2） Reactor优势与不足：

* 优势：
1）响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；
2）编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；
3）可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；
4）可复用性，reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

* 不足：
1）相比传统的简单模型，Reactor增加了一定的复杂性，因而有一定的门槛，并且不易于调试。
2）Reactor模式需要底层的Synchronous Event Demultiplexer支持，比如Java中的Selector支持，操作系统的select系统调用支持，如果要自己实现Synchronous Event Demultiplexer可能不会有那么高效。
3） Reactor模式在IO读写数据时还是在同一个线程中实现的，即使使用多个Reactor机制的情况下，那些共享一个Reactor的Channel如果出现一个长时间的数据读写，会影响这个Reactor中其他Channel的相应时间，比如在大文件传输时，IO操作就会影响其他Client的相应时间，因而对这种操作，使用传统的Thread-Per-Connection或许是一个更好的选择，或则此时使用Proactor模式。


（3） 采用非阻塞的I/O+事件驱动epoll+线程池来实现单reactor 模式

非阻塞的I/O+事件驱动epoll+线程池来实现单reactor 模式的工作流程如下：

（1）主线程向epoll内核事件表内注册socket上的可读就绪事件。

（2）主线程调用epoll_wait()等待socket上有数据可读

（3）当socket上有数据可读，epoll_wait 通知主线程。主线程从socket可读事件放入请求队列。

（4）睡眠在请求队列上的某个可读工作线程被唤醒，从socket上读取数据，处理客户的请求。然后向 epoll内核事件表里注册写的就绪事件

（5）主线程调用epoll_wait()等待数据可写 。

![image](https://user-images.githubusercontent.com/81791654/167974867-db84b62b-4e8c-4238-b5b5-bbdd7895c0ac.png)

（1）线程池优势：
① 通过重用现有的线程而不是创建新线程，可以在处理多个请求时分摊在线程创建和销毁过程产生的巨大开销。
② 当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，从而提高了响应性。
③ 通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态。同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。

（2）epoll：
① 支持一个进程打开大数目的socket描述符(FD)
② IO效率不随FD数目增加而线性下降，因为获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。
③ epoll除了提供select/poll 那种IO事件的电平触发（Level Triggered）外，还提供了边沿触发（Edge Triggered），这就使得用户空间程序有可能缓存IO状态，减少epoll_wait/epoll_pwait的调用，提高应用程序效率。使用mmap加速内核与用户空间的消息传递。



https://blog.csdn.net/qq_39751437/article/details/105446909
