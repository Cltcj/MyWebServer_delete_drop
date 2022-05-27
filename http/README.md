## epoll 系列系统调用

![image](https://user-images.githubusercontent.com/81791654/170219359-1913b1ed-886c-439a-a95a-63542a6c7556.png)

**epoll_create函数**

```cpp
#include <sys/epoll.h>
int epoll_create(int size)
```

创建一个指示epoll内核事件表的文件描述符，该描述符将用作其他epoll系统调用的第一个参数，size不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符将用其他所有 epoll系统调用的第一个参数，以指定要访问的内核事件表。

下面的函数用来操作 epoll 的内核事件表：

**epoll_ctl函数**

```cpp
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
```

该函数用于操作内核事件表监控的文件描述符上的事件：注册、修改、删除

* epfd：为epoll_creat的句柄

* op：表示动作，用3个宏来表示：

EPOLL_CTL_ADD (注册新的fd到epfd)，

EPOLL_CTL_MOD (修改已经注册的fd的监听事件)，

EPOLL_CTL_DEL (从epfd删除一个fd)；

* fd是要操作的文件描述符

* op参数则指定操作类型。操作类型有如下3种：

* event：告诉内核需要监听的事件

上述event是epoll_event结构体指针类型，表示内核所监听的事件，具体定义如下：

```cpp
struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```
events描述事件类型，其中epoll事件类型有以下几种

* EPOLLIN：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）

* EPOLLOUT：表示对应的文件描述符可以写

* EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）

* EPOLLERR：表示对应的文件描述符发生错误

* EPOLLHUP：表示对应的文件描述符被挂断；

* EPOLLET：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)而言的

* EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里

