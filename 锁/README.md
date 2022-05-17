RAII “Resource Acquisition Is Initialization” 资源获取即初始化：

在构造函数中申请分配资源，在析构函数中释放资源。因为C++的语言机制保证了，当一个对象创建的时候，自动调用构造函数，当对象超出作用域的时候会自动调用析构函数。所以，在RAII的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定。

RAII的核心思想是将资源或者状态与对象的生命周期绑定，通过C++的语言机制，实现资源和状态的安全管理,***智能指针***是RAII最好的例子



线程同步机制包装类
===============
多线程同步，确保任一时刻只能有一个线程能进入关键代码段.
> * 信号量
> * 互斥锁
> * 条件变量

对上边介绍的三种锁进行封装，将锁的创建和销毁封装在类的构造和析构函数中，实现RAII机制
类中主要是Linux下三种锁进行封装，将锁的创建于销毁函数封装在类的构造与析构函数中，实现RAII[“Resource Acquisition is Initialization”，资源获取即初始化] 机制。


三种线程同步机制：POSIX信号量、互斥量、条件变量


## POSIX信号量

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

## 互斥锁【互斥量】

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

