## 多路IO技术--select

多路IO技术: select, 同时监听多个文件描述符, 将监控的操作交给内核去处理,数据类型fd_set: 文件描述符集合--本质是位图(关于集合可联想一个信号集sigset_t)

int select(int nfds, fd_set * readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
