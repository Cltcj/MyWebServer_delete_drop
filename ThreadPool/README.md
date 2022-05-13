# ThreadPool

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
