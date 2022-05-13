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

class ThreadPool {
  public:
      ThreadPool(size_t);
      template<class F, class... Args>
      auto enqueue(F&& f, Args&&... args) 
          -> std::future<typename std::result_of<F(Args...)>::type>;
      ~ThreadPool();
  private:
      // need to keep track of threads so we can join them
      std::vector< std::thread > workers;
      // the task queue
      std::queue< std::function<void()> > tasks;
      
      // synchronization
      std::mutex queue_mutex;
      std::condition_variable condition;
      bool stop;
  };
  
  
  构造函数解析
  
  inline ThreadPool::ThreadPool(size_t threads)
      :   stop(false)
  {
      for(size_t i = 0;i<threads;++i)
          workers.emplace_back(
              [this]
              {
                  for(;;)
                  {
                      std::function<void()> task;
  ​
                      {
                          std::unique_lock<std::mutex> lock(this->queue_mutex);
                          this->condition.wait(lock,
                              [this]{ return this->stop || !this->tasks.empty(); });
                          if(this->stop && this->tasks.empty())
                              return;
                          task = std::move(this->tasks.front());
                          this->tasks.pop();
                      }
  ​
                      task();
                  }
              }
          );
  }
  
构造函数定义为inline。

接收参数threads表示线程池中要创建多少个线程。

初始化成员变量stop为false，即表示线程池启动着。

然后进入for循环，依次创建threads个线程，并放入线程数组workers中。

在vector中，emplace_back()成员函数的作用是在容器尾部插入一个对象，作用效果与push_back()一样，但是两者有略微差异，即emplace_back(args)中放入的对象的参数，而push_back(OBJ(args))中放入的是对象。即emplace_back()直接在容器中以传入的参数直接调用对象的构造函数构造新的对象，而push_back()中先调用对象的构造函数构造一个临时对象，再将临时对象拷贝到容器内存中。

我们知道，在C++11中，创建线程的方式为：

std::thread t(fun);    //fun为线程的执行函数

所以，上述workers.emplace_back()中，我们传入的lambda表达式就是创建线程的fun()函数。

下面来分析下该lambda表达式：

[this]{
      for(;;)
      {
          std::function<void()> task;
  ​
          {
              std::unique_lock<std::mutex> lock(this->queue_mutex);
              this->condition.wait(lock,
                                   [this]{ return this->stop || !this->tasks.empty(); });
              if(this->stop && this->tasks.empty())
                  return;
              task = std::move(this->tasks.front());
              this->tasks.pop();
          }
  ​
          task();
      }
  }
  
lambda表达式的格式为：

[ 捕获 ] ( 形参 ) 说明符(可选) 异常说明 attr -> 返回类型 { 函数体 }

所以上述lambda表达式为 [ 捕获 ] { 函数体 } 类型。

该lambda表达式捕获线程池指针this用于在函数体中使用（调用线程池成员变量stop、tasks等）

分析函数体，for(;;)为一个死循环，表示每个线程都会反复这样执行，这其实每个线程池中的线程都会这样。

在循环中，，先创建一个封装void()函数的std::function对象task，用于接收后续从任务队列中弹出的真实任务。

在C++11中,
std::unique_lock<std::mutex> lock(this->queue_mutex);

可以在退出作用区域时自动解锁，无需显式解锁。所以，{}起的作用就是在退出 } 时自动回释放线程池的queue_mutex。

在{}中，我们先对任务队列加锁，然后根据条件变量判断条件是否满足。

void
wait(unique_lock<mutex>& lock, _Predicate p)
{
    while (!p())
        wait(lock);
}
  
为条件标量wait的运行机制， wait在p 为false的状态下，才会进入wait(lock)状态。当前线程阻塞直至条件变量被通知。

this->condition.wait(lock,[this]{ return this->stop || !this->tasks.empty(); });
  
所以p表示上述代码中的lambda表达式[this]{ return this->stop || !this->tasks.empty(); }，其中this->stop为false， !this->tasks.empty()也为false。即其表示若线程池已停止或者任务队列中不为空，则不会进入到wait状态。

由于刚开始创建线程池，线程池表示未停止，且任务队列为空，所以每个线程都会进入到wait状态。
  
