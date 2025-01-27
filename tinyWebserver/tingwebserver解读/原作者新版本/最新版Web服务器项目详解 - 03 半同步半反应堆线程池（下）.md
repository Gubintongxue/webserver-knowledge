### 基础知识

#### **静态成员变量**

将类成员变量声明为static，则为静态成员变量，与一般的成员变量不同，无论建立多少对象，都只有一个静态成员变量的拷贝，静态成员变量属于一个类，所有对象共享。

静态变量在编译阶段就分配了空间，对象还没创建时就已经分配了空间，放到全局静态区。

- 静态成员变量

- - 最好是类内声明，类外初始化（以免类名访问静态成员访问不到）。
  - 无论公有，私有，静态成员都可以在类外定义，但私有成员仍有访问权限。
  - 非静态成员类外不能初始化。
  - 静态成员数据是共享的。

#### **静态成员函数**

将类成员函数声明为static，则为静态成员函数。

- 静态成员函数

- - 静态成员函数可以直接访问静态成员变量，不能直接访问普通成员变量，但可以通过参数传递的方式访问。
  - 普通成员函数可以访问普通成员变量，也可以访问静态成员变量。
  - 静态成员函数没有this指针。非静态数据成员为对象单独维护，但静态成员函数为共享函数，无法区分是哪个对象，因此不能直接访问普通变量成员，也没有this指针。

#### **pthread_create陷阱**

首先看一下该函数的函数原型。

```cpp
1#include <pthread.h>
2int pthread_create (pthread_t *thread_tid,                 //返回新生成的线程的id
3                    const pthread_attr_t *attr,         //指向线程属性的指针,通常设置为NULL
4                    void * (*start_routine) (void *),   //处理线程函数的地址
5                    void *arg);                         //start_routine()中的参数
```

函数原型中的第三个参数，为函数指针，指向处理线程函数的地址。该函数，要求为静态函数。如果处理线程函数为类成员函数时，需要将其设置为**静态成员函数**。

#### **this指针的锅**

pthread_create的函数原型中第三个参数的类型为函数指针，指向的线程处理函数参数类型为`(void *)`,若线程函数为类成员函数，则this指针会作为默认的参数被传进函数中，从而和线程函数参数`(void*)`不能匹配，不能通过编译。

静态成员函数就没有这个问题，里面没有this指针。

### 线程池分析

线程池的设计模式为半同步/半反应堆，其中反应堆具体为Proactor事件处理模式。

具体的，主线程为异步线程，负责监听文件描述符，接收socket新连接，若当前监听的socket发生了读写事件，然后将任务插入到请求队列。工作线程从请求队列中取出任务，完成读写数据的处理。

#### **线程池类定义**

具体定义可以看代码。需要注意，线程处理函数和运行函数设置为私有属性。

```cpp
 template<typename T>
 class threadpool{
     public:
         //thread_number是线程池中线程的数量
         //max_requests是请求队列中最多允许的、等待处理的请求的数量
         //connPool是数据库连接池指针
         threadpool(connection_pool *connPool, int thread_number = 8, int max_request = 10000);
         ~threadpool();
 
        //像请求队列中插入任务请求
        bool append(T* request);

    private:
        //工作线程运行的函数
        //它不断从工作队列中取出任务并执行之
        static void *worker(void *arg);

        void run();

    private:
        //线程池中的线程数
        int m_thread_number;

        //请求队列中允许的最大请求数
        int m_max_requests;

        //描述线程池的数组，其大小为m_thread_number
        pthread_t *m_threads;

        //请求队列
        std::list<T *>m_workqueue;    

        //保护请求队列的互斥锁    
        locker m_queuelocker;

        //是否有任务需要处理
        sem m_queuestat;

        //是否结束线程
        bool m_stop;

        //数据库连接池
        connection_pool *m_connPool;  
};
```

#### **线程池创建与回收**

构造函数中创建线程池,pthread_create函数中将类的对象作为参数传递给静态函数(worker),在静态函数中引用这个对象,并调用其动态方法(run)。

具体的，类对象传递时用this指针，传递给静态函数后，将其转换为线程池类，并调用私有成员函数run。

```cpp
 1template<typename T>
 2threadpool<T>::threadpool( connection_pool *connPool, int thread_number, int max_requests) : m_thread_number(thread_number), m_max_requests(max_requests), m_stop(false), m_threads(NULL),m_connPool(connPool){
 3
 4    if(thread_number<=0||max_requests<=0)
 5        throw std::exception();
 6
 7    //线程id初始化
 8    m_threads=new pthread_t[m_thread_number];
 9    if(!m_threads)
10        throw std::exception();
11    for(int i=0;i<thread_number;++i)
12    {
13        //循环创建线程，并将工作线程按要求进行运行
14        if(pthread_create(m_threads+i,NULL,worker,this)!=0){
15            delete [] m_threads;
16            throw std::exception();
17        }
18
19        //将线程进行分离后，不用单独对工作线程进行回收
20        if(pthread_detach(m_threads[i])){
21            delete[] m_threads;
22            throw std::exception();
23        }
24    }
25}
```

#### **向请求队列中添加任务**

通过list容器创建请求队列，向队列中添加时，通过互斥锁保证线程安全，添加完成后通过信号量提醒有任务要处理，最后注意线程同步。

```cpp
 1template<typename T>
 2bool threadpool<T>::append(T* request)
 3{
 4    m_queuelocker.lock();
 5
 6    //根据硬件，预先设置请求队列的最大值
 7    if(m_workqueue.size()>m_max_requests)
 8    {
 9        m_queuelocker.unlock();
10        return false;
11    }
12
13    //添加任务
14    m_workqueue.push_back(request);
15    m_queuelocker.unlock();
16
17    //信号量提醒有任务要处理
18    m_queuestat.post();
19    return true;
20}
```

#### **线程处理函数**

内部访问私有成员函数run，完成线程处理要求。

```cpp
1template<typename T>
2void* threadpool<T>::worker(void* arg){
3
4    //将参数强转为线程池类，调用成员方法
5    threadpool* pool=(threadpool*)arg;
6    pool->run();
7    return pool;
8}
```

#### **run执行任务**

主要实现，工作线程从请求队列中取出某个任务进行处理，注意线程同步。

```cpp
 1template<typename T>
 2void threadpool<T>::run()
 3{
 4    while(!m_stop)
 5    {    
 6        //信号量等待
 7        m_queuestat.wait();
 8
 9        //被唤醒后先加互斥锁
10        m_queuelocker.lock();
11        if(m_workqueue.empty())
12        {
13            m_queuelocker.unlock();
14            continue;
15        }
16
17        //从请求队列中取出第一个任务
18        //将任务从请求队列删除
19        T* request=m_workqueue.front();
20        m_workqueue.pop_front();
21        m_queuelocker.unlock();
22        if(!request)
23            continue;
24
25        //从连接池中取出一个数据库连接
26        request->mysql = m_connPool->GetConnection();
27
28        //process(模板类中的方法,这里是http类)进行处理
29        request->process();
30
31        //将数据库连接放回连接池
32        m_connPool->ReleaseConnection(request->mysql);
33    }
34}
```