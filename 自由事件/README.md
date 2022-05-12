# libevent
##  libevent介绍

1 事件驱动, 高性能, 轻量级, 专注于网络
2 源代码精炼, 易读 
3 跨平台
4 支持多种I/O多路复用技术, 如epoll select poll等
5 支持I/O和信号等事件

## libevent的安装

登录官方网站: http://libevent.org, 查看相关信息

libevent源码下载主要分2个大版本：

1.1.4.x 系列, 较为早期版本, 适合源码学习

2.2.x系列, 较新的版本, 代码量比1.4版本多很多, 功能也更完善。

**第一步: 解压`libevent-2.0.22-stable.tar.gz` **

>解压: `tar -zxvf libevent-2.0.22-stable.tar.gz`
>`cd`到`libevent-2.0.22-stable`目录下, 查看`README`文件, 该文件里描述了安装的详细步骤, 可参照这个文件进行安装.

**第二步: 进入源码目录:**

执行配置`./configure`, 检测安装环境, 生成`makefile`.
执行`./configure`的时候也可以指定路径, `./configure --prefix=/usr/xxxxx`, 这样就可以安装到指定的目录下, 但是这样在进行源代码编译的时候需要指定用-I头文件的路径和用-L库文件的路径. 若默认安装不指定--prefix, 则会安装到系统默认的路径下, 编译的时候可以不指定头文件和库文件所在的路径.
执行`make`命令编译整个项目文件.
通过执行`make`命令, 会生成一些库文件(动态库和静态库)和可执行文件.
执行`sudo make install`进行安装
安装需要root用户权限, 这一步需要输入当前用户的密码
执行这一步, 可以将刚刚编译成的库文件和可执行文件以及一些头文件拷贝到/usr/local目录下:
----头文件拷贝到了`/usr/local/include`目录下;
----库文件拷贝到了`/usr/local/lib`目录下.


## libevent的核心实现:

在linux上, 其实质就是epoll反应堆。

libevent是事件驱动, epoll反应堆也是事件驱动, 当要监测的事件发生的时候, 就会调用事件对应的回调函数, 执行相应操作. 特别提醒: 事件回调函数是由用户开发的, 但是，不是由用户显示去调用的, 而是由libevent去调用的.

从官网http://libevent.org上下载安装文件之后, 将安装文件上传到linux系统上;源码包的安装,以2.0.22版本为例,在官网可以下载到源码包libevent-2.0.22-stable.tar.gz, 安装步骤与第三方库源码包安装方式基本一致。

## libevent库的使用

进入到libevent-2.0.22-stable/sample下, 可以查看一些示例源代码文件.

使用libevent库编写代码在编译程序的时候需要指定库名:-levent;

安装文件的libevent库文件所在路径:libevent-2.0.22-stable/.libs;

编写代码的时候用到event.h头文件, 或者直接参考sample目录下的源代码文件也可以.

```cpp
#include <event2/event.h>
编译源代码文件(以hello-world.c文件为例)
gcc hello-world.c -levent
```
由于安装的时候已经将头文件和库文件拷贝到了系统头文件所在路径`/usr/local/include`和系统库文件所在路径`/usr/local/lib`, 所以这里编译的时候可以不用指定`-I`和`-L`.

编译示例代码`hello-world.c`程序:

`gcc -o hello-world hello-world.c -levent`

测试: 在另一个终端窗口进行测试, 输入: nc `127.1 9995`, 然后回车立刻显示`Hello, World!`字符串.

## libevent的使用

### libevent的地基-event_base

使用libevent 函数之前需要分配一个或者多个 event_base 结构体, 每个event_base结构体持有一个事件集合, 可以检测以确定哪个事件是激活的, event_base结构相当于epoll红黑树的树根节点, 每个event_base都有一种用于检测某种事件已经就绪的 “方法”(回调函数)

通常情况下可以通过event_base_new函数获得event_base结构。  

下面介绍一些常用函数:

相关函数说明:

```c
struct event_base *event_base_new(void);    //event.h的L:337
  函数说明: 获得event_base结构
  参数说明: 无
  返回值: 
成功返回event_base结构体指针;
失败返回NULL;
```

```c
void event_base_free(struct event_base *);   //event.h的L:561
	函数说明: 释放event_base指针
```

```c
int event_reinit(struct event_base *base);  //event.h的L:349
	函数说明: 如果有子进程, 且子进程也要使用base, 则子进程需要对event_base重新初始化, 此时需要调用event_reinit函数.
函数参数: 由event_base_new返回的执行event_base结构的指针
返回值: 成功返回0, 失败返回-1
```

对于不同系统而言, event_base就是调用不同的多路IO接口去判断事件是否已经被激活, 对于linux系统而言, 核心调用的就是epoll, 同时支持poll和select.


**查看libevent支持的后端的方法有哪些:**

```c
const char **event_get_supported_methods(void);
函数说明: 获得当前系统(或者称为平台)支持的方法有哪些
参数: 无
返回值: 返回二维数组, 类似与main函数的第二个参数**argv.
```

```c
const char * event_base_get_method(const struct event_base *base);
函数说明: 获得当前base节点使用的多路io方法
函数参数: event_base结构的base指针.
返回值: 获得当前base节点使用的多路io方法的指针
```

## 等待事件产生-循环等待event_loop