![image](https://user-images.githubusercontent.com/81791654/170411993-90fc9414-6f0f-44dc-a88c-fb7ce8c6c447.png)

**epoll_wait函数**

```cpp
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

该函数用于等待所监控文件描述符上有事件的产生，成功返回就绪的文件描述符个数，失败返回-1并设置errno

epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样即用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

events：用来存内核得到事件的集合，

maxevents：告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，但必须大于0

timeout：是超时时间，和poll中的timeout参数相同。

* -1：阻塞

* 0：立即返回，非阻塞

* >0：指定毫秒

返回值：成功返回有多少文件描述符就绪，时间到时返回0，出错返回-1

### LT 和 ET模式

![image](https://user-images.githubusercontent.com/81791654/170442531-4c46916f-d849-48c0-9d1e-c99c42d8f8b3.png)

**LT水平触发模式**

epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序可以不立即处理该事件。

当下一次调用epoll_wait时，epoll_wait还会再次向应用程序报告此事件，直至被处理

**ET边缘触发模式**

epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序必须立即处理该事件

必须要一次性将数据读取完，使用非阻塞I/O，读取到出现eagain

### EPOLLONESHOT

![image](https://user-images.githubusercontent.com/81791654/170443912-62ceed4d-c4c5-462a-8148-7ab0c9657626.png)

一个线程读取某个socket上的数据后开始处理数据，在处理过程中该socket上又有新数据可读，此时另一个线程被唤醒读取，此时出现两个线程处理同一个socket

我们期望的是一个socket连接在任一时刻都只被一个线程处理，通过epoll_ctl对该文件描述符注册epolloneshot事件，一个线程处理socket时，其他线程将无法处理，当该线程处理完后，需要通过epoll_ctl重置epolloneshot事件

![image](https://user-images.githubusercontent.com/81791654/170444562-418206bc-48b3-42f3-8289-228e5bbe9800.png)

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

### 有限状态机

有限状态机，是一种抽象的理论模型，它能够把有限个变量描述的状态变化过程，以可构造可验证的方式呈现出来。比如，封闭的有向图。

有限状态机可以通过if-else,switch-case和函数指针来实现，从软件工程的角度看，主要是为了封装逻辑。

带有状态转移的有限状态机示例代码：

![image](https://user-images.githubusercontent.com/81791654/170452323-d60dc672-f583-48a1-9c41-895058efa966.png)

![image](https://user-images.githubusercontent.com/81791654/170452348-274a6d23-68f4-4abc-b848-0671dcdb8d7a.png)

![image](https://user-images.githubusercontent.com/81791654/170452565-faf214ab-3216-4e1f-98f0-c482eaa1012f.png)


关于这部分代码能够参考[http_conn的代码](https://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274278&idx=6&sn=b0a34b4f59f28b0619dcc72f3fcc2243&chksm=83ffbefeb48837e8ae419ae1cbf61b112f24378c6db96a943c5f6671b1e4ab16ea83b4302a1e&scene=178&cur_album_id=1339230165934882817#rd)

&emsp;&emsp;很多网络协议，包括TCP协议和IP协议，都在其头部中提供头部长度字段。程序根据该字段的值就可以知道是否接收到一个完整的协议头部。但HTTP协议并未提供这样的头部长度字段，并且其头部长度变化也很大，可以只有十几字节，也可以有上百字节。根据协议规定，我们判断HTTP头部结束的依据是遇到一个空行，该空行仅包含一对回车换行符（<CR><LF>）。如果一次读操作没有读入HTTP请求的整个头部，即没有遇到空行，那么我们必须等待客户继续写数据并再次读入。因此，我们每完成一次读操作，就要分析新读入的数据中是否有空行。不过在寻找空行的过程中，我们可以同时完成对整个HTTP请求头部的分析（记住，空行前面还有请求行和头部域），以提高解析HTTP请求的效率。
	
&emsp;&emsp;结合有限状态机，我们采用主、从两个有限状态机实现了最简单的HTTP请求的读取和分析。

**http_conn.h类的定义**

```cpp
class http_conn {
public:
	/* 文件名的最大长度 */
	static const int FILENAME_LEN = 200;
	/* 读缓冲区的大小 */
	static const int READ_BUFFER_SIZE = 2048;
	/* 写缓冲区的大小 */
	static const int WRITE_BUFFER_SIZE = 1024;
	enum METHOD {//HTTP请求方法
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
	enum CHECK_STATE {//主状态机的三种可能状态
		//主状态机使用checkstate变量来记录当前的状态
		CHECK_STATE_REQUESTLINE = 0,//表示parse_line函数解析出的行是请求行
		CHECK_STATE_HEADER,//parse_line函数解析出的是头部字段
		CHECK_STATE_CONTENT//parse_line函数解析出的是内容字段
	};
	enum HTTP_CODE {
		/*
		服务器处理HTTP请求的结果：
		*/
		NO_REQUEST,//请求不完整，需要继续读取客户数据
		GET_REQUEST,//获得了一个完整的客户请求
		BAD_REQUEST,//客户请求有语法错误
		NO_RESOURCE,
		FORBIDDEN_REQUEST,//客户对资源没有足够的访问权限
		FILE_REQUEST,
		INTERNAL_ERROR,//服务器内部错误
		CLOSED_CONNECTION//客户端已经关闭连接
	};
	enum LINE_STATUS {
	/*
	从状态机的三种可能状态，即行的读取状态，
	分别表示：读取到一个完整的行、行出错
	和行数据尚且不完整
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
	void init(int sockfd, const sockaddr_in &addr, char *, int, int, string user, string passwd, string sqlname);
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
	int timer_flag;
	int improv;

private:
	/* 初始化连接 */
	void init();
	//process线程工作主函数（包括Process read和write）
	//解析请求报文主函数while根据状态调用不同解析函数
	HTTP_CODE process_read();
	//向m_write_buf写入响应报文数据
	bool process_write(HTTP_CODE ret);

	/* 下面这一组函数被process_read调用以分析HTTP请求 */
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
	/*
		//根据响应报文格式，
		生成对应8个部分，
		以下函数均由do_request调用
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
	int m_state; //读为0，写为1

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
	struct iovec m_iv[2];//io向量机制iovec
	int m_iv_count;
	int cgi;        //是否启用的POST
	char* m_string; //存储请求头数据
	int bytes_to_send;//剩余发送字节数
	int bytes_have_send;//已发送字节数
	char* doc_root;

	map<string, string> m_users;
	int m_TRIGMode;
	int m_close_log;

	char sql_user[100];
	char sql_passwd[100];
	char sql_name[100];
};
```

	
	
**epoll相关代码**
	
项目中epoll相关代码部分包括非阻塞模式、内核事件表注册事件、删除事件、重置EPOLLONESHOT事件四种。

* 非阻塞模式

```cpp
//对文件描述符设置非阻塞
int setnonblocking(int fd) {
	int old_option = fcntl(fd, F_GETFL);
	int new_option = old_option | O_NONBLOCK;
	fcntl(fd, F_SETFL, new_option);
	return old_option;
}	
```

**内核事件表注册新事件，开启EPOLLONESHOT，针对客户端连接的描述符，listenfd不用开启**
	
```cpp
//将内核事件表注册读事件，EF模式，选择开启EPOLLONESHOT
void addfd(int epollfd, int fd, bool one_shot, int TRIGMode) {
    epoll_event event;
    event.data.fd = fd;

    if (1 == TRIGMode)
        event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
    else
        event.events = EPOLLIN | EPOLLRDHUP;

    if (one_shot)
        event.events |= EPOLLONESHOT;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}
```	

**内核事件表删除事件**

```cpp
//从内核时间表删除描述符
void removefd(int epollfd, int fd) {
    epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, 0);
    close(fd);
}
```
	
**重置EPOLLONESHOT事件**

```cpp
//将事件重置为EPOLLONESHOT
void modfd(int epollfd, int fd, int ev, int TRIGMode) {
    epoll_event event;
    event.data.fd = fd;

    if (1 == TRIGMode)
        event.events = ev | EPOLLET | EPOLLONESHOT | EPOLLRDHUP;
    else
        event.events = ev | EPOLLONESHOT | EPOLLRDHUP;

    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}
```

**服务器接收http请求**
	
	
	
	
	
	
	

	
	
	
	
	
	
	
	
	
	
	
	
![image](https://user-images.githubusercontent.com/81791654/170648325-c2101de6-e6de-4ead-a1f9-891421eab2b3.png)
	
![image](https://user-images.githubusercontent.com/81791654/170648708-f6d9d606-ee4e-4ed1-8f79-a29d2cf04b12.png)







