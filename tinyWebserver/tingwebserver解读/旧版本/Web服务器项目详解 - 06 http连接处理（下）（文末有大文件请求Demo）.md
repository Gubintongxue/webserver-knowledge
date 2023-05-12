### Web服务器详解目录

- [00 项目概述](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649273988&idx=1&sn=cc169061e6bb3ca7d14ea138c7706c14&chksm=83ffbfdcb48836cab6833dadd3d4d2ee3b7346bdab791f5c9658714d1907b7104f3ed1ad7883&scene=21#wechat_redirect)
- [01 线程同步机制包装类](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649273996&idx=2&sn=f7de38b997c2217998be508525bc3836&chksm=83ffbfd4b48836c217e4dafc6d77657ea3e55589efe0ec86927fda2d392cc08af1d52e007378&scene=21#wechat_redirect)
- [02 半同步/半反应堆线程池（上）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274118&idx=1&sn=f2f889b83c760f18be0eedbd6a1ebe8b&chksm=83ffbe5eb4883748001b78c47e183900730570edcd07e3279b8d6857f0a2ce568494cbba250f&scene=21#wechat_redirect)
- [03 半同步/半反应堆线程池（下）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274139&idx=1&sn=e4a0c8d35927cc24253e840d22994e3f&chksm=83ffbe43b48837552265d980a59d7fa288c50d56c809d12f132cfefa106e33d90a3e0dd4b315&scene=21#wechat_redirect)
- [04 http连接处理（上）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274161&idx=1&sn=88c8699268f72686b7bb2cfe6dd65bba&chksm=83ffbe69b488377f7cc20cc9a4eafe64d482847365502c16fc57577f93ed5e035fda5e8d4de1&scene=21#wechat_redirect)
- [05 http连接处理（中）](http://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274185&idx=1&sn=4d464f5cd7dc64768e603e874b937446&chksm=83ffbe11b4883707e2f61bec3d0984003565c27f6887342bb667763c771cf42207e11443dbae&scene=21#wechat_redirect)
- **06 http连接处理（下）**
- 07 定时器处理非活动连接（上）
- 08 定时器处理非活动连接（下）
- 09 日志系统（上）
- 10 日志系统（下）
- 11 数据连接池
- 12 注册和登录校验
- 13 服务器测试
- 14 项目遇到的问题及解决方案
- 15 项目涉及的常见面试题

### 本文内容

上一篇详解中，我们对状态机和服务器解析请求报文进行了介绍。

本篇，我们将介绍服务器如何响应请求报文，并将该报文发送给浏览器端。

首先介绍一些基础API，然后结合流程图和代码对服务器响应请求报文进行详解。

**基础API部分**，介绍`stat`、`mmap`、`iovec`、`writev`。

**流程图部分**，描述服务器端响应请求报文的逻辑，各模块间的关系。

**代码部分**，结合代码对服务器响应请求报文进行详解。

### 基础API

为了更好的源码阅读体验，这里提前对代码中使用的一些API进行简要介绍，更丰富的用法可以自行查阅资料。

#### **stat**

stat函数用于取得指定文件的文件属性，并将文件属性存储在结构体stat里，这里仅对其中用到的成员进行介绍。

```
 1#include <sys/types.h>
 2#include <sys/stat.h>
 3#include <unistd.h>
 4
 5//获取文件属性，存储在statbuf中
 6int stat(const char *pathname, struct stat *statbuf);
 7
 8struct stat 
 9{
10   mode_t    st_mode;        /* 文件类型和权限 */
11   off_t     st_size;        /* 文件大小，字节数*/
12};
```

#### **mmap**

用于将一个文件或其他对象映射到内存，提高文件的访问速度。

```
1void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);
2int munmap(void* start,size_t length);
```



- start：映射区的开始地址，设置为0时表示由系统决定映射区的起始地址

- length：映射区的长度

- prot：期望的内存保护标志，不能与文件的打开模式冲突

- - PROT_READ 表示页内容可以被读取

- flags：指定映射对象的类型，映射选项和映射页是否可以共享

- - MAP_PRIVATE 建立一个写入时拷贝的私有映射，内存区域的写入不会影响到原文件

- fd：有效的文件描述符，一般是由open()函数返回

- off_toffset：被映射对象内容的起点

#### **iovec**

定义了一个向量元素，通常，这个结构用作一个多元素的数组。

```
1struct iovec {
2    void      *iov_base;      /* starting address of buffer */
3    size_t    iov_len;        /* size of buffer */
4};
```



- iov_base指向数据的地址
- iov_len表示数据的长度

#### **writev**

writev函数用于在一次函数调用中写多个非连续缓冲区，有时也将这该函数称为聚集写。

```
1#include <sys/uio.h>
2ssize_t writev(int filedes, const struct iovec *iov, int iovcnt);
```



- filedes表示文件描述符
- iov为前述io向量机制结构体iovec
- iovcnt为结构体的个数

若成功则返回已写的字节数，若出错则返回-1。`writev`以顺序`iov[0]`，`iov[1]`至`iov[iovcnt-1]`从缓冲区中聚集输出数据。`writev`返回输出的字节总数，通常，它应等于所有缓冲区长度之和。

### 流程图

浏览器端发出HTTP请求报文，服务器端接收该报文并调用`process_read`对其进行解析，根据解析结果`HTTP_CODE`，进入相应的逻辑和模块。

其中，服务器子线程完成报文的解析与响应；主线程监测读写事件，调用`read_once`和`http_conn::write`完成数据的读取与发送。

![图片](markdown-image/Web服务器项目详解 - 06 http连接处理（下）（文末有大文件请求Demo）.assets/640.jpeg)

#### **HTTP_CODE含义**

表示HTTP请求的处理结果，在头文件中初始化了八种情形，在报文解析与响应中只用到了七种。

- NO_REQUEST

- - 请求不完整，需要继续读取请求报文数据
  - 跳转主线程继续监测读事件

- GET_REQUEST

- - 获得了完整的HTTP请求
  - 调用do_request完成请求资源映射

- NO_RESOURCE

- - 请求资源不存在
  - 跳转process_write完成响应报文

- BAD_REQUEST

- - HTTP请求报文有语法错误或请求资源为目录
  - 跳转process_write完成响应报文

- FORBIDDEN_REQUEST

- - 请求资源禁止访问，没有读取权限
  - 跳转process_write完成响应报文

- FILE_REQUEST

- - 请求资源可以正常访问
  - 跳转process_write完成响应报文

- INTERNAL_ERROR

- - 服务器内部错误，该结果在主状态机逻辑switch的default下，一般不会触发

### 代码分析

#### **do_request**

`process_read`函数的返回值是对请求的文件分析后的结果，一部分是语法错误导致的`BAD_REQUEST`，一部分是`do_request`的返回结果.该函数将网站根目录和`url`文件拼接，然后通过stat判断该文件属性。另外，为了提高访问速度，通过mmap进行映射，将普通文件映射到内存逻辑地址。

为了更好的理解请求资源的访问流程，这里对各种各页面跳转机制进行简要介绍。其中，浏览器网址栏中的字符，即`url`，可以将其抽象成`ip:port/xxx`，`xxx`通过`html`文件的`action`属性进行设置。

m_url为请求报文中解析出的请求资源，以/开头，也就是`/xxx`，项目中解析后的m_url有5种情况。

- /

- - GET请求，跳转到judge.html，即欢迎访问界面
  - action属性设置为0和1

- /0

- - GET请求，跳转到register.html，即注册界面
  - action属性设置为3check.cgi

- /1

- - GET请求，跳转到log.html，即登录界面
  - action属性设置为2check.cgi

- /2check.cgi

- - POST请求，进行登录校验
  - 验证成功跳转到welcome.html，即资源请求成功界面
  - 验证失败跳转到logError.html，即登录失败界面

- /3check.cgi

- - POST请求，进行注册校验
  - 注册成功跳转到log.html，即登录界面
  - 注册失败跳转到registerError.html，即注册失败界面

- /test.jpg

- - GET请求，请求服务器上的图片资源

如果大家对上述设置方式不理解，不用担心。具体的登录和注册校验功能会在第12节进行详解，到时候还会针对html进行介绍。

```
 1//网站根目录，文件夹内存放请求的资源和跳转的html文件
 2const char* doc_root="/home/qgy/github/ini_tinywebserver/root";
 3
 4http_conn::HTTP_CODE http_conn::do_request()
 5{
 6    //将初始化的m_real_file赋值为网站根目录
 7    strcpy(m_real_file,doc_root);
 8    int len=strlen(doc_root);
 9
10    //找到m_url中/的位置
11    const char *p = strrchr(m_url, '/'); 
12
13    //实现登录和注册校验
14    if(cgi==1 && (*(p+1) == '2' || *(p+1) == '3'))
15    {
16        //根据标志判断是登录检测还是注册检测
17
18        //同步线程登录校验
19
20        //CGI多进程登录校验
21    }
22
23    //如果请求资源为/0，表示跳转注册界面
24    if(*(p+1) == '0'){
25        char *m_url_real = (char *)malloc(sizeof(char) * 200);
26        strcpy(m_url_real,"/register.html");
27
28        //将网站目录和/register.html进行拼接，更新到m_real_file中
29        strncpy(m_real_file+len,m_url_real,strlen(m_url_real));
30
31        free(m_url_real);
32    }
33    //如果请求资源为/1，表示跳转登录界面
34    else if( *(p+1) == '1'){
35        char *m_url_real = (char *)malloc(sizeof(char) * 200);
36        strcpy(m_url_real,"/log.html");
37
38        //将网站目录和/log.html进行拼接，更新到m_real_file中
39        strncpy(m_real_file+len,m_url_real,strlen(m_url_real));
40
41        free(m_url_real);
42    }
43    else
44        //如果以上均不符合，即不是登录和注册，直接将url与网站目录拼接
45        //这里的情况是welcome界面，请求服务器上的一个图片
46        strncpy(m_real_file+len,m_url,FILENAME_LEN-len-1);
47
48    //通过stat获取请求资源文件信息，成功则将信息更新到m_file_stat结构体
49    //失败返回NO_RESOURCE状态，表示资源不存在
50    if(stat(m_real_file,&m_file_stat)<0)
51        return NO_RESOURCE;
52
53    //判断文件的权限，是否可读，不可读则返回FORBIDDEN_REQUEST状态
54    if(!(m_file_stat.st_mode&S_IROTH))
55        return FORBIDDEN_REQUEST;
56    //判断文件类型，如果是目录，则返回BAD_REQUEST，表示请求报文有误
57    if(S_ISDIR(m_file_stat.st_mode))
58        return BAD_REQUEST;
59
60    //以只读方式获取文件描述符，通过mmap将该文件映射到内存中
61    int fd=open(m_real_file,O_RDONLY);
62    m_file_address=(char*)mmap(0,m_file_stat.st_size,PROT_READ,MAP_PRIVATE,fd,0);
63
64    //避免文件描述符的浪费和占用
65    close(fd);
66
67    //表示请求文件存在，且可以访问
68    return FILE_REQUEST;
69}
```

#### **process_write**

根据`do_request`的返回状态，服务器子线程调用`process_write`向`m_write_buf`中写入响应报文。

- add_status_line函数，添加状态行：http/1.1 状态码 状态消息

- add_headers函数添加消息报头，内部调用add_content_length和add_linger函数

- - content-length记录响应报文长度，用于浏览器端判断服务器是否发送完数据
  - connection记录连接状态，用于告诉浏览器端保持长连接

- add_blank_line添加空行

上述涉及的5个函数，均是内部调用`add_response`函数更新`m_write_idx`指针和缓冲区`m_write_buf`中的内容。

```
 1bool http_conn::add_response(const char* format,...)
 2{
 3    //如果写入内容超出m_write_buf大小则报错
 4    if(m_write_idx>=WRITE_BUFFER_SIZE)
 5        return false;
 6
 7    //定义可变参数列表
 8    va_list arg_list;
 9
10    //将变量arg_list初始化为传入参数
11    va_start(arg_list,format);
12
13    //将数据format从可变参数列表写入缓冲区写，返回写入数据的长度
14    int len=vsnprintf(m_write_buf+m_write_idx,WRITE_BUFFER_SIZE-1-m_write_idx,format,arg_list);
15
16    //如果写入的数据长度超过缓冲区剩余空间，则报错
17    if(len>=(WRITE_BUFFER_SIZE-1-m_write_idx)){
18        va_end(arg_list);
19        return false;
20    }
21
22    //更新m_write_idx位置
23    m_write_idx+=len;
24    //清空可变参列表
25    va_end(arg_list);
26
27    return true;
28}
29
30//添加状态行
31bool http_conn::add_status_line(int status,const char* title)
32{
33    return add_response("%s %d %s\r\n","HTTP/1.1",status,title);
34}
35
36//添加消息报头，具体的添加文本长度、连接状态和空行
37bool http_conn::add_headers(int content_len)
38{
39    add_content_length(content_len);
40    add_linger();
41    add_blank_line();
42}
43
44//添加Content-Length，表示响应报文的长度
45bool http_conn::add_content_length(int content_len)
46{
47    return add_response("Content-Length:%d\r\n",content_len);
48}
49
50//添加文本类型，这里是html
51bool http_conn::add_content_type()
52{
53    return add_response("Content-Type:%s\r\n","text/html");
54}
55
56//添加连接状态，通知浏览器端是保持连接还是关闭
57bool http_conn::add_linger()
58{
59    return add_response("Connection:%s\r\n",(m_linger==true)?"keep-alive":"close");
60}
61//添加空行
62bool http_conn::add_blank_line()
63{
64    return add_response("%s","\r\n");
65}
66
67//添加文本content
68bool http_conn::add_content(const char* content)
69{
70    return add_response("%s",content);
71}
```

响应报文分为两种，一种是请求文件的存在，通过`io`向量机制`iovec`，声明两个`iovec`，第一个指向`m_write_buf`，第二个指向`mmap`的地址`m_file_address`；一种是请求出错，这时候只申请一个`iovec`，指向`m_write_buf`。

- iovec是一个结构体，里面有两个元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是writev将要发送的数据。
- 成员iov_len表示实际写入的长度



```
 1bool http_conn::process_write(HTTP_CODE ret)
 2{
 3    switch(ret)
 4    {
 5        //内部错误，500
 6        case INTERNAL_ERROR:
 7        {
 8            //状态行
 9            add_status_line(500,error_500_title);
10            //消息报头
11            add_headers(strlen(error_500_form));
12            if(!add_content(error_500_form))
13                return false;
14            break;
15        }
16        //报文语法有误，404
17        case BAD_REQUEST:
18        {
19            add_status_line(404,error_404_title);
20            add_headers(strlen(error_404_form));
21            if(!add_content(error_404_form))
22                return false;
23            break;
24        }
25        //资源没有访问权限，403
26        case FORBIDDEN_REQUEST:
27        {
28            add_status_line(403,error_403_title);
29            add_headers(strlen(error_403_form));
30            if(!add_content(error_403_form))
31                return false;
32            break;
33        }
34        //文件存在，200
35        case FILE_REQUEST:
36        {
37            add_status_line(200,ok_200_title);
38            //如果请求的资源存在
39            if(m_file_stat.st_size!=0)
40            {
41                add_headers(m_file_stat.st_size);
42                //第一个iovec指针指向响应报文缓冲区，长度指向m_write_idx
43                m_iv[0].iov_base=m_write_buf;
44                m_iv[0].iov_len=m_write_idx;
45                //第二个iovec指针指向mmap返回的文件指针，长度指向文件大小
46                m_iv[1].iov_base=m_file_address;
47                m_iv[1].iov_len=m_file_stat.st_size;
48                m_iv_count=2;
49                return true;
50            }
51            else
52            {
53                //如果请求的资源大小为0，则返回空白html文件
54                const char* ok_string="<html><body></body></html>";
55                add_headers(strlen(ok_string));
56                if(!add_content(ok_string))
57                    return false;
58            }
59        }
60        default:
61            return false;
62    }
63    //除FILE_REQUEST状态外，其余状态只申请一个iovec，指向响应报文缓冲区
64    m_iv[0].iov_base=m_write_buf;
65    m_iv[0].iov_len=m_write_idx;
66    m_iv_count=1;
67    return true;
68}
```

#### **http_conn::write**

服务器子线程调用`process_write`完成响应报文，随后注册`epollout`事件。服务器主线程检测写事件，并调用`http_conn::write`函数将响应报文发送给浏览器端。

该函数具体逻辑如下：

首先初始化剩余发送数据和已发送数据大小，byte_to_send和byte_have_send，然后通过writev函数循环发送响应报文数据，并判断响应报文整体是否发送成功。

- 若writev单次发送成功，更新byte_to_send和byte_have_send的大小，若响应报文整体发送成功,则取消mmap映射,并判断是否是长连接.

- - 长连接重置http类实例，注册读事件，不关闭连接，
  - 短连接直接关闭连接

- 若writev单次发送不成功，判断是否是写缓冲区满了。

- - 若不是因为缓冲区满了而失败，取消mmap映射，关闭连接
  - 若eagain则满了，注册写事件，等待下一次写事件触发（当写缓冲区从不可写变为可写，触发epollout），因此在此期间无法立即接收到同一用户的下一请求，但可以保证连接的完整性。



```
 1bool http_conn::write()
 2{
 3    int temp=0;
 4    int bytes_have_send=0;
 5
 6    //这里不科学 
 7    //将要发送的数据长度初始化为响应报文缓冲区长度
 8    int bytes_to_send=m_write_idx;
 9
10    //若要发送的数据长度为0
11    //表示响应报文为空，一般不会出现这种情况
12    if(bytes_to_send==0)
13    {
14        modfd(m_epollfd,m_sockfd,EPOLLIN);
15        init();
16        return true;
17    }
18
19    //这里的循环非常鸡肋，下面会详述
20    while(1)
21    {
22        //将响应报文的状态行、消息头、空行和响应正文发送给浏览器端
23        temp=writev(m_sockfd,m_iv,m_iv_count);
24        //成功发送，则返回temp字节数，发送失败需要判断是缓冲区满了，还是发送错误
25        if(temp<=-1)
26        {
27            //判断缓冲区是否满了
28            if(errno==EAGAIN)
29            {
30                //重新注册写事件
31                modfd(m_epollfd,m_sockfd,EPOLLOUT);
32                return true;
33            }
34            //如果发送失败，但不是缓冲区问题，取消映射
35            unmap();
36            return false;
37        }
38        //更新剩余数据和已发送数据
39        bytes_to_send-=temp;
40        bytes_have_send+=temp;
41
42        //你是不是很疑惑这里？？下面会详细解释这里      
43        //剩余数据和已发送数据比较，判断是否发送完。
44        if(bytes_to_send<=bytes_have_send)
45        {
46            unmap();
47            //浏览器的请求为长连接
48            if(m_linger)
49            {
50                //重新初始化HTTP对象
51                init();
52                //在epoll树上重新注册读事件
53                modfd(m_epollfd,m_sockfd,EPOLLIN);
54                return true;
55            }
56            else
57            {
58                //若为短连接，主线程中从epoll树上删除该描述符
59                modfd(m_epollfd,m_sockfd,EPOLLIN);
60                return false;
61            }
62        }
63    }
64}
```

有不少小伙伴后台私信问我，对判断发送完成的条件`bytes_to_send<=bytes_have_send`，表示疑惑，不清楚为什么这样写，并私信我讨论。

其实，这个写法是原作者的代码，我在学习时，也对这里有疑问。后面测试的时候发现没有报错，就将其搁置，没有继续深究。

那么，**为什么没有报错呢？**

由于我在项目中仅向服务器端请求了一个图片文件，该文件较小，所以调用一次`writev`函数就将数据全部发送了出去， 发送数据变量`bytes_to_send`更新为0，相对应的已发送变量`bytes_have_send`更新为发送的全部字节数。所以，凑巧符合该判断条件。

昨晚与一位小伙伴讨论后，发现这部分代码还有几处不严谨的地方，这里可以作为参考，并未同步到Github。

- bytes_to_send初始化有误，m_write_idx只表示响应报文的状态行、消息报头和空行，修改为缓冲区和请求资源的长度之和
- bytes_to_send<=bytes_have_send有误，修改为bytes_to_send<=0



```
1//将bytes_to_send初始化为服务器端发送的全部数据
2int bytes_to_send=0;
3for(int i = 0; i < m_iv_count; ++i){
4    bytes_to_send += int(m_iv[i].iov_len);
5}
6
7//将判断条件进行更新
8if(bytes_to_send<=0){
9}
```

将上述不严谨之处修改后，接下来就是最鸡肋的`while`循环。

当请求小文件，也就是调用一次`writev`函数就可以将数据全部发送出去的时候，不会报错，此时不会再次进入`while`循环。

一旦请求服务器文件较大文件时，需要多次调用`writev`函数，便会出现问题，不是文件显示不全，就是无法显示。

今天早上，分析定位到了问题，并初步解决，实现了`Demo`。等代码测试完成后，我会将其同步到`Github`。

### 大文件请求Demo

对数据传输过程分析后，定位到`writev`的`m_iv`结构体成员有问题，每次传输后不会自动偏移文件指针和传输长度，还会按照原有指针和原有长度发送数据。

根据前面的基础`API`分析，我们知道`writev`以顺序`iov[0]`，`iov[1]`至`iov[iovcnt-1]`从缓冲区中聚集输出数据。项目中，申请了2个`iov`，其中`iov[0]`为存储报文状态行的缓冲区，`iov[1]`指向资源文件指针。

初期解决思路如下：

- 由于报文消息报头较小，第一次传输后，需要更新m_iv[1].iov_base和iov_len，m_iv[0].iov_len置成0，只传输文件，不用传输响应消息头
- 每次传输后都要更新下次传输的文件起始位置和长度

对该Bug初步修正后，我在Ubuntu下的Chrome浏览器进行了大文件测试，请求服务器上大图(6M)和视频(493k)，效果如下。

- 视频测试

![图片](markdown-image/Web服务器项目详解 - 06 http连接处理（下）（文末有大文件请求Demo）.assets/640-16799865047571.gif)

- 大图测试

![图片](markdown-image/Web服务器项目详解 - 06 http连接处理（下）（文末有大文件请求Demo）.assets/640-16799865047572.gif)

后续，我会写一篇推文，对大文件传输Bug定位、解决思路的代码实现进行介绍。

完。