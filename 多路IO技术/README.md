# 多路IO技术

客户端程序要同时处理多个socket。

客户端程序要同时处理用户输入和网络连接。

TCP服务器要同时处理监听 socket 和连接 socket。这是I/O复用最多的场合。

服务器要同时处理TCP请求和UDP请求。

服务器要同时监听多个端口，或处理多种服务。

I/O复用虽然能同时监听多个文件描述符，但它本身是阻塞的。

Linux下实现I/O复用的系统调用主要有select、poll和epoll

## select系统复用

&emsp;&emsp;select系统调用的用途是：在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常等事件。

![image](https://user-images.githubusercontent.com/81791654/169495554-25f6778d-13ff-4033-a3a0-f644348a9e43.png)

## poll 

&emsp;&emsp;

![image](https://user-images.githubusercontent.com/81791654/169496459-9efa13d2-9065-4a63-9f1c-81f4fe146782.png)

## epoll 

**内核事件表**

&emsp;&emsp;epoll是Linux特有的I/O复用函数。epoll会使用一组函数来完成任务，而不是单个函数。epoll会把文件描述符上的事件放在内核里的一个事件表中，这代表它无须像select和poll那样每次调用都要重复传入文件描述符集或事件集。但epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表。这个文件描述符使用如下epoll_create函数来创建：

![image](https://user-images.githubusercontent.com/81791654/169548261-3f36c27a-dff4-499f-b360-ca70fc7d09ca.png)

下面的函数用来操作epoll的内核事件表：
![image](https://user-images.githubusercontent.com/81791654/169548428-f1f3ed79-d4e2-4a3f-8bc1-ce350ecf6761.png)

**epoll_wait函数**
![image](https://user-images.githubusercontent.com/81791654/169548795-4f886ae2-be22-438b-a987-421c33e925c0.png)

&emsp;&emsp;epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样既用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

**LT和ET模式**

&emsp;&emsp;epoll对文件描述符的操作有两种模式：LT（电平触发）模式和ET（边沿触发）模式。LT是默认的工作模式，这种模式下epoll相当于一个效率较高的poll。当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符。ET模式是epoll的高效工作模式。

![image](https://user-images.githubusercontent.com/81791654/169551534-2f6a29e5-3628-415b-b26d-53920873c1bc.png)













多路IO技术: select, 同时监听多个文件描述符, 将监控的操作交给内核去处理,数据类型fd_set: 文件描述符集合--本质是位图(关于集合可联想一个信号集sigset_t)

```c
int select(int nfds, fd_set * readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```


