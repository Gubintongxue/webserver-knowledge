[(46条消息) 【面试专栏】自己整理的WebServer项目问题_温酒煮青梅的博客-CSDN博客](https://blog.csdn.net/weixin_44484715/article/details/120825122)



1.怎样应对服务器的大流量、高并发
客户端：
尽量减少请求数量：依靠客户端自身的缓存或处理能力
尽量减少对服务端资源的不必要耗费：重复使用某些资源，如连接池
服务端：
增加资源供给：更大的网络带宽，使用更高配置的服务器
请求分流：使用集群,分布式的系统架构
应用优化：使用更高效的编程语言,优化处理业务逻辑的算法
2.线程池与多线程的设计思路
设计一个任务队列，作为临界资源
初始化n个线程，开始运行，对任务队列加锁取拿取任务执行
当任务队列为空时，所有子线程（工作线程）阻塞（pthread_cond_wait）
主线程感知到内核事件表上有事件发生时，将任务添加到任务队列中，唤醒睡眠的工作线程（pthread_cond_signal pthread_cond_broadcast）
3、客户端断开连接，服务端epoll监听到的事件是什么
在使用 epoll 时，客户端正常断开连接（调用 close()），在服务器端会触发一个 epoll 事件。在早期的内核中，这个 epoll 事件一般是 EPOLLIN，即 0x1，代表连接可读。

连接池检测到某个连接发生 EPOLLIN 事件且没有错误后，会认为有请求到来，将连接交给上层进行处理。这样一来，上层尝试在对端已经 close() 的连接上读取请求，只能读到 EOF（文件末尾），会认为发生异常，报告一个错误。

后期的内核中增加了 EPOLLRDHUP 事件，代表对端断开连接。对端连接断开触发的 epoll 事件会包含 EPOLLIN | EPOLLRDHUP，即 0x2001。有了这个事件，对端断开连接的异常就可以在底层进行处理了，不用再移交到上层

4、epoll的线程安全
简要结论就是epoll是通过锁来保证线程安全的, epoll中粒度最小的自旋锁ep->lock(spinlock)用来保护就绪的队列, 互斥锁ep->mtx用来保护epoll的重要数据结构红黑树

主要的两个函数：

epoll_ctl()：当需要根据不同的operation通过ep_insert() 或者ep_remove()等接口对epoll自身的数据结构进行操作时都提前获得了ep->mtx锁
epll_wait()：获得自旋锁 ep->lock来保护就绪队列
5、SO_REUSEDADDR和SO_REUSEDPORT
在TCP连接下，如果服务器主动关闭连接（比如ctrl+c结束服务器进程），那么由于服务器这边会出现time_wait状态，所以不能立即重新启动服务器进程。在标准文档中，2MSL时间为两分钟。

一个端口释放后会等待两分钟之后才能再被使用，SO_REUSEADDR是让端口释放后立即就可以被再次使用。
如果不进行端口重用的话，客户端可能不受什么影响，因为在客户端主动关闭后，客户端可以使用另一个端口与服务端再次建立连接；但是服务端主动关闭连接后，其周知端口在两分钟内不能再次使用，就很麻烦

6、HTTP请求怎么拆包
HTTP请求内容：请求行，请求头，空行，请求体

一个有报文的请求到服务器时，请求头里都会有content_length，这个指定了报文的大小。报文如果很大的时候，会通过一部分一部分的发送请求，直到结束。当这个过程中，出现多个请求，第一个请求会带有请求头信息，前面一个请求的发送的报文如果没有满时，会把后面一个请求的内容填上，这个操作就叫粘包。这样粘包后，它会通过content_length字段的大小，来做拆包。

7、VS怎么debug多线程
https://www.cnblogs.com/thaughtZhao/p/4277941.html

7-2、Clion怎么调试多线程
https://blog.csdn.net/m0_38129920/article/details/87454405

8、HTTP请求，判断get提交还是post提交
浏览器判断方法：用户名，密码输入内容123，点击提示按钮，观察上面提示栏的信息（username和password的值）：

1、如果是这样：

 username和password均有输入值，则是get提交；

2、如果是这样

 username和password均无输入值，则是post提交；但下面的详细信息中必有username和password输入值信息；

服务器判断：解析出状态行，查看请求方法

9、HTTP怎么接收图片和视频流
使用Content-Type字段，说明响应体中的媒体数据类型

10、pthread_XX这些和C++的thread库，底层实现的区别
https://segmentfault.com/a/1190000002655852

11、水平触发和边沿触发的优缺点、使用场景
https://blog.csdn.net/weixin_49199646/article/details/112298712

12、怎样正确关闭线程池
https://www.jianshu.com/p/4aee873d607a

13、大文件传输的方案
https://blog.csdn.net/luguifang2011/article/details/26134151?ops_request_misc=&request_id=&biz_id=&utm_medium=distribute.pc_search_result.none-task-blog-2alles_rank~default-9-26134151.pc_search_all_es&utm_term=%E5%A4%A7%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E7%9A%84%E6%8A%80%E5%B7%A7&spm=1018.2226.3001.4187

socket本身缓冲区的限制，大概一次只能发送4K左右的数据，所以在传输大数据时客户端就需要进行分包，在目的地重新组包
已有一些消息/通讯中间件对此进行了封装，提供了直接发送大数据/文件的接口
利用共享目录，ftp，ssh等系统命令
14.大数相加（包含负数）
https://www.jianshu.com/p/2ffb8cfdc97b
————————————————
版权声明：本文为CSDN博主「温酒煮青梅」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/weixin_44484715/article/details/120825122