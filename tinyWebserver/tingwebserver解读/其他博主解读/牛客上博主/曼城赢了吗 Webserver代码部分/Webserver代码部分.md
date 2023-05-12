# Webserver代码部分

# 1.线程同步机制封装类locker.h

```c++
#ifndef LOCKER_H
#define LOCKER_H

#include <pthread.h>
#include <exception>
#include <semaphore.h>
// 线程同步机制封装类

// 互斥锁类
class locker
{
private:    
    pthread_mutex_t m_mutex;            // 互斥量类型
public:
    locker(){
        if(pthread_mutex_init(&m_mutex, NULL) != 0){    // 创建
            throw std::exception();
        }
    }

    ~locker(){
        pthread_mutex_destroy(&m_mutex);                // 释放
    }

    bool lock(){
        return pthread_mutex_lock(&m_mutex) == 0;       // 上锁
    }

    bool unlock(){
        return pthread_mutex_unlock(&m_mutex) == 0;     // 解锁
    }

    pthread_mutex_t* get(){ //获取
        return &m_mutex;
    }
};

// 条件变量类
class cond{
private:
    pthread_cond_t m_cond;              // 条件变量类型
public:
    cond(){
        if(pthread_cond_init(&m_cond, NULL) != 0){      // 初始化
            throw std::exception();
        }
    }

    ~cond(){
        pthread_cond_destroy(&m_cond);                  // 释放
    }

    bool wait(pthread_mutex_t* mutex){              
        return pthread_cond_wait(&m_cond, mutex) == 0;  // 等待
    }
 
    bool timedwait(pthread_mutex_t* mutex, struct timespec t){  // 在一定时间等待
        return pthread_cond_timedwait(&m_cond, mutex, &t) == 0;
    }

    bool signal(){
        return pthread_cond_signal(&m_cond) == 0;       // 唤醒一个/多个线程
    }

    bool broadcast(){
        return pthread_cond_broadcast(&m_cond) == 0;    // 唤醒所有线程
    }
};

// 信号量类
class sem
{
private:
    sem_t m_sem;
public:
    sem(){
        if(sem_init(&m_sem, 0, 0) != 0){    // 默认构造
            throw std::exception();
        }
    }

    sem(int num){           // 传参构造
        if(sem_init(&m_sem, 0, num) != 0){
            throw std::exception();
        }
    }
    
    ~sem(){                 // 释放
        sem_destroy(&m_sem);
    }

    bool wait(){            // 等待（减少）信号量
        return sem_wait(&m_sem) == 0;
    }

    bool post(){            // 增加信号量
        return sem_post(&m_sem) == 0;
    }
};

#endif
```

# 2.线性池类threadpool.h

