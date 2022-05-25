# ThreadPool

动态创建进程（或线程）是比较耗费时间的，这将导致较慢的客户响应。

动态创建的子进程（或子线程）通常只用来为一个客户服务（除非我们做特殊的处理），这将导致系统上产生大量的细微进程（或线程）。进程（或线程）间的切换将消耗大量CPU时间。

## 进程池

&emsp;&emsp;进程池和线程池相似，线程池是由服务器预先创建的一组子进程这些子进程的数目在3-10个之间。进程池中的所有子进程都运行着相同的代码，并具有相同的属性。当有新任务到来时，主进程将通过某种方式选择进程池中的某一个子进程来为之服务。相比于动态创建子进程，选择一个已经存在的子进程的代价显然要小得多。主进程选择哪个子进程来为新任务服务，则有两种方式：

* 主进程使用某种算法来主动选择子进程。
* 主进程和所有子进程通过一个共享的工作队列来同步，子进程都睡眠在该工作队列上，当有新的任务来，主线程将任务添加到工作队列中。

当选择好子进程后，主进程还需要使用某种通知机制来告诉目标子进程有新任务需要处理，并传递必要的数据。最简单的方法是：在父进程和子进程之间预先建立好一条管道，然后通过该管道来实现所有的进程间通信。
![image](https://user-images.githubusercontent.com/81791654/169477369-28fc5b71-5fce-4f9a-b4d2-c0674301b276.png)

### 处理多客户

&emsp;&emsp;在使用进程池处理多客户任务时，监听socket和连接sockets是由主进程统一管理这两种socket。
![image](https://user-images.githubusercontent.com/81791654/169478956-06e419a3-3535-41b7-acf3-33aaa5b04e99.png)

### 半同步/半反应堆线程池

&emsp;&emsp;使用一个工作队列完全解除了主线程和工作线程的耦合关系；主线程往工作队列中插入任务，工作线程通过竞争来取得任务并执行它，如果要将该线程池应用到实际服务器程序中，那么我们必须所有客户请求都是无状态的因为同一个连接上的不同请求可能会由不同的线程处理。

服务器编程基本框架：主要由I/O单元，逻辑单元和网络存储单元组成，其中每个单元之间通过请求队列进行通信，从而协同完成任务。其中I/O单元用于处理客户端连接，读写网络数据；逻辑单元用于处理业务逻辑的线程；网络存储单元指本地数据库和文件等。

## 五种I/O模型

阻塞I/O，非阻塞I/O，信号驱动I/O和I/O复用都是同步I/O。同步I/O指内核向应用程序通知的是就绪事件，比如只通知有客户端连接，要求用户代码自行执行I/O操作，异步I/O是指内核向应用程序通知的是完成事件，比如读取客户端的数据后才通知应用程序，由内核完成I/O操作。

reactor模式中，主线程(I/O处理单元)只负责监听文件描述符上是否有事件发生，有的话立即通知工作线程(逻辑单元 )，读写数据、接受新连接及处理客户请求均在工作线程中完成。通常由同步I/O实现。

proactor模式中，主线程和内核负责处理读写数据、接受新连接等I/O操作，工作线程仅负责业务逻辑，如处理客户请求。通常由异步I/O实现

同步I/O模型的工作流程如下（epoll_wait为例）：

主线程往epoll内核事件表注册socket上的读就绪事件。

主线程调用epoll_wait等待socket上有数据可读

当socket上有数据可读，epoll_wait通知主线程,主线程从socket循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。

睡眠在请求队列上某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往epoll内核事件表中注册该socket上的写就绪事件

主线程调用epoll_wait等待socket可写。

当socket上有数据可写，epoll_wait通知主线程。主线程往socket上写入服务器处理客户请求的结果。

## 同步I/O模拟proactor模式

由于异步I/O并不成熟，实际中使用较少，这里将使用同步I/O模拟实现proactor模式。

同步I/O模型的工作流程如下（epoll_wait为例）：

* 主线程往epoll内核事件表注册socket上的读就绪事件。
* 主线程调用epoll_wait等待socket上有数据可读
* 当socket上有数据可读，epoll_wait通知主线程,主线程从socket循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。
* 睡眠在请求队列上某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往epoll内核事件表中注册该socket上的写就绪事件
* 主线程调用epoll_wait等待socket可写。
* 当socket上有数据可写，epoll_wait通知主线程。主线程往socket上写入服务器处理客户请求的结果。

## 半同步/半反应堆

半同步/半反应堆并发模式是半同步/半异步的变体，将半异步具体化为某种事件处理模式.

**半同步/半异步模式工作流程**

同步线程用于处理客户逻辑

异步线程用于处理I/O事件

异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中

请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象

**半同步/半反应堆工作流程（以Proactor模式为例）**

主线程充当异步线程，负责监听所有socket上的事件

若有新请求到来，主线程接收之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件

如果连接socket上有读写事件发生，主线程从socket上接收数据，并将数据封装成请求对象插入到请求队列中

所有工作线程睡眠在请求队列上，当有任务到来时，通过竞争（如互斥锁）获得任务的接管权

## 线程池分析

线程池的设计模式为半同步/半反应堆，其中反应堆具体为Proactor事件处理模式。

具体的，主线程为异步线程，负责监听文件描述符，接收socket新连接，若当前监听的socket发生了读写事件，然后将任务插入到请求队列。工作线程从请求队列中取出任务，完成读写数据的处理。

**相关知识补充：**

1、线程创建和结束
背景知识：
&emsp;&emsp;在一个文件内的多个函数通常通常都是按照main函数中出现的顺序来执行，但是在分时系统下，我们可以让每个函数都作为一个逻辑流并发执行，最简单的方式就是采用多线程策略。在main函数中调用多线程接口创建接口创建线程，每个线程对应特定的函数（操作），这样就可以不按照main函数中各个函数出现的顺序来执行，避免了忙等的情况。线程基本操作的接口如下：

**相关接口：**

**创建线程 - pthread_create**
```cpp
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
```
* 函数参数：
`pthread_t`：传出参数，保存系统为我们分配好的线程`ID`，当前`Linux`中可理解为：`typedef unsigned long int pthread_t`。
`attr`：通常传`NULL`，表示使用线程默认属性。若想使用具体属性也可以修改该参数。
`start_routine`：函数指针，指向线程主函数(线程体)，该函数运行结束，则线程结束。
`arg`：线程主函数执行期间所使用的参数。

&emsp;&emsp;函数原型中的第三个参数，为函数指针，指向处理线程函数的地址。该函数，要求为静态函数。如果处理线程函数为类成员函数时，需要将其设置为静态成员函数。

&emsp;&emsp;pthread_create的函数原型中第三个参数的类型为函数指针，指向的线程处理函数参数类型为`(void *)`,若线程函数为类成员函数，则this指针会作为默认的参数被传进函数中，从而和线程函数参数`(void*)`不能匹配，不能通过编译。

静态成员函数就没有这个问题，里面没有this指针。

**获取线程ID - pthread_self**
```cpp
#include <pthread.h>
pthread_t pthread_self(void);//调用时，会打印线程ID
```

**等待线程结束 - pthread_join**
阻塞等待线程退出，获取线程退出状态。其作用，对应进程中的waitpid() 函数。
```cpp
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
```

**结束线程 - pthread_exit**

```cpp
void pthread_exit(void *retval);
```
&emsp;&emsp;在线程中禁止调用 **exit** 函数，否则会导致整个进程退出，取而代之的是调用 **pthread_exit** 函数，这个函数是使一个线程退出，如果主线程调用 **pthread_exit** 函数也不会使整个进程退出，不影响其他线程的执行。

**NOTE：pthread_exit或者return返回的指针所指向的内存单元必须是全局的或者是用malloc分配的，不能在线程函数的栈上分配，因为当其它线程得到这个返回指针时线程函数已经退出了，栈空间就会被回收。**

## 线程池分析

&emsp;&emsp;线程池的设计模式为半同步/半反应堆，其中反应堆具体为Proactor事件处理模式。

&emsp;&emsp;具体的，主线程为异步线程，负责监听文件描述符，接收socket新连接，若当前监听的socket发生了读写事件，然后将任务插入到请求队列。工作线程从请求队列中取出任务，完成读写数据的处理。

### 线程池类定义

```cpp
//线程池类，将它定义为模板类是为了代码复用，模板参数T是任务类
template <template T>
class threadpool {
public:
	/*  thread_number是线程池中线程的数量，
        max_requests是请求是请求队列中最多允许的、等待处理的请求的数量
        connPool是数据库连接池指针
     */
	threadpool(int actor_model, connection_pool* connPool, int thread_number = 8, int max_request = 10000);
	~threadpool();
    //向请求队列中插入任务请求
	bool append(T* request, int state);
	bool append_p(T* request);

private:
	/*  工作线程允许的函数，
        它不断从工作队列中取出任务并执行之
     */
	static void* worker(void* arg);
	void run();

private:
	int m_thread_number;            //线程池中的线程数
	int m_max_requests;             //请求队列中允许的最大请求数
	pthread_t* m_threads;           //描述线程池的数组，其大小为m_thread_number
	std::list<T*> m_workqueue;      //请求队列
	locker m_queuelocker;           //保护请求队列的互斥锁
	sem m_queuestat;                //是否有任务需要处理
	connection_pool* m_connPool;    //数据库连接池
	int m_actor_model;              //模型切换
};
```

### 线程池创建与回收

&emsp;&emsp;构造函数中创建线程池,`pthread_create`函数中将类的对象作为参数传递给静态函数(`worker`),在静态函数中引用这个对象,并调用其动态方法(`run`)。

具体的，类对象传递时用`this`指针，传递给静态函数后，将其转换为线程池类，并调用私有成员函数`run`。

```cpp
/*
    构造函数中创建线程池,pthread_create函数中将类的对象作为参数传递给静态函数(worker),
    在静态函数中引用这个对象,并调用其动态方法(run)。
    具体的，类对象传递时用this指针，传递给静态函数后，将其转换为线程池类，并调用私有成员函数run。
 */
template <template T>
threadpool<T>::threadpool(int actor_model, connection_pool* connPool, int thread_number, int max_requests) : m_actor_model(actor_model), m_thread_number(thread_number), m_max_requests(max_requests), m_threads(NULL), m_connPool(connPool) {
	if (thread_number <= 0 || max_requests <= 0)
		throw std::exception();
    //线程id初始化
	m_threads = new pthread_t[m_thread_number];
	if (!m_threads)
		throw std::exception();
	for (int i = 0; i < thread_number; ++i) {
        //循环创建线程，并将工作线程按要求进行运行
		if (pthread_create(m_threads + i, NULL, worker, this) != 0) {//线程id，指向线程属性的指针通常设置为NULL，处理线程函数的地址，前者的参数
			delete[] m_threads;
			throw std::exception();
		}
        //将线程进行分离后，不用单独对工作线程进行回收
		if (pthread_detach(m_threads[i])) {
			delete[] m_threads;
			throw std::exception();
		}
	}
}

template <typename T>
threadpool<T>::~threadpool() {
    delete[] m_threads;
}
```

### 向请求队列中添加任务

&emsp;&emsp;通过list容器创建请求队列，向队列中添加时，通过互斥锁保证线程安全，添加完成后通过信号量提醒有任务要处理，最后注意线程同步。

```cpp
/*
 * 向请求队列中添加任务
 * 通过list容器创建请求队列，向队列中添加时，通过互斥锁保证线程安全，
 * 添加完成后通过信号量提醒有任务要处理，最后注意线程同步。
 */
template <typename T>
//通过list容器创建请求队列，向队列中添加时，通过互斥锁保证线程安全，
//添加完成后通过信号量提醒有任务要处理，最后注意线程同步。
bool threadpool<T>::append(T* request, int state) {
    m_queuelocker.lock();
    //根据硬件，预先设置请求队列的最大值
    if (m_workqueue.size() >= m_max_requests) {
        m_queuelocker.unlock();
        return false;
    }
    request->m_state = state;
    //添加任务
    m_workqueue.push_back(request);
    m_queuelocker.unlock();
    //信号量提醒有任务要处理
    m_queuestat.post();
    return true;
}
template <typename T>
bool threadpool<T>::append_p(T* request) {
    m_queuelocker.lock();
    //根据硬件，预先设置请求队列的最大值
    if (m_workqueue.size() >= m_max_requests) {
        m_queuelocker.unlock();
        return false;
    }
    //添加任务
    m_workqueue.push_back(request);
    m_queuelocker.unlock();
    //信号量提醒有任务要处理
    m_queuestat.post();
    return true;
}
```

### 线程处理函数

内部访问私有成员函数`run`，完成线程处理要求。

```cpp
/* 线程处理函数：内部访问私有成员函数run，完成线程处理要求。 */
template <typename T>
//内部访问私有成员函数run，完成线程处理要求。
void* threadpool<T>::worker(void* arg) {
    //将参数强转为线程池类，调用成员方法
    threadpool* pool = (threadpool*)arg;
    pool->run();
    return pool;
}
```

### run执行任务

&emsp;&emsp;主要实现，工作线程从请求队列中取出某个任务进行处理，注意线程同步。

```cpp
/* 线程处理函数：内部访问私有成员函数`run`，完成线程处理要求。 */
template <typename T>
void threadpool<T>::run() {
    while (true) {
        //信号量等待
        m_queuestat.wait();
        //被唤醒后先加互斥锁
        m_queuelocker.lock();
        if (m_workqueue.empty()) {
            m_queuelocker.unlock();
            continue;
        }
        /*
            从请求队列中取出第一个任务
            将任务从请求队列删除
        */
        T* request = m_workqueue.front();
        m_workqueue.pop_front();
        m_queuelocker.unlock();
        if (!request)
            continue;
        if (1 == m_actor_model) {
            if (0 == request->m_state) {
                if (request->read_once()) {
                    request->improv = 1;
                    connectionRAII mysqlcon(&request->mysql, m_connPool);
                    //process(模板类中的方法,这里是http类)进行处理
                    request->process();
                }
                else {
                    request->improv = 1;
                    request->timer_flag = 1;
                }
            }
            else {
                if (request->write()) {
                    request->improv = 1;
                }
                else {
                    request->improv = 1;
                    request->timer_flag = 1;
                }
            }
        }
        else {
            connectionRAII mysqlcon(&request->mysql, m_connPool);
            request->process();
        }
    }
}
```











