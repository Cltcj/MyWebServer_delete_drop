# 进程、线程的通信方式

## 进程间通信方式：

## 线程间通信方式





**信号量**

信号量：当多个进程同时访问系统上的某个资源的时候，比如同时写一个数据库的某条记录，或者同时修改某个文件，就需要考虑进程的同步问题，以确保任意时刻只有一个进程可以拥有对资源的独占式访问。通常，程序对共享资源的访问的代码只是很短的一段，但就是者一段代码引发了进程之间的竟态条件。我们称这段代码为关键代码段，或者临界区。对进程同步，也就是确保任一时刻只有一个进程能进入关键代码段。

Dijkstra提出的信号量（Semaphore）概念是并发编程领域迈出的重要一步。信号量是一种特殊的变量，它只能取自然数值并且支持两种操作：等待（wait）和信号（signal）。通常信号量的这两种操作更常用的称呼是P（传递）、V（释放）操作。

加入有信号量SV，则对它的P、V操作含义如下：
P（SV），如果SV的值大于0，就将它减1；如果SV的值为0，则挂起进程的执行。
V（SV），如果有其他进程因为等待SV而挂起，则唤醒之；如果没有，则将SV加1.


![image](https://user-images.githubusercontent.com/81791654/169467078-bdc3d2aa-cb67-48b0-a802-a0ba66d56c05.png)



三种线程同步机制：POSIX信号量、互斥量、条件变量



RAII “Resource Acquisition Is Initialization” 资源获取即初始化：

在构造函数中申请分配资源，在析构函数中释放资源。因为C++的语言机制保证了，当一个对象创建的时候，自动调用构造函数，当对象超出作用域的时候会自动调用析构函数。所以，在RAII的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定。

RAII的核心思想是将资源或者状态与对象的生命周期绑定，通过C++的语言机制，实现资源和状态的安全管理,**智能指针**是RAII最好的例子

## 三种专门用于线程同步的机制

线程同步机制包装类
===============
多线程同步，确保任一时刻只能有一个线程能进入关键代码段.
> * 信号量
> * 互斥锁
> * 条件变量

对上边介绍的三种锁进行封装，将锁的创建和销毁封装在类的构造和析构函数中，实现RAII机制
类中主要是Linux下三种锁进行封装，将锁的创建于销毁函数封装在类的构造与析构函数中，实现RAII[“Resource Acquisition is Initialization”，资源获取即初始化] 机制。

### POSIX信号量

POSIX信号量函数的名字都以sem_开头：

常用的POSIX信号量函数有下面5个：
![image](https://user-images.githubusercontent.com/81791654/168770197-7a2642e0-e98d-450c-b57e-94bf96f985cd.png)


头文件 `"#include <semaphore.h>"`

使用`sem_t i`;定义`sem_t`变量

```c
int sem_init(sem_t * sem, int pshared, unsigned int value);
```

用于初始化未命名信号量；pshared有0和非零两种情况，0代表当前进程的局部信号量，否则可在多个进程之间共享；value指定信号量的初始值。**注意!!!**不可初始化一个已经被初始化的信号量，这将导致不可预测的结果。

```c
int sem_destory(sem_t *sem);
```

用于销毁指定的信号量。**注意!!!**销毁一个正在被其他线程等待的信号量会导致不可预期的结果.

```c
int sem_wait(sem_t *sem);
```

用于将信号量的值减一，如果信号量值为0，`sem_wait`会阻塞，直到信号量非零。

```c
int sem_trywait(sem_t *sem);
```

和`sem_wait`类似，不过会立刻返回，是非阻塞版本的`sem_wait`，当信号量非零时，`sem_trywait`对信号量执行减一操作，信号量的值为0时，将返回-1，并设置`errno`为`EAGAIN`。

```c
int sem_post(sem_t *sem);
```

将信号量值加一，当信号量的值大于0时，其他正在阻塞的`sem_wait`将会有一个被唤醒。

以上函数成功返回0，失败返回-1并设置errno。

![image](https://user-images.githubusercontent.com/81791654/169470185-b8130097-1be8-4392-9a29-09d41e80092b.png)


### 互斥锁【互斥量】

&emsp;&emsp;互斥锁（也称互斥量）可以用于保护关键代码段，以确保其独占式的访问，这有点像一个二进制信号量。当进入关键代码段时，我们需要获得互斥锁并将其加锁，这等价于二进制信号量的P操作：当离开关键代码段时，我们需要对互斥锁解锁，以唤醒其他等待该互斥锁的线程，这等价于二进制信号量的V操作。

![image](https://user-images.githubusercontent.com/81791654/169471147-20b9b7e1-a816-427b-a37f-a6c45facb0b5.png)

头文件 `#include <pthread.h>`

结构体 `pthread_mutex_t mutex;`
```c
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
```
初始化互斥锁，mutexattr用于指定互斥锁属性，如果设置为NULL则使用默认属性。还可以用`pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;`来初始化互斥锁，实际上`PTHREAD_MUTEX_INITIALIZER`是吧互斥锁各个字段都初始化为0。
```c
int pthread_mutex_destory(pthread_mutex_t *mutex);
```
销毁互斥锁，释放其占用的内核资源，销毁一个已加锁的互斥锁会导致不可预测的行为。
```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
```
原子操作，给一个互斥锁加锁，如果目标已经锁上，则调用会阻塞。
```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
```
原子操作，与`pthread_mutex_lock`类似，区别在于会立刻返回，是非阻塞版本的`pthread_mutex_lock`。未加锁时执行加锁操作，已加锁的情况下会返回错误码EBUSY。[这里讨论的pthread_mutex_lock和pthread_mutex_trylock是针对普通锁，其他类型的锁会有>不同行为]
```c
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
原子操作，解锁，如果有其他线程等待该锁，会得到。

**补充：结构体pthread_mutexattr_t类型的设置和获取。**

头文件 `#include <pthread.h>`
```c
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
```
初始化互斥锁属性对象。
```c
int pthread_mutexattr_destory(pthread_mutex_t *attr);
```
销毁互斥锁属性对象。

`pshared`属性的设置和获取。
```c
pthread_mutexattr_getpshared(const pthread_mutexattr_t *attr, int *pshared);
```
获取attr互斥锁pshared属性，存储在pshared中。
```c
pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared);
```
设置互斥锁对象pshared属性。

`pshared`属性：

`PTHREAD_PROCESS_SHARED` 互斥锁可以被进程共享。
`PTHREAD_PROCESS_PRIVATE` 互斥锁只能和被锁的初始化线程隶属于同一进程的线程共享。
type属性的获取和设置
```c
int pthread_mutexattr_gettype(const pthread_mutexattr_t *attr, int *type);
```
获取type属性，存储在变量type中。
```c
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
```
设置type属性。

type属性：

`PTHREAD_MUTEX_NORMAL` 普通锁。默认类型，容易引发问题：如果对已经加锁的普通锁再加锁，会引发死锁；对已经解锁的普通锁解锁，对被其他进程加锁的普通锁解锁，会导致不可预期的结果。

`PTHREAD_MUTEX_ERRORCHECK` 检错锁。如果对已经加锁的检错锁再次加锁，加锁操作会返回EDEADLK；对已经解锁的检错锁解锁，或对已经其他进程加锁的检错锁解锁，解锁操作返回EPERM。[相当于解决了简单锁引发死锁和不确定结果的问题]

`PTHREAD_MUTEX_RECURSIVE` 嵌套锁。允许一个线程再释放锁之前多次加锁而不发生死锁。但如果其他线程要获取这个锁，当前锁的拥有者必须进程相应次数的解锁操作（有点类似上边信号量。）。如果对一个已经被其他线程加锁的嵌套锁解锁，或对已经解锁的嵌套锁再次解锁，解锁操作返回EPERM

`PTHREAD_MUTEX_DEFAULT` 默认锁。对加锁的默认锁再次加锁，对被其他线程加锁的默认锁解锁，对已经解锁的默认锁解锁，都会导致不可预测的行为。因为这种锁实现的时候可能被映射为上面三种锁之一。


## 条件变量

提供一种线程通知机制，当某个共享数据达到某个条件时，唤醒等待这个共享数据的线程。

![image](https://user-images.githubusercontent.com/81791654/169471654-6be39917-c8fc-42e0-92a9-293525135f7d.png)
![image](https://user-images.githubusercontent.com/81791654/169471690-170fafdb-6cda-4799-8c3c-700cfea01bcf.png)


头文件`#include <pthread.h>`

函数：
```c
int pthread_cond_init(pthread_cond_t *cond, const pthread_condattr_t *cond_attr);
```
初始化条件变量，`cond_attr`参数指定条件变量属性。如果设置为NULL则表示使用默认属性。也可以使用`pthread_cond_t cond = PTHREAD_COND_INITIALIZER`;初始化一个条件变量，`PTHREAD_COND_INITIALIZER`实际上是把条件变量的各个字段初始化为0。
```c
int pthread_cond_destory(pthread_cond_t *cond);
```
用于销毁条件变量，以释放其占用的内核资源。
```c
int pthread_cond_broadcast(pthread_cond_t *cond);
```
以广播的形式唤醒所有等待cond的线程。
```c
int pthread_cond_signal(pthread_cond_t *cond);
```
只唤醒一个等待的线程，唤醒哪个取决于线程的优先级和调度策略。
```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
```
用于等待目标条件变量，mutex用于保护条件变量的互斥锁。一般而言，流程是：先锁住，然后调用`pthread_cond_wait`，`pthread_cond_wait`会把锁解开，放在等待队列中（这两部操作是原子操作）；如果被唤醒，重新锁上，返回。这样做的好处是，不用每次都【上锁，询问条件变量满不满足，不满足重新解锁】这种操作，条件变量满足之后自动返回。
```c
int pthread_cond_timedwait(pthread_cond_t *cond, pthread_mutex_t *mutex, const struct timespec *abstime)
```
和上边的等待类似，但是在给定时刻前条件没有满足，则返回ETIMEDOUT，结束等待。

以上函数成功返回0，失败返回-1并设置errno。

补充：`pthread_condattr_t`条件变量属性结构体结构体

属性：

进程共享属性【pshared属性】
时钟属性
函数:

头文件`#include <pthread.h>`

初始化和反初始化
```c
int pthread_condattr_init(pthread_condattr_t *attr);
```
功能：对条件变量属性结构体初始化。调用此函数之后，条件变量属性结构体的属性都是系统默认值，如果想要设置其他属性，还需要调用不同的函数进行设置
```c
int pthread_condattr_destroy(pthread_condattr_t* attr);
```
功能：对条件变量属性结构体反初始化（销毁）。只反初始化，不释放内存

pthread属性的获取与设置
```c
int pthread_condattr_getshared(const pthread_condattr_t* restrict attr,int* restrict pshared);
```
获取条件变量的进程共享属性
```c
int pthread_condattr_setshared(pthread_condattr_t* attr,int pshared);
```
设置条件变量的进程共享属性

`pshared`属性：

`PTHREAD_PROCESS_SHARED` 互斥锁可以被进程共享。
`PTHREAD_PROCESS_PRIVATE` 互斥锁只能和被锁的初始化线程隶属于同一进程的线程共享。
clock时钟属性的获取与设置
```c
int pthread_condattr_getclock(const pthread_condattr_t* restrict attr,clockid_t *restrict clock_id);
```
此函数获取可被用于pthread_cond_timedwait函数的时钟ID。pthread_cond_timedwait函数使用前需要用pthread_condattr_t对条件变量进行初始化
```c
int pthread_condattr_setclock(pthread_condattr_t* attr,clockid_t clock_id);
```
此函数用于设置`pthread_cond_timewait`函数使用的时钟ID

clock属性：

>`CLOCK_REALTIME`:实时系统时间

>`CLOCK_MONOTONIC`:不带负跳数的实时系统时间

>`CLOCK_PROCESS_CPUTIME_ID`: 调用进程的CPU时间

>`CLOCK_THREAD_CPUTIME_ID`: 调用线程的CPU时间
