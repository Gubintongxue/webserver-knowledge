本篇文章主要介绍`Http`和`Buffer`部分  
`Http`请求和响应部分，其主要包含三个类`httpConn`, `httpRequest`, `httpResponse`  
`Buffer`解决的是并发情况下的读写问题

## 1\. http请求和响应报文

首先让我们熟悉一下请求（Request）和响应（Response）报文，这两种报文在`WebServer`服务器中分别对应`httpRequest`和`httpResponse`类进行处理。

### 1.1 http请求报文

HTTP请求报文由四部分组成：

-   请求行
-   请求头部
-   空行
-   请求数据

请求分为两种，GET和POST，让我们来看一个例子  
GET

```http
（请求行部分）
GET /search?hl=zh-CN&source=hp&q=domety&aq=f&oq= HTTP/1.1  
（请求头部分）
Accept: image/gif, image/x-xbitmap, image/jpeg, image/pjpeg, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, application/x-silverlight, application/x-shockwave-flash, */*  
Referer: <a href="http://www.google.cn/">http://www.google.cn/</a>  
Accept-Language: zh-cn  
Accept-Encoding: gzip, deflate  
User-Agent: Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; TheWorld)  
Host: <a href="http://www.google.cn">www.google.cn</a>  
Connection: Keep-Alive  
Cookie: PREF=ID=80a06da87be9ae3c:U=f7167333e2c3b714:NW=1:TM=1261551909:LM=1261551917:S=ybYcq2wpfefs4V9g; NID=31=ojj8d-IygaEtSxLgaJmqSjVhCspkviJrB6omjamNrSm8lZhKy_yMfO2M4QMRKcH1g0iQv9u-2hfBW7bUFwVh7pGaRUb0RnHcJU37y-FxlRugatx63JLv7CWMD6UB_O_r
（空行）
（请求数据）
```

POST

```http
POST / HTTP1.1
User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
Content-Type:application/x-www-form-urlencoded
Content-Length:40
Connection: Keep-Alive
空行
name=Professional%20Ajax&publisher=Wiley
```

### 1.2 http响应报文

http响应报文由四部分组成

-   状态行
-   消息报头
-   空行
-   响应正文  
    一个例子如下

```http
（状态行）
HTTP/1.1 200 OK
（消息报头）
Date: Sat, 31 Dec 2005 23:59:59 GMT
Content-Type: text/html;charset=ISO-8859-1
Content-Length: 122
（空行）
（响应正文）
＜html＞
＜head＞
＜title＞Wrox Homepage＜/title＞
＜/head＞
＜body＞
＜!-- body goes here --＞
＜/body＞
＜/html＞
```

## 2 `httpConn`类

`httpConn`类处理的是和`http`连接相关的问题，它用于处理和`client`的连接  
其成员变量有

```c++
    static bool isET; // 是否是边缘触发
    static const char* srcDir; //根路径
    static std::atomic<int> userCount; //总用户数

    int fd_; //文件描述符
    struct  sockaddr_in addr_; //连接时的socket地址
    
    bool isClose_; //是否关闭
    
    int iovCnt_; //供readv/writev使用，表示缓冲区的个数
    struct iovec iov_[2]; //供readv/writev使用，表示缓冲区地址(iov_base)和长度(iov_len)的结构体
    
    Buffer readBuff_; // 读缓冲区
    Buffer writeBuff_; // 写缓冲区

    HttpRequest request_; // httpRequest对象
    HttpResponse response_; // httpResponse对象
```

从成员变量中就可以看出，`httpConn`就是`http`连接的一个表示，因为成员函数就显而易见了

-   `init/close`: 初始化`httpConn`和关闭文件
-   `read/write`:读取数据到`readBuff`中/将`writeBuff`中写出
-   `process`: 解析`readBuff`，将相应的`response`写到`writeBuff`
-   `IsKeepAlive`: 是否还保持连接，通过`request_`的`isKeepAlive`进行判断
-   `get`相关的方法: 获得地址/端口号等等

### 2.1 `init`和`close`

`init`: 初始化成员变量和静态变量，例如`userCount++`,设置`fd`, `addr`，初始化`writeBuff`和`readBuff`等等  
`close`: 解除`response_`建立的文件映射，`userCount--`,关闭`fd`，设置`isClosed`，在析构函数中会调用`close`

### 2.2 `read`