```c++
#ifndef THREADPOOL_H
#define THREADPOOL_H

#include <pthread.h>
#include <list>
#include "locker.h"
#include <cstdio>

// 线程池类，定义成模板类，为了代码的复用，模板参数T是任务类
template<typename T>
class threadpool
{
private:
    int m_thread_num;               // 线程数量
    pthread_t * m_threads;          // 线程池数组，大小为m_thread_num，声明为指针，后面动态创建数组
    int m_max_requests;             // 请求队列中的最大等待数量
    std::list<T*> m_workqueue;      // 请求队列，由threadpool类型的示例进行管理
    locker m_queue_locker;          // 互斥锁
    sem m_queue_stat;               // 信号量
    bool m_stop;                    // 是否结束线程，线程根据该值判断是否要停止

    static void* worker(void* arg); // 静态函数，线程调用，不能访问非静态成员
    void run();                     // 线程池已启动，执行函数

public:
    threadpool(int thread_num = 8, int max_requests = 10000);
    ~threadpool();
    bool append(T* request);    // 添加任务的函数
};


template<typename T>
threadpool<T>::threadpool(int thread_num, int max_requests) :   // 构造函数，初始化
        m_thread_num(thread_num), m_max_requests(max_requests),
        m_stop(false), m_threads(NULL)
{
    if(thread_num <= 0 || max_requests <= 0){
        throw std::exception();
    }

    m_threads = new pthread_t[m_thread_num];    // 动态分配，创建线程池数组
    if(!m_threads){
        throw std::exception();
    }

    for(int i = 0; i < thread_num; ++i){
        printf("creating the N0.%d thread.\n", i);

        // 创建线程, worker（线程函数） 必须是静态的函数
        if(pthread_create(m_threads + i, NULL, worker, this) != 0){     // 通过最后一个参数向 worker 传递 this 指针，来解决静态函数无法访问非静态成员的问题
            //静态函数不依赖于任何特定的类实例，而是属于整个类本身。因此，它不能访问实例变量，因为这些变量只有在类的实例化过程中才会被创建。
            delete [] m_threads;        // 创建失败，则释放数组空间，并抛出异常
            throw std::exception();
        }

        // 设置线程分离，结束后自动释放空间
        if(pthread_detach(m_threads[i])){
            delete [] m_threads;
            throw std::exception();
        }  
    }
}

template<typename T>
threadpool<T>::~threadpool(){       // 析构函数
    delete [] m_threads;            // 释放线程数组空间
    m_stop = true;                  // 标记线程结束
}

template<typename T>
bool threadpool<T>::append(T* request){     // 添加请求队列
      m_queue_locker.lock();                // 队列为共享队列，上锁
      if(m_workqueue.size() > m_max_requests){
        m_queue_locker.unlock();            // 队列元素已满
        return false;                       // 添加失败
      }

      m_workqueue.push_back(request);       // 将任务加入队列
      m_queue_locker.unlock();              // 解锁
      m_queue_stat.post();                  // 增加信号量，线程根据信号量判断阻塞还是继续往下执行
      return true;
}

template<typename T>
void* threadpool<T>::worker(void* arg){     // arg 为线程创建时传递的threadpool类的 this 指针参数
    threadpool* pool = (threadpool*) arg;   // 参数是一个实例，创建一个实例去运行
    pool->run();    // 线程实际执行函数
    return pool;    // 无意义
}

template<typename T>
void threadpool<T>::run(){              // 线程实际执行函数
    while(!m_stop){                     // 判断停止标记
        m_queue_stat.wait();            // 等待信号量有数值（减一）
        m_queue_locker.lock();          // 上锁
        if(m_workqueue.empty()){        // 空队列
            m_queue_locker.unlock();    // 解锁
            continue;
        }

        T* request = m_workqueue.front();   // 取出任务
        m_workqueue.pop_front();            // 移出队列
        m_queue_locker.unlock();            // 解锁
        if(!request){
            continue;
        }

        request->process();             // 任务类 T 的执行函数
    }
}

#endif
```

# 3.主函数main

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <error.h>
#include <fcntl.h>      
#include <sys/epoll.h>
#include <signal.h>
#include <assert.h>
#include "locker.h"
#include "threadpool.h"
#include "http_conn.h"
#include "lst_timer.h"
#include "log.h"

#define MAX_FD 65535            // 最大文件描述符（客户端）数量
#define MAX_EVENT_SIZE 10000    // 监听的最大的事件数量

static int pipefd[2];           // 管道文件描述符 0为读，1为写
// static sort_timer_lst timer_lst;// 定时器链表

// 信号处理，添加信号捕捉
void addsig(int sig, void(handler)(int)){       
    struct sigaction sigact;                    // sig 指定信号， void handler(int) 为处理函数
    memset(&sigact, '\0', sizeof(sigact));      // bezero 清空
    sigact.sa_flags = 0;                        // 调用sa_handler
    // sigact.sa_flags |= SA_RESTART;                  // 指定收到某个信号时是否可以自动恢复函数执行，不需要中断后自己判断EINTR错误信号
    sigact.sa_handler = handler;                // 指定回调函数
    sigfillset(&sigact.sa_mask);                // 将临时阻塞信号集中的所有的标志位置为1，即都阻塞
    //sa_mask：信号屏蔽字，用于设置在处理该信号时要屏蔽的信号集。我们不希望在处理信号中断时被其他的信号打断
    sigaction(sig, &sigact, NULL);              // 设置信号捕捉sig信号值
}

// 向管道写数据的信号捕捉回调函数
void sig_to_pipe(int sig){
    int save_errno = errno;
    int msg = sig;
    send( pipefd[1], ( char* )&msg, 1, 0 );
    errno = save_errno;
    //写之前保存了errno的值，以便在写操作之后恢复errno的值，防止被信号处理函数修改。
}

// 添加文件描述符到epoll中 （声明成外部函数）
extern void addfd(int epoll_fd, int fd, bool one_shot, bool et); 