&emsp;&emsp;libevent在地基打好之后, 需要等待事件的产生, 也就是等待事件被激活, 所以程序不能退出, 对于epoll来说, 我们需要自己控制循环, 而在libevent中也给我们提供了API接口, 类似where(1)的功能. 

函数如下：

```c
int event_base_loop(struct event_base *base, int flags);   //event.h的L:660
/*
 * 函数说明: 进入循环等待事件
 * 参数说明:
 * base: 由event_base_new函数返回的指向event_base结构的指针
 * flags的取值：
 * #define EVLOOP_ONCE	0x01
 * 				只触发一次, 如果事件没有被触发, 阻塞等待
 * #define EVLOOP_NONBLOCK	0x02
 * 				非阻塞方式检测事件是否被触发, 不管事件触发与否, 都会立即返回.
 */
```

**上面这个函数一般不用, 而大多数都调用libevent给我们提供的另外一个API：**

```c
int event_base_dispatch(struct event_base *base);   //event.h的L:364

/*
 * 函数说明: 进入循环等待事件
 * 参数说明:由event_base_new函数返回的指向event_base结构的指针
 * 调用该函数, 相当于没有设置标志位的event_base_loop。程序将会一直运行, 直到没有需要检测的事件了, 或者被结束循环的API终止。
 */
int event_base_loopexit(struct event_base *base, const struct timeval *tv);
int event_base_loopbreak(struct event_base *base);
struct timeval {
	long    tv_sec;                    
	long    tv_usec;            
};
```

两个函数的区别是如果正在执行激活事件的回调函数, 那么`event_base_loopexit`将在事件回调执行结束后终止循环（如果tv时间非NULL, 那么将等待tv设置的时间后立即结束循环）, 而`event_base_loopbreak`会立即终止循环。

## 使用libevent库的步骤：

1 创建根节点--event_base_new

2 设置监听事件和数据可读可写的事件的回调函数

设置了事件对应的回调函数以后, 当事件产生的时候会自动调用回调函数

3 事件循环--event_base_dispatch

相当于while(1), 在循环内部等待事件的发生,  若有事件发生则会触发事件对应的回调函数。

4 释放根节点--event_base_free

释放由event_base_new和event_new创建的资源, 分别调用event_base_free和event_free函数.

## 事件驱动-event

事件驱动实际上是libevent的核心思想, 本小节主要介绍基本的事件event。

主要的状态转化：

![image](https://user-images.githubusercontent.com/81791654/167972257-a4bfd4d0-255e-4e93-9b8d-40486dbe4731.png)


主要几个状态：

无效的指针: 此时仅仅是定义了 `struct event *ptr`

非未决：相当于创建了事件, 但是事件还没有处于被监听状态, 类似于我们使用epoll的时候定义了struct epoll_event ev并且对ev的两个字段进行了赋值, 但是此时尚未调用epoll_ctl对事件上树.

未决：就是对事件开始监听, 暂时未有事件产生。相当于调用epoll_ctl对要监听的事件上树, 但是没有事件产生.

激活：代表监听的事件已经产生, 这时需要处理, 相当于调用epoll_wait函数有返回, 当事件被激活以后,  libevent会调用该事件对应的回调函数

libevent的事件驱动对应的结构体为struct event, 对应的函数在图上也比较清晰, 下面介绍一下主要的函数:

```c
typedef void (*event_callback_fn)(evutil_socket_t fd, short events, void *arg);
struct event *event_new(struct event_base *base, evutil_socket_t fd, short events, event_callback_fn cb, void *arg);

/*
函数说明: event_new负责创建event结构指针, 同时指定对应的地基base, 		  还有对应的文件描述符, 事件, 以及回调函数和回调函数的参数。
参数说明：
base: 对应的根节点--地基
fd: 要监听的文件描述符
events:要监听的事件
	#define  EV_TIMEOUT    0x01   //超时事件
	#define  EV_READ       0x02    //读事件
	#define  EV_WRITE      0x04    //写事件
	#define  EV_SIGNAL     0x08    //信号事件
	#define  EV_PERSIST     0x10    //周期性触发
	#define  EV_ET         0x20    //边缘触发, 如果底层模型支持设置则有效, 若不支持则无效.
	若要想设置持续的读事件则： EV_READ | EV_PERSIST
*/
```

cb 回调函数, 原型如下：

`typedef void (*event_callback_fn)(evutil_socket_t fd, short events, void *arg);`

注意: 回调函数的参数就对应于event_new函数的fd, event和arg

```c
#define evsignal_new(b, x, cb, arg)             \                                                                    
      event_new((b), (x), EV_SIGNAL|EV_PERSIST, (cb), (arg))
```


```c
int event_add(struct event *ev, const struct timeval *timeout);
函数说明: 将非未决态事件转为未决态, 相当于调用epoll_ctl函数(EPOLL_CTL_ADD), 开始监听事件是否产生, 相当于epoll的上树操作.
参数说明：
	ev: 调用event_new创建的事件
timeout: 限时等待事件的产生, 也可以设置为NULL, 没有限时。
```

```c
int event_del(struct event *ev);
函数说明: 将事件从未决态变为非未决态, 相当于epoll的下树（epoll_ctl调用			EPOLL_CTL_DEL操作）操作。
参数说明: ev指的是由event_new创建的事件.
```

```c
void event_free(struct event *ev);
函数说明: 释放由event_new申请的event节点。
```