```c++
ssize_t HttpConn::read(int* saveErrno) {
    ssize_t len = -1;
    do {
        len = readBuff_.ReadFd(fd_, saveErrno);
        if (len <= 0) {
            break;
        }
    } while (isET);
    return len;
}
```

### 2.3 `write`

当`fd`注册`EPOLLIN`事件时，当TCP读缓冲区有数据到达时就会触发`EPOLLIN`事件。当`fd`注册`EPOLLOUT`事件时，当TCP写缓冲区有剩余空间时就会触发`EPOLLOUT`事件，此时`DealWrite`就是处理`EPOLLOUT`事件。  
这里有一个问题是，为什么要放到循环中?

> 查看`writev` 的用法有提到  
> When using non-blocking I/O on objects, such as sockets, that are subject to flow control, write() and writev() may write fewer bytes than requested; the return value must be noted, and the remainder of the operation should be retried when possible

`writev`是分散写,也就是你的数据可以这里一块,那里一块,然后只要将这些数据的首地址,长度什么的写到一个`iovc`的结构体数组里,传递给`writev`,`writev`就帮你来写这些数据,在你看来,分散的数据就像是连成了一体。但是在非阻塞IO的情况下，如果writev返回一个大于0的值num,这个值又小于所有要传递的文件块的总长度,这意味着什么,意味着数据还没有写完。如果你还想写的话,你下一次调用writev的时候要重新整理iovc数组

```
ssize_t HttpConn::write(int* saveErrno) {
    ssize_t len = -1;
    do {
        len = writev(fd_, iov_, iovCnt_);
        if(len <= 0) {
            *saveErrno = errno;
            break;
        }
        if(iov_[0].iov_len + iov_[1].iov_len  == 0) { break; } /* 传输结束 */
        else if(static_cast<size_t>(len) > iov_[0].iov_len) {
            iov_[1].iov_base = (uint8_t*) iov_[1].iov_base + (len - iov_[0].iov_len);
            iov_[1].iov_len -= (len - iov_[0].iov_len);
            if(iov_[0].iov_len) {
                writeBuff_.RetrieveAll();
                iov_[0].iov_len = 0;
            }
        }
        else {
            iov_[0].iov_base = (uint8_t*)iov_[0].iov_base + len; 
            iov_[0].iov_len -= len; 
            writeBuff_.Retrieve(len);
        }
    } while(isET || ToWriteBytes() > 10240);
    return len;
}
```

### 2.4 `process`函数

-   初始化`request_`: `request_.init()`
-   解析`readBuff_`: `request.parse(readBuff_)`,同时用解析的内容初始化`response_`
-   生成回应内容: `response_.makeResponse()`
-   更新`iov`和`iov_cnt`: `iov[0]`存储的是响应头，`iov[1]`存储的是响应文件`response_.File()`

## 3 `httpRequest`类

我们仍旧从成员变量开始分析

```
    PARSE_STATE state_; // 状态
    std::string method_, path_, version_, body_; //方法，路径，版本，请求体
    std::unordered_map<std::string, std::string> header_; //请求头
    std::unordered_map<std::string, std::string> post_; //post的内容

    static const std::unordered_set<std::string> DEFAULT_HTML;
    static const std::unordered_map<std::string, int> DEFAULT_HTML_TAG;
```

可以看出`httpRequest`是对`socket`解析到的`http`状态的解析和保存。最主要的是`parse`函数，剩下的就约等于字符串的处理，将处理的字符串保存在成员变量中。

### 3.1 `parse`函数

其利用有限状态机模型，每次解析一行，从已知状态跳转到下一个状态，可以参考1.1中的`http`请求报文

```c++
bool HttpRequest::parse(Buffer& buff) {
   const char CRLF[] = "\r\n";
   if(buff.ReadableBytes() <= 0) {
       return false;
   }
   while(buff.ReadableBytes() && state_ != FINISH) {
       const char* lineEnd = search(buff.Peek(), buff.BeginWriteConst(), CRLF, CRLF + 2);
       std::string line(buff.Peek(), lineEnd);
       switch(state_)
       {
       case REQUEST_LINE:
           if(!ParseRequestLine_(line)) {
               return false;
           }
           ParsePath_(); // state在其中会被置为HEADERS
           break;    
       case HEADERS:
           ParseHeader_(line); // state被置为BODY
           if(buff.ReadableBytes() <= 2) {
               state_ = FINISH;
           }
           break;
       case BODY:
           ParseBody_(line);
           break;
       default:
           break;
       }
       if(lineEnd == buff.BeginWrite()) { break; }
       buff.RetrieveUntil(lineEnd + 2);
   }
   LOG_DEBUG("[%s], [%s], [%s]", method_.c_str(), path_.c_str(), version_.c_str());
   return true;
}
```

