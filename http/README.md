## epoll 系列系统调用

![image](https://user-images.githubusercontent.com/81791654/170219359-1913b1ed-886c-439a-a95a-63542a6c7556.png)

**epoll_create函数**

```cpp
#include <sys/epoll.h>
int epoll_create(int size)
```

创建一个指示epoll内核事件表的文件描述符，该描述符将用作其他epoll系统调用的第一个参数，size不起作用。

**epoll_ctl函数**

```cpp
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```
该函数用于操作内核事件表监控的文件描述符上的事件：注册、修改、删除

* epfd：为epoll_creat的句柄

* op：表示动作，用3个宏来表示：

* * EPOLL_CTL_ADD (注册新的fd到epfd)，

EPOLL_CTL_MOD (修改已经注册的fd的监听事件)，

EPOLL_CTL_DEL (从epfd删除一个fd)；

event：告诉内核需要监听的事件

上述event是epoll_event结构体指针类型，表示内核所监听的事件，具体定义如下：













## HTTP—Hyper Text Transfer Protocol（超文本传输协议）

HTTP（HyperText Transfer Protocol，超文本传输协议）是一种用于分布式、协作式和超媒体信息系统的应用层协议。HTTP 是万维网的数据通信的基础。

HTTP_CODE含义
表示HTTP请求的处理结果，在头文件中初始化了八种情形

* NO_REQUEST：请求不完整，需要继续读取请求报文数据

* GET_REQUEST：获得了完整的HTTP请求

* NO_RESOURCE：请求资源不存在

* BAD_REQUEST：HTTP请求报文有语法错误或请求资源为目录

* FORBIDDEN_REQUEST：请求资源禁止访问，没有读取权限

* FILE_REQUEST：请求资源可以正常访问

* INTERNAL_ERROR：服务器内部错误



