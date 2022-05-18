# HTTP—Hyper Text Transfer Protocol（超文本传输协议）

HTTP（HyperText Transfer Protocol，超文本传输协议）是一种用于分布式、协作式和超媒体信息系统的应用层协议。HTTP 是万维网的数据通信的基础。

HTTP_CODE含义
表示HTTP请求的处理结果，在头文件中初始化了八种情形

* NO_REQUEST

请求不完整，需要继续读取请求报文数据

* GET_REQUEST

获得了完整的HTTP请求

* NO_RESOURCE

请求资源不存在

* BAD_REQUEST

HTTP请求报文有语法错误或请求资源为目录

* FORBIDDEN_REQUEST

请求资源禁止访问，没有读取权限

* FILE_REQUEST

请求资源可以正常访问

* INTERNAL_ERROR

服务器内部错误
