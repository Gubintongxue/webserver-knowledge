## 目录

- [00 项目概述](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649273988&idx=1&sn=cc169061e6bb3ca7d14ea138c7706c14&chksm=83ffbfdcb48836cab6833dadd3d4d2ee3b7346bdab791f5c9658714d1907b7104f3ed1ad7883&scene=21#wechat_redirect)
- [01 线程同步机制包装类](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649273996&idx=2&sn=f7de38b997c2217998be508525bc3836&chksm=83ffbfd4b48836c217e4dafc6d77657ea3e55589efe0ec86927fda2d392cc08af1d52e007378&scene=21#wechat_redirect)
- [02 半同步/半反应堆线程池（上）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274118&idx=1&sn=f2f889b83c760f18be0eedbd6a1ebe8b&chksm=83ffbe5eb4883748001b78c47e183900730570edcd07e3279b8d6857f0a2ce568494cbba250f&scene=21#wechat_redirect)
- [03 半同步/半反应堆线程池（下）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274139&idx=1&sn=e4a0c8d35927cc24253e840d22994e3f&chksm=83ffbe43b48837552265d980a59d7fa288c50d56c809d12f132cfefa106e33d90a3e0dd4b315&scene=21#wechat_redirect)
- **04 http连接处理（上）**
- 05 http连接处理（中）
- 06 http连接处理（下）
- 07 定时器处理非活动连接（上）
- 08 定时器处理非活动连接（下）
- 09 日志系统（上）
- 10 日志系统（下）
- 11 数据连接池
- 12 注册和登录校验
- 13 服务器测试
- 14 项目遇到的问题及解决方案
- 15 项目涉及的常见面试题

------

### **写在前面**

感谢各位看官老爷的耐心等待，最近我在**肝毕业论文和华为软挑的热身赛**，导致公号的推文推迟了一周多。

在此，向各位表示**抱歉**。

目前毕业论文已经提交，华为比赛也进入最后的调优阶段，成绩在**0.1**附近浮动。由于我不再参加后续的初赛，31号热身赛结束后，会将**代码开源**，也会**做一次抽奖**，大家可以期待一下。

后面，我会尽快将服务器项目讲解系列更新完，助力大家备战秋招。

------

### **本文内容**

在服务器项目中，http请求的处理与响应至关重要，关系到用户界面的跳转与反馈。这里，社长将其分为上、中、下三个部分来讲解，具体的：

- **上篇，梳理基础知识，结合代码分析http类及请求接收**
- **中篇，结合代码分析请求报文解析**
- **下篇，结合代码分析请求报文响应**

**基础知识方面**，包括HTTP报文格式，状态码和有限状态机。

**代码分析方面**，首先对服务器端处理http请求的全部流程进行简要介绍，然后结合代码对http类及请求接收进行详细分析。

------

### **HTTP报文格式**

HTTP报文分为请求报文和响应报文两种，每种报文必须按照特有格式生成，才能被浏览器端识别。

其中，浏览器端向服务器发送的为请求报文，服务器处理后返回给浏览器端的为响应报文。

#### 请求报文

HTTP请求报文由请求行（request line）、请求头部（header）、空行和请求数据四个部分组成。

其中，请求分为两种，GET和POST，具体的：

- **GET**

```
    GET /562f25980001b1b106000338.jpg HTTP/1.1
    Host:img.mukewang.com
    User-Agent:Mozilla/5.0 (Windows NT 10.0; WOW64)
    AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36
    Accept:image/webp,image/*,*/*;q=0.8
    Referer:http://www.imooc.com/
    Accept-Encoding:gzip, deflate, sdch
    Accept-Language:zh-CN,zh;q=0.8
    空行
    请求数据为空
```

- **POST**

```
    POST / HTTP1.1
    Host:www.wrox.com
    User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
    Content-Type:application/x-www-form-urlencoded
    Content-Length:40
    Connection: Keep-Alive
    空行
    name=Professional%20Ajax&publisher=Wiley
```

> - **请求行**，用来说明请求类型,要访问的资源以及所使用的HTTP版本。
>   GET说明请求类型为GET，/562f25980001b1b106000338.jpg(URL)为要访问的资源，该行的最后一部分说明使用的是HTTP1.1版本。
>
> - **请求头部**，紧接着请求行（即第一行）之后的部分，用来说明服务器要使用的附加信息。
>
> - - HOST，给出请求资源所在服务器的域名。
>   - User-Agent，HTTP客户端程序的信息，该信息由你发出请求使用的浏览器来定义,并且在每个请求中自动发送等。
>   - Accept，说明用户代理可处理的媒体类型。
>   - Accept-Encoding，说明用户代理支持的内容编码。
>   - Accept-Language，说明用户代理能够处理的自然语言集。
>   - Content-Type，说明实现主体的媒体类型。
>   - Content-Length，说明实现主体的大小。
>   - Connection，连接管理，可以是Keep-Alive或close。
>
> - **空行**，请求头部后面的空行是必须的即使第四部分的请求数据为空，也必须有空行。
>
> - **请求数据**也叫主体，可以添加任意的其他数据。

#### 响应报文

HTTP响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

```
HTTP/1.1 200 OK
Date: Fri, 22 May 2009 06:07:21 GMT
Content-Type: text/html; charset=UTF-8
<html>
      <head></head>
      <body>
            <!--body goes here-->
      </body>
</html>
```

> - 状态行，由HTTP协议版本号， 状态码， 状态消息 三部分组成。
>   第一行为状态行，（HTTP/1.1）表明HTTP版本为1.1版本，状态码为200，状态消息为OK。
> - 消息报头，用来说明客户端要使用的一些附加信息。
>   第二行和第三行为消息报头，Date:生成响应的日期和时间；Content-Type:指定了MIME类型的HTML(text/html),编码类型是UTF-8。
> - 空行，消息报头后面的空行是必须的。
> - 响应正文，服务器返回给客户端的文本信息。空行后面的html部分为响应正文。

------

### **HTTP状态码**

HTTP有5种类型的状态码，具体的：

- 1xx：指示信息--表示请求已接收，继续处理。

- 2xx：成功--表示请求正常处理完毕。

- - 200 OK：客户端请求被正常处理。
  - 206 Partial content：客户端进行了范围请求。

- 3xx：重定向--要完成请求必须进行更进一步的操作。

- - 301 Moved Permanently：永久重定向，该资源已被永久移动到新位置，将来任何对该资源的访问都要使用本响应返回的若干个URI之一。
  - 302 Found：临时重定向，请求的资源现在临时从不同的URI中获得。

- 4xx：客户端错误--请求有语法错误，服务器无法处理请求。

- - 400 Bad Request：请求报文存在语法错误。
  - 403 Forbidden：请求被服务器拒绝。
  - 404 Not Found：请求不存在，服务器上找不到请求的资源。

- 5xx：服务器端错误--服务器处理请求出错。

- - 500 Internal Server Error：服务器在执行请求时出现错误。
  - 503 Service Unavaliable：服务器正在停机维护。

------

### **有限状态机**

有限状态机，是一种抽象的理论模型，它能够把有限个变量描述的状态变化过程，以可构造可验证的方式呈现出来。比如，封闭的有向图。

有限状态机可以通过if-else,switch-case和函数指针来实现，从软件工程的角度看，主要是为了封装逻辑。

带有状态转移的有限状态机示例代码。

```
STATE_MACHINE(){
    State cur_State = type_A;
    while(cur_State != type_C){
        Package _pack = getNewPackage();
        switch(){
            case type_A:
                process_pkg_state_A(_pack);
                cur_State = type_B;
                break;
            case type_B:
                process_pkg_state_B(_pack);
                cur_State = type_C;
                break;
        }
    }
}
```


该状态机包含三种状态：type_A，type_B和type_C。其中，type_A是初始状态，type_C是结束状态。

状态机的当前状态记录在cur_State变量中，逻辑处理时，状态机先通过getNewPackage获取数据包，然后根据当前状态对数据进行处理，处理完后，状态机通过改变cur_State完成状态转移。

有限状态机一种逻辑单元内部的一种高效编程方法，在服务器编程中，服务器可以根据不同状态或者消息类型进行相应的处理逻辑，使得程序逻辑清晰易懂。

------

### **http处理流程**

首先对http报文处理的流程进行简要介绍，然后具体介绍http类的定义和服务器接收http请求的具体过程。

#### http报文处理流程

- 浏览器端发出http连接请求，主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列，工作线程从任务队列中取出一个任务进行处理。(**本篇讲**)
- 工作线程取出任务后，调用process_read函数，通过主、从状态机对请求报文进行解析。(**中篇讲**)
- 解析完之后，跳转do_request函数生成响应报文，通过process_write写入buffer，返回给浏览器端。(**下篇讲**)

#### http类

这一部分代码在TinyWebServer/http/http_conn.h中，主要是http类的定义。

```
class http_conn{
    public:
        //设置读取文件的名称m_real_file大小
        static const int FILENAME_LEN=200;
        //设置读缓冲区m_read_buf大小
        static const int READ_BUFFER_SIZE=2048;
        //设置写缓冲区m_write_buf大小
        static const int WRITE_BUFFER_SIZE=1024;
        //报文的请求方法，本项目只用到GET和POST
        enum METHOD{GET=0,POST,HEAD,PUT,DELETE,TRACE,OPTIONS,CONNECT,PATH};
        //主状态机的状态
        enum CHECK_STATE{CHECK_STATE_REQUESTLINE=0,CHECK_STATE_HEADER,CHECK_STATE_CONTENT};
        //报文解析的结果
        enum HTTP_CODE{NO_REQUEST,GET_REQUEST,BAD_REQUEST,NO_RESOURCE,FORBIDDEN_REQUEST,FILE_REQUEST,INTERNAL_ERROR,CLOSED_CONNECTION};
        //从状态机的状态
        enum LINE_STATUS{LINE_OK=0,LINE_BAD,LINE_OPEN};

    public:
        http_conn(){}
        ~http_conn(){}

    public:
        //初始化套接字地址，函数内部会调用私有方法init
        void init(int sockfd,const sockaddr_in &addr);
        //关闭http连接
        void close_conn(bool real_close=true);
        void process();
        //读取浏览器端发来的全部数据
        bool read_once();
        //响应报文写入函数
        bool write();
        sockaddr_in *get_address(){
            return &m_address;  
        }
        //初始化数据库读取表
        void initmysql_result();

    private:
        void init();
        //从m_read_buf读取，并处理请求报文
        HTTP_CODE process_read();
        //向m_write_buf写入响应报文数据
        bool process_write(HTTP_CODE ret);
        //主状态机解析报文中的请求行数据
        HTTP_CODE parse_request_line(char *text);
        //主状态机解析报文中的请求头数据
        HTTP_CODE parse_headers(char *text);
        //主状态机解析报文中的请求内容
        HTTP_CODE parse_content(char *text);
        //生成响应报文
        HTTP_CODE do_request();

        //m_start_line是已经解析的字符
        //get_line用于将指针向后偏移，指向未处理的字符
        char* get_line(){return m_read_buf+m_start_line;};

        //从状态机读取一行，分析是请求报文的哪一部分
        LINE_STATUS parse_line();

        void unmap();

        //根据响应报文格式，生成对应8个部分，以下函数均由do_request调用
        bool add_response(const char* format,...);
        bool add_content(const char* content);
        bool add_status_line(int status,const char* title);
        bool add_headers(int content_length);
        bool add_content_type();
        bool add_content_length(int content_length);
        bool add_linger();
        bool add_blank_line();

    public:
        static int m_epollfd;
        static int m_user_count;

    private:
        int m_sockfd;
        sockaddr_in m_address;

        //存储读取的请求报文数据
        char m_read_buf[READ_BUFFER_SIZE];
        //缓冲区中m_read_buf中数据的最后一个字节的下一个位置
        int m_read_idx;
        //m_read_buf读取的位置m_checked_idx
        int m_checked_idx;
        //m_read_buf中已经解析的字符个数
        int m_start_line;

        //存储发出的响应报文数据
        char m_write_buf[WRITE_BUFFER_SIZE];
        //指示buffer中的长度
        int m_write_idx;

        //主状态机的状态
        CHECK_STATE m_check_state;
        //请求方法
        METHOD m_method;

        //以下为解析请求报文中对应的6个变量
        //存储读取文件的名称
        char m_real_file[FILENAME_LEN];
        char *m_url;
        char *m_version;
        char *m_host;
        int m_content_length;
        bool m_linger;

        char *m_file_address;        //读取服务器上的文件地址
        struct stat m_file_stat;
        struct iovec m_iv[2];        //io向量机制iovec
        int m_iv_count;
        int cgi;                     //是否启用的POST
        char *m_string;              //存储请求头数据
};
```

在http请求接收部分，会涉及到init和read_once函数，但init仅仅是对私有成员变量进行初始化，不用过多讲解。

这里，对read_once进行介绍。read_once读取浏览器端发送来的请求报文，直到无数据可读或对方关闭连接，读取到m_read_buffer中，并更新m_read_idx。

```
//循环读取客户数据，直到无数据可读或对方关闭连接
bool http_conn::read_once()
{
    if(m_read_idx>=READ_BUFFER_SIZE)
    {
        return false;
    }
    int bytes_read=0;
    while(true)
    {
        //从套接字接收数据，存储在m_read_buf缓冲区
        bytes_read=recv(m_sockfd,m_read_buf+m_read_idx,READ_BUFFER_SIZE-m_read_idx,0);
        if(bytes_read==-1)
        {    
            //非阻塞ET模式下，需要一次性将数据读完
            if(errno==EAGAIN||errno==EWOULDBLOCK)
                break;
            return false;
        }
        else if(bytes_read==0)
        {
            return false;
        }
        //修改m_read_idx的读取字节数
        m_read_idx+=bytes_read;
    }
    return true;
}
```

#### 服务器接收http请求

浏览器端发出http连接请求，主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列，工作线程从任务队列中取出一个任务进行处理。下面，对前述流程进行具体介绍。

- 主线程创建多个http类对象

```
//main.c 第120行
//创建MAX_FD个http类对象
http_conn* users=new http_conn[MAX_FD];
```

- 接收http请求，初始化http对象

```
//main.c 第194行
//接收http请求生成连接套接字
int connfd=accept(listenfd,(struct sockaddr*)&client_address,&client_addrlength);

//main.c 第207行
//对上述类对象进行初始化
users[connfd].init(connfd,client_address);
```

- 监测到读事件，将数据读入到对应缓冲区m_read_buf，将该对象插入到请求队列

```
//main.c 第271行
//监测读事件
if(events[i].events&EPOLLIN){
    //读入对应缓冲区
    if(users[sockfd].read_once()){ 
        //通过sockfd偏移，将该对象放入任务队列
        pool->append(users+sockfd);
    }
}
```

------

完。