![image](https://user-images.githubusercontent.com/81791654/168278712-a53fd619-e6b2-4b81-a458-1c6d620a54f8.png)
  
在线程池刚刚创建，所有的线程都阻塞在了此处，即wait处。

若后续条件变量来了通知，线程就会继续往下进行：

  if(this->stop && this->tasks.empty())
      return;
 

若线程池已经停止且任务队列为空，则线程返回，没必要进行死循环。

  task = std::move(this->tasks.front());
  this->tasks.pop();
 

这样，将任务队列中的第一个任务用task标记，然后将任务队列中该任务弹出。（此处线程实在获得了任务队列中的互斥锁的情况下进行的，从上图可以看出，在条件标量唤醒线程后，线程在wait周期内得到了任务队列的互斥锁才会继续往下执行。所以最终只会有一个线程拿到任务，不会发生惊群效应）

在退出了{ }，我们队任务队列的所加的锁也释放了，然后我们的线程就可以执行我们拿到的任务task了，执行完毕之后，线程又进入了死循环。

至此，我们分析了ThreadPool的构造函数。

添加任务函数解析
复制代码
  template<class F, class... Args>
  auto ThreadPool::enqueue(F&& f, Args&&... args) 
      -> std::future<typename std::result_of<F(Args...)>::type>
  {
      using return_type = typename std::result_of<F(Args...)>::type;
  ​
      auto task = std::make_shared< std::packaged_task<return_type()> >(
              std::bind(std::forward<F>(f), std::forward<Args>(args)...)
          );
          
      std::future<return_type> res = task->get_future();
      {
          std::unique_lock<std::mutex> lock(queue_mutex);
  ​
          // don't allow enqueueing after stopping the pool
          if(stop)
              throw std::runtime_error("enqueue on stopped ThreadPool");
  ​
          tasks.emplace([task](){ (*task)(); });
      }
      condition.notify_one();
      return res;
  }
复制代码
 

添加任务的函数本来不难理解，但是作者增加了许多新的C++11特性，这样就变得难以理解了。

  template<class F, class... Args>
  auto ThreadPool::enqueue(F&& f, Args&&... args) 
      -> std::future<typename std::result_of<F(Args...)>::type>
 

equeue是一个模板函数，其类型形参为F与Args。其中class... Args表示多个类型形参。

auto用于自动推导出equeue的返回类型，函数的形参为(F&& f, Args&&... args)，其中&&表示右值引用。表示接受一个F类型的f，与若干个Args类型的args。

-> std::future<typename std::result_of<F(Args...)>::type>
表示返回类型，与lambda表达式中的表示方法一样。

返回的是什么类型呢？

  typename std::result_of<F(Args...)>::type   //获得以Args为参数的F的函数类型的返回类型
  std::future<typename std::result_of<F(Args...)>::type>
  //std::future用来访问异步操作的结果
 

所以，最终返回的是放在std::future中的F(Args…)返回类型的异步执行结果。

举个简单的例子来理解吧：

  
复制代码
  // 来自 packaged_task 的 future
  std::packaged_task<int()> task([](){ return 7; }); // 包装函数，将lambda表达式进行包装
  std::future<int> f1 = task.get_future();  // 定义一个future对象f1，存放int型的值。此处已经表明：将task挂载到线程上执行，然后返回的结果才会保存到f1中
  std::thread(std::move(task)).detach(); // 将task函数挂载在线程上运行
  ​
  f1.wait();  //f1等待异步结果的输入
  f1.get();   //f1获取到的异步结果
  
  struct S {
      double operator()(char, int&);
      float operator()(int) { return 1.0;}
  };
  ​
  std::result_of<S(char, int&)>::type d = 3.14; // d 拥有 double 类型，等价于double d = 3.14
  std::result_of<S(int)>::type x = 3.14; // x 拥有 float 类型，等价于float x = 3.14
复制代码
 

经过上述两个简单的小例子可以知道：

复制代码
  -> std::future<typename std::result_of<F(Args...)>::type>
  //等价于
  //F(Args...) 为  int f(args)
  //std::result_of<F(Args...)>::type  表示为 int
  //std::future<int> f1
  //return f1
  //在后续我们根据f1.get就可以取出存放在里面的int值
  //最终返回了一个F(Args...)类型的值，而这个值是存储在std::future中，因为线程是异步处理的
复制代码
接着分析：

  
 using return_type = typename std::result_of<F(Args...)>::type;
表示使用return_type表示F(Args...)的返回类型。

  
  auto task = std::make_shared< std::packaged_task<return_type()> >(
              std::bind(std::forward<F>(f), std::forward<Args>(args)...)
          );
 

由上述小例子，我们已经知道std::packaged_task是一个包装函数，所以

复制代码
  auto sp = std::make_shared<C>(12);   --->   auto sp = new C(12)  //创建一个智能指针sp，其指向一个用12初始化的C类对象
      
  std::packaged_task<return_type()>   //表示包装一个返回值为return_type的函数
      
  auto task = std::make_shared< std::packaged_task<return_type()> > (std::bind(std::forward<F>(f), std::forward<Args>(args)...)   //创建一个智能指针task，其指向一个用std::bind(std::forward<F>(f), std::forward<Args>(args)... 来初始化的 std::packaged_task<return_type()> 对象
  ​
  //即  std::packaged_task<return_type()> t1(std::bind(std::forward<F>(f), std::forward<Args>(args)...)
  //然后task指向了t1，即task指向了返回值为return_type的f(args)
      
  std::packaged_task<int()> task(std::bind(f, 2, 11));    //将函数f(2,11)打包成task，其返回值为int
复制代码
 

 

所以最终，task指向了传递进来的函数。

   std::future<return_type> res = task->get_future();
  //res中保存了类型为return_type的变量，有task异步执行完毕才可以将值保存进去
 

所以，res会在异步执行完毕后即可获得所求。

复制代码
  {
      std::unique_lock<std::mutex> lock(queue_mutex);
  ​
      // don't allow enqueueing after stopping the pool
      if(stop)
          throw std::runtime_error("enqueue on stopped ThreadPool");
  ​
      tasks.emplace([task](){ (*task)(); });  //(*task)() ---> f(args)
  }
复制代码
 

在新的作用于内加锁，若线程池已经停止，则抛出异常。

否则，将task所指向的f(args)插入到tasks任务队列中。需要指出，这儿的emplace中传递的是构造函数的参数。

  condition.notify_one(); //任务加入任务队列后，需要去唤醒一个线程
  return res; //待线程执行完毕，将异步执行的结果返回
经过上述分析，这样将每个人物插入到任务队列中的过程就完成了。

 

析构函数解析
复制代码
  inline ThreadPool::~ThreadPool()
  {
      {
          std::unique_lock<std::mutex> lock(queue_mutex);
          stop = true;
      }
      condition.notify_all();
      for(std::thread &worker: workers)
          worker.join();
  }
复制代码
 

在析构函数中，先对任务队列中加锁，将停止标记设置为true，这样后续即使有新的插入任务操作也会执行失败。

使用条件变量唤醒所有线程，所有线程都会往下执行:

  if(this->stop && this->tasks.empty())
      return;
 

在stop设置为true且任务队列中为空时，对应的线程进而跳出循环结束。

  for(std::thread &worker: workers)
     worker.join();
 

将每个线程设置为join，等到每个线程结束完毕后，主线程再退出。

 

主函数解析
复制代码
  ThreadPool pool(4); //创建一个线程池，池中线程为4
  std::vector< std::future<int> > results;    //创建一个保存std::future<int>的数组，用于存储4个异步线程的结果
  ​
  for(int i = 0; i < 8; ++i) {    //创建8个任务
      results.emplace_back(   //一次保存每个异步结果
          pool.enqueue([i] {  //将每个任务插入到任务队列中，每个任务的功能均为“打印+睡眠1s+打印+返回结果”
              std::cout << "hello " << i << std::endl;
              std::this_thread::sleep_for(std::chrono::seconds(1));
              std::cout << "world " << i << std::endl;
              return i*i;
          })
      );
  }
  ​
  for(auto && result: results)    //一次取出保存在results中的异步结果
      std::cout << result.get() << ' ';
  std::cout << std::endl;
复制代码
 

需要对主函数中的任务函数进行说明：

  [i] {   //将每个任务插入到任务队列中，每个任务的功能均为“打印+睡眠1s+打印+返回结果”
      std::cout << "hello " << i << std::endl;
      std::this_thread::sleep_for(std::chrono::seconds(1));
      std::cout << "world " << i << std::endl;
      return i*i;
  }
 

这个lambda表达式用来表示一个匿名函数，该函数分写执行 打印-睡眠-打印-返回结果。

pool.enqueue(fun);
 

对应于类中的

  auto ThreadPool::enqueue(F&& f, Args&&... args) 
      -> std::future<typename std::result_of<F(Args...)>::type>
其中，F&& f 是lambda表达式（或者说fun）的形参，而参数为0。

而

std::future<typename std::result_of<F(Args...)>::type>
则用来保存 i*i 。

对应的

std::result_of<F(Args...)>::type    //int型
上述是简要的分析。



