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
   函数说明: 进入循环等待事件
参数说明:
base: 由event_base_new函数返回的指向event_base结构的指针
flags的取值：
#define EVLOOP_ONCE	0x01
				只触发一次, 如果事件没有被触发, 阻塞等待
#define EVLOOP_NONBLOCK	0x02
				非阻塞方式检测事件是否被触发, 不管事件触发与否, 都会					立即返回.
```

**上面这个函数一般不用, 而大多数都调用libevent给我们提供的另外一个API：**





```c
const char * event_base_get_method(const struct event_base *base);
函数说明: 获得当前base节点使用的多路io方法
函数参数: event_base结构的base指针.
返回值: 获得当前base节点使用的多路io方法的指针
```

```c
const char * event_base_get_method(const struct event_base *base);
函数说明: 获得当前base节点使用的多路io方法
函数参数: event_base结构的base指针.
返回值: 获得当前base节点使用的多路io方法的指针
```
