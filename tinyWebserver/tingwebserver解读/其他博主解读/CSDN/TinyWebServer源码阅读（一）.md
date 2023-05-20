# 主程序的步骤

在本部分，主要去解析WebServer的四个主要组成部分

-   Webserver类：供`main`函数使用的`webserver`主类，通过该类可以进行网络通信的各种连接
-   Epoll类：通过`epoll`函数进行系统调用
-   Thread类：负责线程池的管理
-   Sql资源管理部分  
    从`main.cpp`中，我们可以看到，主程序做了两件事
-   创建一个`webServer`
-   调用了`server.start()` 启动创建的`server`

## 1 WebServer类

下面进入到`WebServer`类的代码中一探究竟

### 1.1 初始化

在列表初始化中，初始化了`port`,`Timer`,`ThreadPool`,`Epoller`  
在构造函数中主要包含了以下步骤

1.  初始化`HttpConn`的静态变量，包括
    -   `userCount`: 默认为0
    -   `srcDir`: `getcwd + '/resources/'`
2.  初始化`SqlConnPool`
3.  初始化`Webserver`的`eventMode_`
4.  `InitSocket_`:
    -   主要包含初始化socket的各种配置，创建`listenFd`
5.  初始化`log`

`InitSocket_`具体包含：

> -   创建socket
> -   设置数据没发送完毕的时候容许逗留（[setsockopt用法介绍](https://www.cnblogs.com/eeexu123/p/5275783.html)）  
>     `setsockopt(listenFd_, SOL_SOCKET, SO_LINGER, &optLinger, sizeof(optLinger));`
> -   设置端口复用 `setsockopt(listenFd_, SOL_SOCKET, SO_REUSEADDR, (const void*)&optval, sizeof(int));`
> -   bind + listen
> -   添加fd到监听红黑树上
>     -   实际通过`Epoller`类进行操作，核心代码为`epoll_ctl(epollFd_, EPOLL_CTL_ADD, fd, &ev)`
>     -   修改`listenFd_`为非阻塞: `SetFdNonBlock(listenFd_)`

### 1.2 WebServer的启动

涉及到的是`WebServer Start`部分的代码，`timeMS`表示事件等待超时的时间

1.  判断`isClose_`是否为`true`，不为`true`时一直停留在循环内部（步骤2-步骤4）
2.  通过`timer`更新`timeMS`
3.  调用`int eventCnt = epoller_->Wait(timeMS);`获取满足监听事件的总数
4.  遍历获取的事件对事件进行处理
    -   如果是`listenfd_`，调用`DealListen_()`
    -   如果`EPOLLRDHUP | EPOLLHUP | EPOLLERROR`调用`CloseConn_()`
    -   如果是`EPOLLIN`，调用`DealRead_()`
    -   如果是`EPOLLOUT`，调用`DealWrite_()`
5.  跳转至步骤1

-   `DealListen_()`的处理

> -   调用`accept`
>     -   出错处理
> -   添加`client`
>     -   在`AddClient_`中执行
>     -   包括添加到`timer`，调用`epoller_->AddFd`添加到监听红黑树上，设置`nonblock`等一系列步骤

-   `CloseConn_()`的处理

> -   `epoller_->DelFd()`
> -   `client`关闭连接

-   `DealRead_()`/`DealWrite_()`的处理

> -   延长时间
> -   加入`threadPool`

## 2 Epoller类

`Epoller`非常简单，主要是对`epoll`操作的封装，内部用`vector<struct epoll_event> events`存储监听树上的事件  
主要方法如下

-   `AddFd(int fd, uint32_t event)`
-   `ModFd(int fd, uint32_t event)`
-   `DelFd(int fd)`
-   `Wait(int timeoutMs)`
-   `GetEventFd(size_t i)`
-   `GetEvents(size_t i)`  
    这里可以直接参考`epoll`的用法

## 3 Thread类

核心代码在`pool/threadPool`中，在`ThreadPool`成员变量为`Pool`

```c++
   struct Pool {
       std::mutex mtx;
       std::condition_variable cond;
       bool isClosed;
       std::queue<std::function<void()>> tasks;
    };
    std::shared_ptr<Pool> pool_;
```

对其进行分析，里面主要含有锁`mutex`，条件变量`cond`，是否关闭`isClosed`，函数组成的队列`queue<std::function<void()>> tasks`  
`pool`是多个线程之间的共享资源，因为对`pool`的所有操作都需要先获取锁

### 3.1 线程池

首先来到构造函数，它做的事是

-   初始化`pool_`: 这里采用`make_shared`进行初始化（[c11 make\_shared](https://www.jianshu.com/p/03eea8262c11)），构造函数建议使用`make_shared`而不是`new`分配对象
-   创建`threadCount`个线程：用`detach`方法作为后台线程，线程的使用可以参考[joining-and-detaching-threads](https://thispointer.com/c11-multithreading-part-2-joining-and-detaching-threads/)

创建的后台线程做的事如下：  
\- 使用`std::unique_lock`创建一个锁（[使用方法](https://blog.csdn.net/shoufei403/article/details/107510476)），该构造方法会直接对`mutex`对象加锁  
\- while循环中  
\- 当`tasks`不为空时，可以看到已经获取了`lock`，取出第一个`task`，释放`lock`,执行`task`，最后获取锁进到下一次循环中  
\- 当`tasks`为空时，如果有停止标志，退出循环，否则等待  
\- 当 `std::condition_variable` 对象的某个 wait 函数被调用的时候，它使用 `std::unique_lock`(通过 `std::mutex`) 来锁住当前线程。当前线程会一直被阻塞，直到另外一个线程在相同的 `std::condition_variable` 对象上调用了 `notification` 函数来唤醒当前线程  
\- 观察到在`addTask`方法中， `pool_->cond.notify_one()`会将其唤醒

```c++
   explicit ThreadPool(size_t threadCount = 8): pool_(std::make_shared<Pool>()) {
           assert(threadCount > 0);
           for(size_t i = 0; i < threadCount; i++) {
               std::thread([pool = pool_] {
                   std::unique_lock<std::mutex> locker(pool->mtx);
                   while(true) {
                       if(!pool->tasks.empty()) {
                           auto task = std::move(pool->tasks.front());
                           pool->tasks.pop();
                           locker.unlock();
                           task();
                           locker.lock();
                       } 
                       else if(pool->isClosed) break;
                       else pool->cond.wait(locker);
                   }
               }).detach();
           }
   }
```

### 3.2 回调函数的放入

`std::function<void()>`是可调用对象包装器，可用于`callback`，参考[why use std::function](https://stackoverflow.com/questions/11352936/why-do-we-use-stdfunction-in-c-rather-than-the-original-c-function-pointer), [std::function and std::bind](https://stackoverflow.com/questions/55124517/stdfunction-and-stdbind-return-value), [std::bind](https://blog.csdn.net/Jxianxu/article/details/107382049)

整个放入到tasks到流程如下

```c++
threadpool_->AddTask(std::bind(&WebServer::OnWrite_, this, client))
void AddTask(F&& task) {
    {
        std::lock_guard<std::mutex> locker(pool_->mtx);
        pool_->tasks.emplace(std::forward<F>(task));
    }
    pool_->cond.notify_one();
}
std::queue<std::function<void()>> tasks;
```

`std::bind`将`client`参数绑定到`WebServer::OnWrite_`方法，这样返回的`functor`就变为`void()`，放入到`std::function<void()>`可调用对象包装器中。

### 3.3 析构函数

在析构函数中，需要唤醒阻塞的后台进程，使其结束。

```c++
~ThreadPool() {
    if(static_cast<bool>(pool_)) {
        {
            std::lock_guard<std::mutex> locker(pool_->mtx);
            pool_->isClosed = true;
        }
        pool_->cond.notify_all();
    }
}
```

## 4 Sql资源管理

主要包含以下文件：`sqlConnRAII.h`, `sqlconnpool.h`, `sqlconnpool.c`

### 4.1 RAII机制

RAII（Resource Acquisition Is Initialization）为资源获取即初始化，`sqlConnRAII.h`专门负责了sql资源的连接，关键代码为在构造函数中调用`*sql = connpool->GetConn();`，在析构函数中调用`connpool_->FreeConn(sql_);`  
内部成员有两个：

-   `MYSQL *sql_;`
-   `SqlConnPool* connpool_;`

### 4.2 SqlConnPool

`sqlconnpool.h`中，静态方法为`instance`: 它用static声明了一个`SqlConnPool`对象并返回了一个地址  
成员方法包括

-   `MYSQL *GetConn()`
-   `void FreeConn(MYSQL conn)`
-   `int GetFreeConnCount()`
-   `void Init(args)`
-   `void ClosePool()`  
    成员变量包括
-   `std::queue<MYSQL *> connQue_`;
-   `std::mutex mtx_`;
-   `sem_t semId_`;
-   `int MAX_CONN_`;

#### 4.2.1 Init和ClosePool

首先来到`init`方法，它调用`connSize`次  
\- `mysql_init`  
\- `mysql_real_connect`  
\- 并`MYSQL`对象添加到`connQue_`中  
之后初始化`MAC_CONN_`和初始化信号量`semId_`为`MAX_CONN_`  
在`ClosePool`中，主要做的事情是清空`connQue_`,调用`mysql_library_end`结束`mysql`

#### 4.2.2 GetConn和FreeConn

这里采用sem和[lock\_guard](https://blog.csdn.net/fengbingchun/article/details/78649260)对多线程并发访问进行控制，`sem_wait`和`sem_post`对连接数进行原子操作，`lock_guard`在构造时会自动获取锁，析构时会释放锁，用来取出`sql`对象

```c++
MYSQL* SqlConnPool::GetConn() {
    MYSQL *sql = nullptr;
    if(connQue_.empty()){
        LOG_WARN("SqlConnPool busy!");
        return nullptr;
    }
    sem_wait(&semId_);
    {
        lock_guard<mutex> locker(mtx_);
        sql = connQue_.front();
        connQue_.pop();
    }
    return sql;
}
void SqlConnPool::FreeConn(MYSQL* sql) {
    assert(sql);
    lock_guard<mutex> locker(mtx_);
    connQue_.push(sql);
    sem_post(&semId_);
}
```

## 参考资料

1.  c11语法: [Cplusplus-A-Glance-Part-of-n](https://www.codeproject.com/Articles/312029/Cplusplus-A-Glance-Part-of-n)
2.  c++多线程编程：[C++11 并发指南](https://www.cnblogs.com/haippy/p/3235560.html)
3.  writev和readv：[高级I/O之readv和writev函数](https://blog.csdn.net/weixin_36750623/article/details/84579243)