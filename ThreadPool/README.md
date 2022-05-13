# ThreadPool

线程池主要包括以下几个功能组成：

1、工作线程：一般初始化线程池中的维护的线程个数，当没有任务的时候，线程处于阻塞状态。直到添加新的任务到线程池中，则随机唤醒其中一个线程，来处理任务。当有多个任务到来时，则随机唤醒多个线程来处理任务。

2、任务接口：添加任务的接口,以供工作线程调度任务的执行。

3、任务队列：添加的任务先放到任务队列中去，当任务队列不为空时，则唤醒线程执行任务。




ThreadPool类中有：

5个成员变量
std::vector< std::thread > workers 用于存放线程的数组，用vector容器保存

std::queue< std::function<void()> > tasks 用于存放任务的队列，用queue队列进行保存。任务类型为std::function<void()>。因为 std::function是通用多态函数封装器，也就是说本质上任务队列中存放的是一个个函数

std::mutex queue_mutex 一个访问任务队列的互斥锁，在插入任务或者线程取出任务都需要借助互斥锁进行安全访问

std::condition_variable condition 一个用于通知线程任务队列状态的条件变量，若有任务则通知线程可以执行，否则进入wait状态

bool stop 标识线程池的状态，用于构造与析构中对线程池状态的了解

3个成员函数
ThreadPool(size_t) 线程池的构造函数

auto enqueue(F&& f, Args&&... args) 将任务添加到线程池的任务队列中

~ThreadPool() 线程池的析构函数