### 3.2 `parseHeader`

直接采用`regex`字符串匹配，将陪陪的字符串保存在`header`中

```
void HttpRequest::ParseHeader_(const string& line) {
    regex patten("^([^:]*): ?(.*)$");
    smatch subMatch;
    if(regex_match(line, subMatch, patten)) {
        header_[subMatch[1]] = subMatch[2];
    }
    else {
        state_ = BODY;
    }
}
```

## 4 `httpResponse`类

成员变量如下

```c++
    int code_; // status code
    bool isKeepAlive_;

    std::string path_;
    std::string srcDir_;
    
    char* mmFile_; //
    struct stat mmFileStat_;
```

### 4.1 `init`

采用`response_.parse()`获得的`src`, `path`, `code`, `keepalive`进行初始化

```c++
void HttpResponse::Init(const string& srcDir, string& path, bool isKeepAlive, int code){
    assert(srcDir != "");
    if(mmFile_) { UnmapFile(); }
    code_ = code;
    isKeepAlive_ = isKeepAlive;
    path_ = path;
    srcDir_ = srcDir;
    mmFile_ = nullptr; 
    mmFileStat_ = { 0 };
}
```

### 4.2 `makeResponse`

```c++
void HttpResponse::MakeResponse(Buffer& buff) {
    /* 判断请求的资源文件 */
    if(stat((srcDir_ + path_).data(), &mmFileStat_) < 0 || S_ISDIR(mmFileStat_.st_mode)) {
        code_ = 404;
    }
    else if(!(mmFileStat_.st_mode & S_IROTH)) {
        code_ = 403;
    }
    else if(code_ == -1) { 
        code_ = 200; 
    }
    ErrorHtml_();
    AddStateLine_(buff);
    AddHeader_(buff);
    AddContent_(buff);
}
```

`stat`用法如下

> int stat(const char \*path, struct stat \*buf);  
> stat() function is used to list properties of a file identified by path. It reads all file properties and dumps to buf structure. The function is defined in sys/stat.h header file.  
> return 0 when success, -1 when unable to get file properties.

这里先判断是否能获得文件属性，

-   不能获得文件属性 or 文件属性为目录 -> `code_ = 404`
-   其他组读权限 -> `code_ = 403`
-   没有异常情况 -> `code_ = 200`

之后根据`code_`到`errorHtml`，在这一步中如果根据错误码把`mmFileStat_`设置为对应的界面，如果`code`正常的话，其实这步会被跳过

调用`AddStateLine_`添加状态行

调用`AddHeader_`加入消息头

调用`AddContent_`添加消息内容

## `Buffer`

`buffer`主要管理的是字符串的在并发场景下的存储和读取，从`buffer`的成员变量来看，`buffer`的组成是非常简单的

```c++
    std::vector<char> buffer_;
    std::atomic<std::size_t> readPos_;
    std::atomic<std::size_t> writePos_;
```

-   一个可扩容的`vector`
-   `atomic`类型的`readPos_`和`writePos_`组成，`readPos_`表示读操作应该开始的位置，`writePos_`表示写操作应该开始的位置，`readPos_`和`writePos_`之间的区域则是写入了但还没有读的区域。

主要方法包括：

-   `retrieve(size_t len)`: 将`readPos_`向后移动`len`
-   `retrieveAll()`: `readPos_`和`writePos`都置为0
-   `RetrieveAllToStr()`: 将`readPos_`和`writePos`读入到`string`中
-   `MakeSpace(int len)`: 判断能写入的区域是否能容纳下写入的长度，不能的话就`resize`一下`buffer`，否则把待写的内容移动到`buffer`的最前端
-   `Append(const char *str, size_t len)`: 先确保空间可用，不够就扩容，之后用`copy`函数将数据写入到`buffer`中，更新`writePos_`
-   `ReadFd(int fd, int *saveErrno)`: 使用`readv`函数从`fd`中读取内容到`buffer`中
-   `WriteFd(int fd, int *saveErrno)`: 使用`write`函数将`buffer`内容写到`fd`中