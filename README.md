# MyWebServer


![image](https://user-images.githubusercontent.com/81791654/169032750-94884aec-1c5d-4a60-a519-8ed275dab51a.png)




![image](https://user-images.githubusercontent.com/81791654/171119662-180a1548-6e97-47a7-8887-d907fb7bae1a.png)


整体的流程概述：

此项目包括：

[1、三种线程同步机制的封装](https://github.com/Cltcj/MyWebServer/tree/main/lock)

[2、线程池](https://github.com/Cltcj/MyWebServer/tree/main/threadpool)

[3、http连接处理](https://github.com/Cltcj/MyWebServer/tree/main/http)

[4、定时器处理非活动连接](https://github.com/Cltcj/MyWebServer/tree/main/timer)

[5、数据库连接池](https://github.com/Cltcj/MyWebServer/tree/main/CGImysql)

[6、日志系统](https://github.com/Cltcj/MyWebServer/tree/main/log)

接下来，我们来聊聊各模块作用及执行流程。

第一部分是用于多线程的同步和互斥，用来保证任一时刻只有一个线程能够进入关键代码段。

第二部分则是一个半同步/半反应堆线程池，这是通过同步I/O模拟proactor模式，使用一个工作队列处理主线程和工作线程的耦合关系：主线程往工作线程中插入任务，工作线程通过竞争来取得任务并执行它。

第三个部分是关于http连接处理，这一部分是核心部分，主要为通过线程池以及有限状态机方式来处理HTTP请求报文的解析等任务


第四部分使用定时器来处理非活动的连接，什么意思呢？

比如一些用户connect()到服务器后，长时间没有动作，然后也不会断开，这样就会导致服务端的文件描述符被一直占用，导致连接资源的浪费。而定时器把这些超时的非活动连接释放掉，关闭其占用的文件描述符。

项目中使用的是SIGALRM信号来实现定时器，利用alarm函数周期性的触发SIGALRM信号，信号处理函数利用管道通知主循环，主循环接收到该信号后对升序链表上所有定时器进行处理，若该段时间内没有交换数据，则将该连接关闭，释放所占用的资源。

项目中将连接资源、定时事件和超时时间封装为定时器类，具体为：

连接资源包括客户端套接字地址、文件描述符和定时器

定时事件为回调函数，将其封装起来由用户自定义，这里是删除非活动socket上的注册事件，并关闭

定时器超时时间 = 浏览器和服务器连接时刻 + 固定时间(TIMESLOT)，可以看出，定时器使用绝对时间作为超时值，这里alarm设置为5秒，连接超时为15秒

第五部分是一个数据库连接池，用来预先生成一些数据库连接放在那里供用户请求使用。

第六部分是日志，由服务器自动创建，并记录运行状态，错误信息，访问数据的文件。

 