// 从epoll中删除文件描述符
extern void rmfd(int epoll_fd, int fd);

// 在epoll中修改文件描述符
extern void modfd(int epoll_fd, int fd, int ev);

// 文件描述符设置非阻塞操作
extern void set_nonblocking(int fd);

int main(int argc, char* argv[]){

    if(argc <= 1){      // 形参个数，第一个为执行命令的名称
        EMlog(LOGLEVEL_ERROR,"run as: %s port_number\n", basename(argv[0]));      // argv[0] 可能是带路径的，用basename转换
        exit(-1);
    }

    // 获取端口号
    int port = atoi(argv[1]);   // 字符串转整数

    // 对SIGPIE信号进行处理(捕捉忽略，默认退出)
    // 当往一个写端关闭的管道或socket连接中连续写入数据时会引发SIGPIPE信号,引发SIGPIPE信号的写操作将设置errno为EPIPE
    // 因为SIGPIPE信号的默认行为是结束进程，而我们绝对不希望因为写操作的错误而导致程序退出，所以我们捕捉并忽略
    addsig(SIGPIPE, SIG_IGN);           // https://blog.csdn.net/chengcheng1024/article/details/108104507
    
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);    // 监听套接字
    assert( listen_fd >= 0 );                            // ...判断是否创建成功
    //assert宏是一个调试宏，用于在程序运行时进行断言检查，如果断言条件为假（即listen_fd小于0），则会触发断言失败，导致程序终止，并在标准错误流中输出错误信息。

    // 设置端口复用
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEPORT, &reuse, sizeof(reuse));

    // 绑定
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);
    int ret = bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    assert( ret != -1 );    // ...判断是否成功

    // 监听
    ret = listen(listen_fd, 8);
    assert( ret != -1 );    // ...判断是否成功

    // 创建epoll对象，事件数组（IO多路复用，同时检测多个事件）
    epoll_event events[MAX_EVENT_SIZE]; // 结构体数组，接收检测后的数据
    int epoll_fd = epoll_create(5);     // 参数 5 无意义， > 0 即可，返回一个int类型的epoll文件描述符
    assert( epoll_fd != -1 );
    // 将监听的文件描述符添加到epoll对象中
    addfd(epoll_fd, listen_fd, false, false);  // 监听文件描述符不需要 ONESHOT & ET
    
    // 创建管道
    ret = socketpair(PF_UNIX, SOCK_STREAM, 0, pipefd);
    assert( ret != -1 );
    set_nonblocking( pipefd[1] );               // 写管道非阻塞
    addfd(epoll_fd, pipefd[0], false, false ); // epoll检测读管道

    // 设置信号处理函数
    addsig(SIGALRM, sig_to_pipe);   // 定时器信号
    addsig(SIGTERM, sig_to_pipe);   // SIGTERM 关闭服务器
    bool stop_server = false;       // 关闭服务器标志位

    // 创建一个保存所有客户端信息的数组
    http_conn* users = new http_conn[MAX_FD];
    http_conn::m_epoll_fd = epoll_fd;       // 静态成员，类共享

    // 创建线程池，初始化线程池
    threadpool<http_conn> * pool = NULL;    // 模板类 指定任务类类型为 http_conn
    try{
        pool = new threadpool<http_conn>;
    }catch(...){//...表示捕获所有类型的异常
        exit(-1);
    }

    bool timeout = false;   // 定时器周期已到
    alarm(TIMESLOT);        // 定时产生SIGALRM信号,TIMESLOT的秒数过后产生SIGALRM信号

    while(!stop_server){
        // 检测事件
        int num = epoll_wait(epoll_fd, events, MAX_EVENT_SIZE, -1);     // 阻塞，返回事件数量，第三个参数是第二个参数的size
        if(num < 0 && errno != EINTR){
            //EINTR表示在等待期间收到了中断信号，可以忽略该错误。否则，根据具体情况进行错误处理。
            EMlog(LOGLEVEL_ERROR,"EPOLL failed.\n");
            break;
        }

        // 循环遍历事件数组
        for(int i = 0; i < num; ++i){

            int sock_fd = events[i].data.fd;
            if(sock_fd == listen_fd){   // 监听文件描述符的事件响应
                // 有客户端连接进来
                struct sockaddr_in client_addr;
                socklen_t client_addr_len = sizeof(client_addr);
                int conn_fd = accept(listen_fd,(struct sockaddr*)&client_addr, &client_addr_len);
                // ...判断是否连接成功

                if(http_conn::m_user_cnt >= MAX_FD){
                    // 目前连接数满了
                    // ...给客户端写一个信息：服务器内部正忙
                    close(conn_fd);
                    continue;
                }
                // 将新客户端数据初始化，放到数组中
                users[conn_fd].init(conn_fd, client_addr);  // conn_fd 作为索引
                // 当listen_fd也注册了ONESHOT事件时(addfd)，
                // 接受了新的连接后需要重置socket上EPOLLONESHOT事件，确保下次可读时，EPOLLIN 事件被触发，因此不应该将listen注册为oneshot
                // modfd(epoll_fd, listen_fd, EPOLLIN); 
 
            }   
                // 读管道有数据，SIGALRM 或 SIGTERM信号触发
            else if(sock_fd == pipefd[0] && (events[i].events & EPOLLIN)){  
                int sig;
                char signals[1024];
                ret = recv(pipefd[0], signals, sizeof(signals), 0);
                if(ret == -1){
                    continue;
                }else if(ret == 0){
                    continue;
                }else{
                    for(int i = 0; i < ret; ++i){
                        switch (signals[i]) // 字符ASCII码
                        {
                        case SIGALRM:
                        // 用timeout变量标记有定时任务需要处理，但不立即处理定时任务
                        // 这是因为定时任务的优先级不是很高，我们优先处理其他更重要的任务。
                            timeout = true;
                            break;
                        case SIGTERM:
                            //程收到SIGTERM信号时，它会尝试优雅地终止自己的运行，也就是说，它会完成当前正在执行的任务，清理资源并退出
                            stop_server = true;
                        }
                    }
                }
            }
            else if(events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)){
                // 对方异常断开 或 错误 等事件
                EMlog(LOGLEVEL_DEBUG,"-------EPOLLRDHUP | EPOLLHUP | EPOLLERR--------\n");
                users[sock_fd].conn_close(); 
                http_conn::m_timer_lst.del_timer(users[sock_fd].timer);  // 移除其对应的定时器
            }
            else if(events[i].events & EPOLLIN){
                EMlog(LOGLEVEL_DEBUG,"-------EPOLLIN-------\n\n");
                if (users[sock_fd].read()){         // 主进程一次性读取缓冲区的所有数据
                    pool->append(users + sock_fd);  // 加入到线程池队列中，数组指针 + 偏移 &users[sock_fd]
                }else{
                    users[sock_fd].conn_close();
                    http_conn::m_timer_lst.del_timer(users[sock_fd].timer);  // 移除其对应的定时器
                }

            }
            else if(events[i].events & EPOLLOUT){
                EMlog(LOGLEVEL_DEBUG, "-------EPOLLOUT--------\n\n");
                if (!users[sock_fd].write()){       // 主进程一次性写完所有数据
                    users[sock_fd].conn_close();    // 写入失败
                    http_conn::m_timer_lst.del_timer(users[sock_fd].timer);  // 移除其对应的定时器   
                }
            }
        }
        // 最后处理定时事件，因为I/O事件有更高的优先级。当然，这样做将导致定时任务不能精准的按照预定的时间执行。
        if(timeout) {
            // 定时处理任务，实际上就是调用tick()函数
            http_conn::m_timer_lst.tick();
            // 因为一次 alarm 调用只会引起一次SIGALARM 信号，所以我们要重新定时，以不断触发 SIGALARM信号。
            alarm(TIMESLOT);
            timeout = false;    // 重置timeout
        }
    }
    close(epoll_fd);
    close(listen_fd);
    close(pipefd[1]);
    close(pipefd[0]);
    delete[] users;
    delete pool;
    return 0;
}
/*basename用于从路径名中提取文件名部分，返回字符串中最后一个斜杠字符之后的所有字符
atoi函数，将字符串转换为int
sigfillset函数用于将信号集合中所有信号都设置为“被阻塞”状态
SIG_IGN是一个指向空函数的指针，执行“什么都不做”
catch(...)表示处理任何类型的异常
extern关键字，用于声明一个变量或者函数是在其他文件中定义的

*/
```

待更新……