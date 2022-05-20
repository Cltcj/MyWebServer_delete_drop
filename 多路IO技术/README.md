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

&emsp;&emsp;epoll是Linux特有的I/O复用函数。epoll会使用一组函数来完成任务，而不是单个函数。epoll会把文件描述符上的事件放在内核里的一个事件表中，这代表它无须像select和poll












多路IO技术: select, 同时监听多个文件描述符, 将监控的操作交给内核去处理,数据类型fd_set: 文件描述符集合--本质是位图(关于集合可联想一个信号集sigset_t)

```c
int select(int nfds, fd_set * readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```


