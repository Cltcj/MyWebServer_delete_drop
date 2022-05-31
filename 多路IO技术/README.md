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



## epoll 系列系统调用

![image](https://user-images.githubusercontent.com/81791654/170219359-1913b1ed-886c-439a-a95a-63542a6c7556.png)

**epoll_create函数**

```cpp
#include <sys/epoll.h>
int epoll_create(int size)
```

创建一个指示epoll内核事件表的文件描述符，该描述符将用作其他epoll系统调用的第一个参数，size不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符将用其他所有 epoll系统调用的第一个参数，以指定要访问的内核事件表。

下面的函数用来操作 epoll 的内核事件表：

**epoll_ctl函数**

```cpp
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```

该函数用于操作内核事件表监控的文件描述符上的事件：注册、修改、删除

* epfd：为epoll_creat的句柄

* op：表示动作，用3个宏来表示：

EPOLL_CTL_ADD (注册新的fd到epfd)，

EPOLL_CTL_MOD (修改已经注册的fd的监听事件)，

EPOLL_CTL_DEL (从epfd删除一个fd)；

* fd是要操作的文件描述符

* op参数则指定操作类型。操作类型有如下3种：

* event：告诉内核需要监听的事件

上述event是epoll_event结构体指针类型，表示内核所监听的事件，具体定义如下：

```cpp
struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```
events描述事件类型，其中epoll事件类型有以下几种

* EPOLLIN：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）

* EPOLLOUT：表示对应的文件描述符可以写

* EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）

* EPOLLERR：表示对应的文件描述符发生错误

* EPOLLHUP：表示对应的文件描述符被挂断；

* EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)而言的

* EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

![image](https://user-images.githubusercontent.com/81791654/170411993-90fc9414-6f0f-44dc-a88c-fb7ce8c6c447.png)

**epoll_wait函数**

```cpp
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

该函数用于等待所监控文件描述符上有事件的产生，成功返回就绪的文件描述符个数，失败返回-1并设置errno

epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样即用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

events：用来存内核得到事件的集合，

maxevents：告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，但必须大于0

timeout：是超时时间，和poll中的timeout参数相同。

* -1：阻塞

* 0：立即返回，非阻塞

* >0：指定毫秒

返回值：成功返回有多少文件描述符就绪，时间到时返回0，出错返回-1

### LT 和 ET模式

![image](https://user-images.githubusercontent.com/81791654/170442531-4c46916f-d849-48c0-9d1e-c99c42d8f8b3.png)

**LT水平触发模式**

epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序可以不立即处理该事件。

当下一次调用epoll_wait时，epoll_wait还会再次向应用程序报告此事件，直至被处理

**ET边缘触发模式**

epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序必须立即处理该事件

必须要一次性将数据读取完，使用非阻塞I/O，读取到出现eagain

### EPOLLONESHOT

![image](https://user-images.githubusercontent.com/81791654/170443912-62ceed4d-c4c5-462a-8148-7ab0c9657626.png)

一个线程读取某个socket上的数据后开始处理数据，在处理过程中该socket上又有新数据可读，此时另一个线程被唤醒读取，此时出现两个线程处理同一个socket

我们期望的是一个socket连接在任一时刻都只被一个线程处理，通过epoll_ctl对该文件描述符注册epolloneshot事件，一个线程处理socket时，其他线程将无法处理，当该线程处理完后，需要通过epoll_ctl重置epolloneshot事件

![image](https://user-images.githubusercontent.com/81791654/170444562-418206bc-48b3-42f3-8289-228e5bbe9800.png)

