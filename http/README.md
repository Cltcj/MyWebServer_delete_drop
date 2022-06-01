## HTTP—Hyper Text Transfer Protocol（超文本传输协议）


HTTP（HyperText Transfer Protocol，超文本传输协议）是一种用于分布式、协作式和超媒体信息系统的应用层协议。HTTP 是万维网的数据通信的基础。

**HTTP报文格式**

HTTP报文分为请求报文和响应报文两种，每种报文必须按照特有格式生成，才能被浏览器端识别。

其中，浏览器端向服务器发送的为请求报文，服务器处理后返回给浏览器端的为响应报文。

**请求报文**

HTTP请求报文由请求行（request line）、请求头部（header）、空行和请求数据四个部分组成。

![image](https://user-images.githubusercontent.com/81791654/170449792-b55ec234-0f7d-4abd-80d2-9f3a807a27e7.png)

**请求行**，用来说明请求类型,要访问的资源以及所使用的HTTP版本。

**请求头部**，紧接着请求行（即第一行）之后的部分，用来说明服务器要使用的附加信息。

* HOST，给出请求资源所在服务器的域名。

* User-Agent，HTTP客户端程序的信息，该信息由你发出请求使用的浏览器来定义,并且在每个请求中自动发送等。

* Accept，说明用户代理可处理的媒体类型。

* Accept-Encoding，说明用户代理支持的内容编码。

* Accept-Language，说明用户代理能够处理的自然语言集。

* Content-Type，说明实现主体的媒体类型。

* Content-Length，说明实现主体的大小。

* Connection，连接管理，可以是Keep-Alive或close。

**空行**，请求头部后面的空行是必须的即使第四部分的请求数据为空，也必须有空行。

**请求数据**也叫主体，可以添加任意的其他数据。

**HTTP_CODE含义**

表示HTTP请求的处理结果，在头文件中初始化了八种情形

* NO_REQUEST：请求不完整，需要继续读取请求报文数据

* GET_REQUEST：获得了完整的HTTP请求

* NO_RESOURCE：请求资源不存在

* BAD_REQUEST：HTTP请求报文有语法错误或请求资源为目录

* FORBIDDEN_REQUEST：请求资源禁止访问，没有读取权限

* FILE_REQUEST：请求资源可以正常访问

* INTERNAL_ERROR：服务器内部错误

响应报文

HTTP响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

* 状态行，由HTTP协议版本号， 状态码， 状态消息 三部分组成。第一行为状态行，（HTTP/1.1）表明HTTP版本为1.1版本，状态码为200，状态消息为OK。

* 消息报头，用来说明客户端要使用的一些附加信息。第二行和第三行为消息报头，Date:生成响应的日期和时间；Content-Type:指定了MIME类型的HTML(text/html),编码类型是UTF-8。

* 空行，消息报头后面的空行是必须的。

* 响应正文，服务器返回给客户端的文本信息。空行后面的html部分为响应正文。

&emsp;&emsp;很多网络协议，包括TCP协议和IP协议，都在其头部中提供头部长度字段。程序根据该字段的值就可以知道是否接收到一个完整的协议头部。但HTTP协议并未提供这样的头部长度字段，并且其头部长度变化也很大，可以只有十几字节，也可以有上百字节。根据协议规定，我们判断HTTP头部结束的依据是遇到一个空行，该空行仅包含一对回车换行符（<CR><LF>）。如果一次读操作没有读入HTTP请求的整个头部，即没有遇到空行，那么我们必须等待客户继续写数据并再次读入。因此，我们每完成一次读操作，就要分析新读入的数据中是否有空行。不过在寻找空行的过程中，我们可以同时完成对整个HTTP请求头部的分析（记住，空行前面还有请求行和头部域），以提高解析HTTP请求的效率。
	
------------------------------------------------------------------------------------------------

### 有限状态机

有限状态机，是一种抽象的理论模型，它能够把有限个变量描述的状态变化过程，以可构造可验证的方式呈现出来。比如，封闭的有向图。

有限状态机可以通过if-else,switch-case和函数指针来实现，从软件工程的角度看，主要是为了封装逻辑。

带有状态转移的有限状态机示例代码：

![image](https://user-images.githubusercontent.com/81791654/170452323-d60dc672-f583-48a1-9c41-895058efa966.png)


![image](https://user-images.githubusercontent.com/81791654/170452348-274a6d23-68f4-4abc-b848-0671dcdb8d7a.png)


![image](https://user-images.githubusercontent.com/81791654/170452565-faf214ab-3216-4e1f-98f0-c482eaa1012f.png)

------------------------------------------------------------------------------------------------
	
## 代码分析：

### http_conn.h

**http_conn.h类的定义**

```cpp
class http_conn {
public:
    //设置读取文件的名称m_real_file大小
    static const int FILENAME_LEN = 200;
    //设置读缓冲区m_read_buf大小
    static const int READ_BUFFER_SIZE = 2048;
    //设置写缓冲区m_write_buf大小
    static const int WRITE_BUFFER_SIZE = 1024;
    enum METHOD { //报文的请求方法，本项目只用到GET和POST
        GET = 0,
        POST,
        HEAD,
        PUT,
        DELETE,
        TRACE,
        OPTIONS,
        CONNECT,
        PATH
    };
    //主状态机使用checkstate变量来记录当前的状态
    enum CHECK_STATE { //主状态机的三种可能状态
        CHECK_STATE_REQUESTLINE = 0,//表示parse_line函数解析出的行是请求行
        CHECK_STATE_HEADER,//parse_line函数解析出的是头部字段
        CHECK_STATE_CONTENT//parse_line函数解析出的是内容字段
    };
    /* 服务器处理HTTP请求的结果：*/
    enum HTTP_CODE {
        NO_REQUEST,//请求不完整，需要继续读取客户数据
        GET_REQUEST,//获得了一个完整的客户请求
        BAD_REQUEST,//客户请求有语法错误
        NO_RESOURCE,
        FORBIDDEN_REQUEST,//客户对资源没有足够的访问权限
        FILE_REQUEST,
        INTERNAL_ERROR,//服务器内部错误
        CLOSED_CONNECTION//客户端已经关闭连接
    };
    enum LINE_STATUS { //从状态机的状态
        /*
         * 从状态机的三种可能状态，即行的读取状态，
         * 分别表示：读取到一个完整的行、行出错
         * 和行数据尚且不完整
         */
        LINE_OK = 0,
        LINE_BAD,
        LINE_OPEN
    };

public:
    http_conn() {}
    ~http_conn() {}

public:
    /* 初始化套接字地址，函数内部会调用私有方法init */
    void init(int sockfd, const sockaddr_in& addr);
    /* 关闭http连接 */
    void close_conn(bool real_close = true);
    /* 处理客户请求 */
    void process();
    /* 读取浏览器端发来的全部数据 */
    bool read_once();
    /* 响应报文写入函数 */
    bool write();
    sockaddr_in* get_address() {
        return &m_address;
    }
    //同步线程初始化数据库读取表
    void initmysql_result(connection_pool* connPool);

private:
    //初始化套接字地址
    void init();
    //process线程工作主函数（包括Process read和write）
    //解析请求报文主函数while根据状态调用不同解析函数
    HTTP_CODE process_read();
    //向m_write_buf写入响应报文数据
    bool process_write(HTTP_CODE ret);
    //主状态机解析报文中的请求行数据
    HTTP_CODE parse_request_line(char* text);//分析请求行
    //主状态机解析报文中的请求头数据
    HTTP_CODE parse_headers(char* text);//分析头部字段
    //主状态机解析报文中的请求内容
    HTTP_CODE parse_content(char* text);//分析HTTP请求的入口函数
    //生成响应报文
    HTTP_CODE do_request();
    //m_start_line是已经解析的字符
    //get_line用于将指针向后偏移，指向未处理的字符
    char* get_line() { return m_read_buf + m_start_line; };

    //从状态机读取一行，分析是请求报文的哪一部分
    LINE_STATUS parse_line();//从状态机，用于解析出一行内容
    /* 下面这一组函数被process_write调用以填充HTTP应答 */
    void unmap();
    /* 根据响应报文格式，
     * 生成对应8个部分，
     * 以下函数均由do_request调用
     */
    bool add_response(const char* format, ...);//把传过来的字符串写到write buf中
    bool add_content(const char* content);
    bool add_status_line(int status, const char* title);
    bool add_headers(int content_length);
    bool add_content_type();
    bool add_content_length(int content_length);
    bool add_linger();
    bool add_blank_line();

public:
    /* 所有socket上的事件都被注册到同一个epoll内核事件表中，所以将epoll文件描述符设置为静态的 */
    static int m_epollfd;
    /* 统计用户数量 */
    static int m_user_count;
    MYSQL* mysql;

private:
    /* 读HTTP连接的socket和对方的socket地址 */
    int m_sockfd;
    sockaddr_in m_address;
    /* 读缓冲区，存储读取的请求报文数据 */
    char m_read_buf[READ_BUFFER_SIZE];
    /* 标识读缓冲中已经读入的客户数据的最后一个字节的下一个位置*/
    int m_read_idx;
    /* 当前正在分析的字符在读缓冲区中的位置 */
    int m_checked_idx;
    /* 当前正在解析的行的起始位置 */
    int m_start_line;

    /* 写缓冲区，存储发出的响应报文数据 */
    char m_write_buf[WRITE_BUFFER_SIZE];
    /* 写缓冲区中待发送的字节数--长度 */
    int m_write_idx;

    /* 主状态机当前所处的状态 */
    CHECK_STATE m_check_state;
    /* 请求方法 */
    METHOD m_method;
    /* 客户请求的目标文件的完整路径，其内容等于doc_root + m_url，doc_root是网站根目录*/
    char m_real_file[FILENAME_LEN];
    /* 客户请求的目标文件的文件名 */
    char* m_url;
    /* HTTP协议版本号 ，这里支持*/
    char* m_version;
    /* 主机名 */
    char* m_host;
    /* HTTP请求的消息体的长度 */
    int m_content_length;
    /* HTTP请求是否要求保持连接 */
    bool m_linger;
    /* 客户请求的目标文件被mmap到内存中的起始位置 */
    char* m_file_address;
    /* 目标文件的状态。通过它我们可以判断文件是否存在、是否为目录、是否可读、并获取文件大小等信息 */
    struct stat m_file_stat;
    /* 我们将采用writev来执行写操作，所以定义下面两个成员，其中m_iv_count表示被写内存块的数量 */
    struct iovec m_iv[2];

    int m_iv_count;
    int cgi;        //是否启用的POST
    char* m_string; //存储请求头数据
    int bytes_to_send;//剩余发送字节数
    int bytes_have_send;//已发送字节数

};
```

------------------------------------------------------------------------------------------------
	
### http_conn.cpp	
	
&emsp;&emsp;这一部分涉及[多路I/O技术](https://github.com/Cltcj/MyWebServer/tree/main/%E5%A4%9A%E8%B7%AFIO%E6%8A%80%E6%9C%AF)
可以自行了解
	
**① 首先对epoll相关代码进行整理**
	
此项目中epoll相关代码部分包括非阻塞模式、内核事件表注册事件、删除事件、重置EPOLLONESHOT事件四种。
	
* 1、设置文件描述符状态

&emsp;&emsp;fcntl可将一个socket设置成非阻塞模式，在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F_SETFD或F_SETFL命令，这样会关闭以前设置的标志位。下面一段代码表示如何设置阻塞和非阻塞模式：

```cpp
flags = fcntl(sockfd, F_GETFL, 0); //获取文件的flags值。
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK); //设置成非阻塞模式；
flags  = fcntl(sockfd,F_GETFL,0);
fcntl(sockfd,F_SETFL,flags&~O_NONBLOCK); //设置成阻塞模式；	
```
	
```cpp
//对文件描述符设置非阻塞
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}	
```	

* 2、内核事件表注册新事件
	
&emsp;&emsp;epoll_event的作用是什么？我们知道epoll_wait函数能获取是否有注册事件发生，但这个事件到底是什么、从哪个 socket 来、发送的时间、包的大小等等信息，都不知道。这就好比一个人在黑黢黢的山洞里，只能听到声响，至于这个声音是谁发出的根本不知道。因此我们就需要struct epoll_event来帮助我们读取信息。

&emsp;&emsp;epoll_event 结构体的定义如上所示，分为 events 和 data 两个部分。其中events 是 epoll 注册的事件，比如EPOLLIN、EPOLLOUT等等，这个参数在epoll_ctl注册事件时，可以明确告知注册事件的类型。data 是一个联合体，它一般是用来传递参数。

	
```cpp
//将内核事件表注册读事件，ET模式，选择开启EPOLLONESHOT，针对客户端连接的描述符，listenfd不用开启
void addfd(int epollfd, int fd, bool one_shot)
{
    epoll_event event;
    event.data.fd = fd;

#ifdef connfdET
    event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
#endif

#ifdef connfdLT
    event.events = EPOLLIN | EPOLLRDHUP;
#endif

#ifdef listenfdET
    event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
#endif

#ifdef listenfdLT
    event.events = EPOLLIN | EPOLLRDHUP;
#endif

    if (one_shot)
        event.events |= EPOLLONESHOT;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}
```

* 3、内核事件表删除事件

```cpp
//从内核时间表删除描述符
void removefd(int epollfd, int fd) {
    epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, 0);
    close(fd);
}
```
	
* 4、重置EPOLLONESHOT事件

```cpp
//将事件重置为EPOLLONESHOT
void modfd(int epollfd, int fd, int ev)
{
    epoll_event event;
    event.data.fd = fd;

#ifdef connfdET
    event.events = ev | EPOLLET | EPOLLONESHOT | EPOLLRDHUP;
#endif

#ifdef connfdLT
    event.events = ev | EPOLLONESHOT | EPOLLRDHUP;
#endif

    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}
```	
	
	
② 第二是http请求接收部分

&emsp;&emsp;这部分会涉及到init和read_once函数，但init仅仅是对私有成员变量进行初始化:

**init函数**
```cpp
//初始化新接受的连接
//check_state默认为分析请求行状态
void http_conn::init()
{
    mysql = NULL;
    bytes_to_send = 0;
    bytes_have_send = 0;
    m_check_state = CHECK_STATE_REQUESTLINE;
    m_linger = false;
    m_method = GET;
    m_url = 0;
    m_version = 0;
    m_content_length = 0;
    m_host = 0;
    m_start_line = 0;
    m_checked_idx = 0;
    m_read_idx = 0;
    m_write_idx = 0;
    cgi = 0;
    memset(m_read_buf, '\0', READ_BUFFER_SIZE);
    memset(m_write_buf, '\0', WRITE_BUFFER_SIZE);
    memset(m_real_file, '\0', FILENAME_LEN);
}	
```
read_once读取浏览器端发送来的请求报文，直到无数据可读或对方关闭连接，读取到m_read_buffer中，并更新m_read_idx。
	
**read_once函数**	
	
```cpp	
//循环读取客户数据，直到无数据可读或对方关闭连接
//非阻塞ET工作模式下，需要一次性将数据读完
bool http_conn::read_once()
{
    if (m_read_idx >= READ_BUFFER_SIZE)
    {
        return false;
    }
    int bytes_read = 0;

#ifdef connfdLT

    bytes_read = recv(m_sockfd, m_read_buf + m_read_idx, READ_BUFFER_SIZE - m_read_idx, 0);
    m_read_idx += bytes_read;

    if (bytes_read <= 0)
    {
        return false;
    }

    return true;

#endif

#ifdef connfdET
    while (true)
    {
        bytes_read = recv(m_sockfd, m_read_buf + m_read_idx, READ_BUFFER_SIZE - m_read_idx, 0);
        if (bytes_read == -1)
        {//非阻塞ET模式下，需要一次性将数据读完
            if (errno == EAGAIN || errno == EWOULDBLOCK)
                break;
            return false;
        }
        else if (bytes_read == 0)
        {
            return false;
        }
        m_read_idx += bytes_read;
    }
    return true;
#endif
}	
```	

**③ 服务器接收客户端（浏览器）发来的http请求报文**
		       
上面，浏览器端发出http连接请求，主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列，工作线程从任务队列中取出一个任务进行处理。

既然浏览器发送了请求报文，那么Web服务器就需要接收客户端（浏览器）发来的请求报文，所以要考虑的一个问题就是Web服务器如何接收客户端发来的HTTP请求报文呢?

Web服务器端通过socket监听来自用户的请求：

```cpp		       
int listenfd = socket(PF_INET, SOCK_STREAM, 0);
assert(listenfd >= 0);

//struct linger tmp={1,0};
//SO_LINGER若有数据待发送，延迟关闭
//setsockopt(listenfd,SOL_SOCKET,SO_LINGER,&tmp,sizeof(tmp));

int ret = 0;
struct sockaddr_in address;
bzero(&address, sizeof(address));
address.sin_family = AF_INET;
address.sin_addr.s_addr = htonl(INADDR_ANY);
address.sin_port = htons(port);

int flag = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
assert(ret >= 0);
ret = listen(listenfd, 5);
assert(ret >= 0);
```
		       
&emsp;&emsp;远端的很多用户会尝试去connect()这个WebServer上正在listen的这个port，而监听到的这些连接会排队等待被accept()。由于用户连接请求是随机到达的异步事件，每当监听socket（listenfd）listen到新的客户连接并且放入监听队列，我们都需要告诉我们的Web服务器有连接来了，accept这个连接，并分配一个逻辑单元来处理这个用户请求。而且，我们在处理这个请求的同时，还需要继续监听其他客户的请求并分配其另一逻辑单元来处理（并发，同时处理多个事件，后面会提到使用线程池实现并发）。这里，服务器通过epoll这种I/O复用技术（还有select和poll）来实现对监听socket（listenfd）和连接socket（客户请求）的同时监听。注意I/O复用虽然可以同时监听多个文件描述符，但是它本身是阻塞的，并且当有多个文件描述符同时就绪的时候，如果不采取额外措施，程序则只能按顺序处理其中就绪的每一个文件描述符，所以为提高效率，我们将在这部分通过线程池来实现并发（多线程并发），为每个就绪的文件描述符分配一个逻辑单元（线程）来处理。


	
```cpp		       
//创建内核事件表
epoll_event events[MAX_EVENT_NUMBER];
epollfd = epoll_create(5);
assert(epollfd != -1);
		       
//将listenfd放在epoll树上
addfd(epollfd, listenfd, false);
		       
//将上述epollfd赋值给http类对象的m_epollfd属性
http_conn::m_epollfd = epollfd;

//创建MAX_FD个http类对象
client_data* users_timer = new client_data[MAX_FD];

while (!stop_server)
{
        //等待所监控文件描述符上有事件的产生
	int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
	if (number < 0 && errno != EINTR)
	{
	    break;
	}
	//对所有就绪事件进行处理
	for (int i = 0; i < number; i++)
	{
	    int sockfd = events[i].data.fd;

	    //处理新到的客户连接
	    if (sockfd == listenfd)
	    {
		struct sockaddr_in client_address;
		socklen_t client_addrlength = sizeof(client_address);
#ifdef listenfdLT
		int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
		if (connfd < 0)
		{
		    continue;
		}
		if (http_conn::m_user_count >= MAX_FD)
		{
		    show_error(connfd, "Internal server busy");
		    continue;
		}
		users[connfd].init(connfd, client_address);
#endif

#ifdef listenfdET
		//循环接收数据
		while (1)
		{
		    int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
		    if (connfd < 0)
		    {
			break;
		    }
		    if (http_conn::m_user_count >= MAX_FD)
		    {
			show_error(connfd, "Internal server busy");
			break;
		    }
		    users[connfd].init(connfd, client_address);
		}
		continue;
#endif
	    }
	    //处理异常事件
	    else if (events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR))
	    {
		//服务器端关闭连接
	    }

	    //处理信号
	    else if ((sockfd == pipefd[0]) && (events[i].events & EPOLLIN))
	    {
	    }

	    //处理客户连接上接收到的数据
	    else if (events[i].events & EPOLLIN)
	    {
		//读入对应缓冲区
		if (users[sockfd].read_once())
		{
		    //若监测到读事件，将该事件放入请求队列
		    pool->append(users + sockfd);
		}
		else
		{
			
		}
	    }    
	}
}
```		       
		       
### 流程图与状态机

&emsp;&emsp;从状态机负责读取报文的一行，主状态机负责对该行数据进行解析，主状态机内部调用从状态机，从状态机驱动主状态机。

![image](https://user-images.githubusercontent.com/81791654/170871894-d528b56a-a53a-4b64-afb3-b2bfead01695.png)

**主状态机**
		       
三种状态，标识解析位置。

* CHECK_STATE_REQUESTLINE，解析请求行

* CHECK_STATE_HEADER，解析请求头

* CHECK_STATE_CONTENT，解析消息体，仅用于解析POST请求	

**从状态机**

三种状态，标识解析一行的读取状态。

* LINE_OK，完整读取一行

* LINE_BAD，报文语法有误

* LINE_OPEN，读取的行不完整

**代码分析-http报文解析**

&emsp;&emsp;前面介绍了服务器接收http请求的流程与细节，简单来讲，浏览器端发出http连接请求，服务器端主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列后，工作线程从任务队列中取出一个任务进行处理。	       
		       
&emsp;&emsp;各子线程通过process函数对任务进行处理，调用process_read函数和process_write函数分别完成报文解析与报文响应两个任务。
		       
可以看出，这里将process()的实现做了两个划分，我们将response改为write(因为回应的本质是向回写)然后并入process()中，使整个逻辑更加完整。

void的修饰不是强制的，你可以写成bool，int，或者自己宏定义，用于判断处理成功与否。

read和write分工操作其实是为了以后的线程池和epoll，这里按下不表。接下来，讲一下如何实现两个不同的功能
		       
```cpp
void http_conn::process() {
    HTTP_CODE read_ret = process_read();

    //NO_REQUEST，表示请求不完整，需要继续接收请求数据
    if (read_ret == NO_REQUEST) {
        //注册并监听读事件
        modfd(m_epollfd, m_sockfd, EPOLLIN);
        return;
    }
    //调用process_write完成报文响应
    bool write_ret = process_write(read_ret);
    if (!write_ret) {
        close_conn();
    }
    //注册并监听写事件
    modfd(m_epollfd, m_sockfd, EPOLLOUT);
}
```	
**HTTP_CODE含义**		       

&emsp;&emsp;表示HTTP请求的处理结果，在头文件中初始化了八种情形，在报文解析时只涉及到四种。

**NO_REQUEST**

* 请求不完整，需要继续读取请求报文数据

**GET_REQUEST**

* 获得了完整的HTTP请求

**BAD_REQUEST**

* HTTP请求报文有语法错误

**INTERNAL_ERROR**

* 服务器内部错误，该结果在主状态机逻辑switch的default下，一般不会触发		       
	
### 解析报文整体流程
		       
process_read通过while循环，将主从状态机进行封装，对报文的每一行进行循环处理。

**判断条件**

* 主状态机转移到CHECK_STATE_CONTENT，该条件涉及解析消息体

* 从状态机转移到LINE_OK，该条件涉及解析请求行和请求头部

* 两者为或关系，当条件为真则继续循环，否则退出		       
		       
**循环体**

* 从状态机读取数据

* 调用get_line函数，通过m_start_line将从状态机读取数据间接赋给text

* 主状态机解析text		       

```cpp		       
get_line函数
//m_start_line是行在buffer中的起始位置，将该位置后面的数据赋给text
//此时从状态机已提前将一行的末尾字符\r\n变为\0\0，所以text可以直接取出完整的行进行解析
char* get_line() {
    return m_read_buf+m_start_line;
}
```		       
		       
```cpp			       
http_conn::HTTP_CODE http_conn::process_read() {
    //初始化从状态机状态、HTTP请求解析结果
    LINE_STATUS line_status = LINE_OK;
    HTTP_CODE ret = NO_REQUEST;
    char* text = 0;

    //这里为什么要写两个判断条件？第一个判断条件为什么这样写？
    //具体的在主状态机逻辑中会讲解。
    //parse_line为从状态机的具体实现
    while ((m_check_state == CHECK_STATE_CONTENT && line_status == LINE_OK) || ((line_status = parse_line()) == LINE_OK)) {
        text = get_line();
        //m_start_line是每一个数据行在m_read_buf中的起始位置
        //m_checked_idx表示从状态机在m_read_buf中读取的位置
        m_start_line = m_checked_idx;
        LOG_INFO("%s", text);
        Log::get_instance()->flush();

        //主状态机的三种状态转移逻辑
        switch (m_check_state)
        {
            case CHECK_STATE_REQUESTLINE:
            {
                //解析请求行
                ret = parse_request_line(text);
                if (ret == BAD_REQUEST)
                    return BAD_REQUEST;
                break;
            }
            case CHECK_STATE_HEADER:
            {
                //解析请求头
                ret = parse_headers(text);
                if (ret == BAD_REQUEST)
                    return BAD_REQUEST;

                //完整解析GET请求后，跳转到报文响应函数
                else if (ret == GET_REQUEST)
                {
                    return do_request();
                }
                break;
            }
            case CHECK_STATE_CONTENT:
            {
                //解析消息体
                ret = parse_content(text);

                //完整解析POST请求后，跳转到报文响应函数
                if (ret == GET_REQUEST)
                    return do_request();

                //解析完消息体即完成报文解析，避免再次进入循环，更新line_status
                line_status = LINE_OPEN;
                break;
            }
            default:
                return INTERNAL_ERROR;
        }
    }
    return NO_REQUEST;
}
```		       

### 从状态机逻辑

&emsp;&emsp;在HTTP报文中，每一行的数据由\r\n作为结束字符，空行则是仅仅是字符\r\n。因此，可以通过查找\r\n将报文拆解成单独的行进行解析，项目中便是利用了这一点。

&emsp;&emsp;从状态机负责读取buffer中的数据，将每行数据末尾的\r\n置为\0\0，并更新从状态机在buffer中读取的位置m_checked_idx，以此来驱动主状态机解析。

从状态机从m_read_buf中逐字节读取，判断当前字节是否为\r

* 接下来的字符是\n，将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK

* 接下来达到了buffer末尾，表示buffer还需要继续接收，返回LINE_OPEN

* 否则，表示语法错误，返回LINE_BAD	

当前字节不是\r，判断是否是\n（一般是上次读取到\r就到了buffer末尾，没有接收完整，再次接收时会出现这种情况）

* 如果前一个字符是\r，则将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK

当前字节既不是\r，也不是\n

* 表示接收不完整，需要继续接收，返回LINE_OPEN

```cpp
//从状态机，用于分析出一行内容
//返回值为行的读取状态，有LINE_OK,LINE_BAD,LINE_OPEN
//m_read_idx指向缓冲区m_read_buf的数据末尾的下一个字节
//m_checked_idx指向从状态机当前正在分析的字节
http_conn::LINE_STATUS http_conn::parse_line()
{
    char temp;
    for (; m_checked_idx < m_read_idx; ++m_checked_idx)
    {
        //temp为将要分析的字节
        temp = m_read_buf[m_checked_idx];
        //如果当前是\r字符，则有可能会读取到完整行
        if (temp == '\r')
        {
            //下一个字符达到了buffer结尾，则接收不完整，需要继续接收
            if ((m_checked_idx + 1) == m_read_idx)
                return LINE_OPEN;
            //下一个字符是\n，将\r\n改为\0\0
            else if (m_read_buf[m_checked_idx + 1] == '\n')
            {
                m_read_buf[m_checked_idx++] = '\0';
                m_read_buf[m_checked_idx++] = '\0';
                return LINE_OK;
            }
            //如果都不符合，则返回语法错误
            return LINE_BAD;
        }
        //如果当前字符是\n，也有可能读取到完整行
        //一般是上次读取到\r就到buffer末尾了，没有接收完整，再次接收时会出现这种情况
        else if (temp == '\n')
        {
            //前一个字符是\r，则接收完整
            if (m_checked_idx > 1 && m_read_buf[m_checked_idx - 1] == '\r')
            {
                m_read_buf[m_checked_idx - 1] = '\0';
                m_read_buf[m_checked_idx++] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;
        }
    }
    //并没有找到\r\n，需要继续接收
    return LINE_OPEN;
}	
```

### 主状态机逻辑

&emsp;&emsp;主状态机初始状态是CHECK_STATE_REQUESTLINE，通过调用从状态机来驱动主状态机，在主状态机进行解析前，从状态机已经将每一行的末尾\r\n符号改为\0\0，以便于主状态机直接取出对应字符串进行处理。

**CHECK_STATE_REQUESTLINE**

* 主状态机的初始状态，调用parse_request_line函数解析请求行

* 解析函数从m_read_buf中解析HTTP请求行，获得请求方法、目标URL及HTTP版本号

* 解析完成后主状态机的状态变为CHECK_STATE_HEADER

```cpp	
//解析http请求行，获得请求方法，目标url及http版本号
http_conn::HTTP_CODE http_conn::parse_request_line(char* text)
{
    //在HTTP报文中，请求行用来说明请求类型,要访问的资源以及所使用的HTTP版本，
    //其中各个部分之间通过\t或空格分隔。
    //请求行中最先含有空格和\t任一字符的位置并返回
    m_url = strpbrk(text, " \t");
    //如果没有空格或\t，则报文格式有误
    if (!m_url)
    {
        return BAD_REQUEST;
    }
    //将该位置改为\0，用于将前面数据取出
    *m_url++ = '\0';
    //取出数据，并通过与GET和POST比较，以确定请求方式
    char* method = text;
    if (strcasecmp(method, "GET") == 0)
        m_method = GET;
    else if (strcasecmp(method, "POST") == 0)
    {
        m_method = POST;
        cgi = 1;
    }
    else
        return BAD_REQUEST;

    //m_url此时跳过了第一个空格或\t字符，但不知道之后是否还有
    //将m_url向后偏移，通过查找，继续跳过空格和\t字符，指向请求资源的第一个字符
    m_url += strspn(m_url, " \t");
    //使用与判断请求方式的相同逻辑，判断HTTP版本号
    m_version = strpbrk(m_url, " \t");
    if (!m_version)
        return BAD_REQUEST;
    *m_version++ = '\0';
    m_version += strspn(m_version, " \t");
    //仅支持HTTP/1.1
    if (strcasecmp(m_version, "HTTP/1.1") != 0)
        return BAD_REQUEST;

    //对请求资源前7个字符进行判断
    //这里主要是有些报文的请求资源中会带有http://，这里需要对这种情况进行单独处理
    if (strncasecmp(m_url, "http://", 7) == 0)
    {
        m_url += 7;
        m_url = strchr(m_url, '/');
    }

    //同样增加https情况
    if (strncasecmp(m_url, "https://", 8) == 0)
    {
        m_url += 8;
        m_url = strchr(m_url, '/');
    }

    //一般的不会带有上述两种符号，直接是单独的/或/后面带访问资源
    if (!m_url || m_url[0] != '/')
        return BAD_REQUEST;

    //当url为/时，显示判断界面
    if (strlen(m_url) == 1)
        strcat(m_url, "judge.html");

    //请求行处理完毕，将主状态机转移处理请求头
    m_check_state = CHECK_STATE_HEADER;
    return NO_REQUEST;
}	
```	
&emsp;&emsp;解析完请求行后，主状态机继续分析请求头。在报文中，请求头和空行的处理使用的同一个函数，这里通过判断当前的text首位是不是\0字符，若是，则表示当前处理的是空行，若不是，则表示当前处理的是请求头。
	
**CHECK_STATE_HEADER**

* 调用parse_headers函数解析请求头部信息

* 判断是空行还是请求头，若是空行，进而判断content-length是否为0，如果不是0，表明是POST请求，则状态转移到CHECK_STATE_CONTENT，否则说明是GET请求，则报文解析结束。

* 若解析的是请求头部字段，则主要分析connection字段，content-length字段，其他字段可以直接跳过，各位也可以根据需求继续分析。

* connection字段判断是keep-alive还是close，决定是长连接还是短连接

* content-length字段，这里用于读取post请求的消息体长度	

**parse_headers函数**
	
```cpp
//解析http请求的一个头部信息
http_conn::HTTP_CODE http_conn::parse_headers(char* text)
{
    //判断是空行还是请求头
    if (text[0] == '\0')
    {
        //判断是GET还是POST请求
        if (m_content_length != 0)
        {
            //POST需要跳转到消息体处理状态
            m_check_state = CHECK_STATE_CONTENT;
            return NO_REQUEST;
        }
        return GET_REQUEST;
    }
    //解析请求头部连接字段
    else if (strncasecmp(text, "Connection:", 11) == 0)
    {
        text += 11;
        //跳过空格和\t字符
        text += strspn(text, " \t");
        if (strcasecmp(text, "keep-alive") == 0)
        {
            //如果是长连接，则将linger标志设置为true
            m_linger = true;
        }
    }
    //解析请求头部内容长度字段
    else if (strncasecmp(text, "Content-length:", 15) == 0)
    {
        text += 15;
        text += strspn(text, " \t");
        m_content_length = atol(text);
    }
    //解析请求头部HOST字段
    else if (strncasecmp(text, "Host:", 5) == 0)
    {
        text += 5;
        text += strspn(text, " \t");
        m_host = text;
    }
    else
    {
        //printf("oop!unknow header: %s\n",text);
        LOG_INFO("oop!unknow header: %s", text);
        Log::get_instance()->flush();
    }
    return NO_REQUEST;
}	
```

如果仅仅是GET请求，如项目中的欢迎界面，那么主状态机只设置之前的两个状态足矣。

GET和POST请求报文的区别之一是有无消息体部分，GET请求没有消息体，当解析完空行之后，便完成了报文的解析。	
	
但后续的登录和注册功能，为了避免将用户名和密码直接暴露在URL中，我们在项目中改用了POST请求，将用户名和密码添加在报文中作为消息体进行了封装。

为此，我们需要在解析报文的部分添加解析消息体的模块。
	
```cpp
while((m_check_state==CHECK_STATE_CONTENT && line_status==LINE_OK)||((line_status=parse_line())==LINE_OK))
```	
那么，这里的判断条件为什么要写成这样呢？

在GET请求报文中，每一行都是\r\n作为结束，所以对报文进行拆解时，仅用从状态机的状态line_status=parse_line())==LINE_OK语句即可。

但，在POST请求报文中，消息体的末尾没有任何字符，所以不能使用从状态机的状态，这里转而使用主状态机的状态作为循环入口条件。

**那后面的&& line_status==LINE_OK又是为什么？**

解析完消息体后，报文的完整解析就完成了，但此时主状态机的状态还是CHECK_STATE_CONTENT，也就是说，符合循环入口条件，还会再次进入循环，这并不是我们所希望的。

为此，增加了该语句，并在完成消息体解析后，将line_status变量更改为LINE_OPEN，此时可以跳出循环，完成报文解析任务。

**CHECK_STATE_CONTENT**

* 仅用于解析POST请求，调用parse_content函数解析消息体

* 用于保存post请求消息体，为后面的登录和注册做准备
	
```cpp
//判断http请求是否被完整读入
http_conn::HTTP_CODE http_conn::parse_content(char* text)
{
    //判断buffer中是否读取了消息体
    if (m_read_idx >= (m_content_length + m_checked_idx))
    {
        text[m_content_length] = '\0';
        //POST请求中最后为输入的用户名和密码
        m_string = text;
        return GET_REQUEST;
    }
    return NO_REQUEST;
}	
```	

状态机和HTTP报文解析是项目中最繁琐的部分。	
	



**服务器接收http请求**
	
浏览器端发出http连接请求，主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列，工作线程从任务队列中取出一个任务进行处理。

	
	
	
	
	
	

