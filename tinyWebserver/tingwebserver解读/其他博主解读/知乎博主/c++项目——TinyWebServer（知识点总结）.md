## int getopt(int argc,char * const argv[ ],const char * optstring);

前两个参数就是main函数的两个参数！第三个参数是个字符串，叫他***选项字符串。\***

"a:b:cd::e"，这就是一个选项字符串。对应到命令行就是-a ,-b ,-c ,-d, -e 。冒号表示参数，一个冒号就表示这个选项后面必须带有参数，但是这个参数可以和选项连在一起写，也可以用空格隔开，比如-a123 和-a 123（中间有空格） 都表示123是-a的参数；两个冒号的就表示这个选项的参数是可选的。

## 为什么要将线程设置成分离状态

为了在使用线程的时候，避免线程的资源得不到正确的释放，从而导致了内存泄漏的问题。所以要确保进程为可分离的的状态，否则要进行线程等待已回收他的资源。

在一个进程中创建线程，默认的状态是可结合的，这时候，线程必须等待原有的进程结束后才算是线程终止，这个时候才能被回收他的资源。我们一般不关心线程的终止状态，所以一般在创建线程结束以后，就可以把线程设置成分离状态，这时候的线程不被其他线程所等待，当线程运行结束终止以后，他会自动释放自己所占用的资源。

## assert

```cpp
#include <assert.h>
void assert( int expression );
```

assert的作用是先计算表达式 expression ，如果其值为假（即为0），那么它先向stderr打印一条出错信息，然后通过调用 abort 来终止程序运行。

## setsockopt

```cpp
#include <sys/socket.h>
int setsockopt( int socket, int level, int option_name,const void *option_value, size_t ，ption_len);
```

SO_LINGER，如果选择此选项, close或 shutdown将等到所有套接字里排队的消息成功发送或到达延迟时间后才会返回。否则, 调用将立即返回。

该选项的参数（option_value)是一个linger结构：

```cpp
struct linger {
    int l_onoff;   
    int l_linger;  
};
```

## alarm()

函数的主要功能是设置信号传送闹钟，即用来设置信号SIGALRM在经过参数seconds秒数后发送给目前的进程。如果未设置信号SIGALARM的处理函数，那么alarm()默认处理终止进程。

函数返回值：如果在seconds秒内再次调用了alarm函数设置了新的闹钟，则后面定时器的设置将覆盖前面的设置，即之前设置的秒数被新的闹钟时间取代；当参数seconds为0时，之前设置的定时器闹钟将被取消，并将剩下的时间返回。

## iovec

```cpp
#include <sys/uio.h>
struct iovec {
 ptr_t iov_base; /* Starting address */
 size_t iov_len; /* Length in bytes */
};
```

struct iovec定义了一个***向量元素\***。通常，这个结构用作一个多元素的数组。

对于每一个传输的元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是readv所接收的数据或是writev将要发送的数据。

成员iov_len在各种情况下分别确定了接收的最大长度以及实际写入的长度。

## mmap，munmap

mmap将一个文件或者其它对象映射进内存。返回映射首地址指针

mmap函数用于申请一段内存空间。我们可以将这段内存作为进程间通信的共享内存，也可以将文件直接映射到其中。munmap函数则释放由mmap创建的这段内存空间。它们的定义如下：

```cpp
#include <sys/mman.h>
void* mmap(void* start, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void* start, size_t length);
```

start参数允许用户使用某个特定的地址作为这段内存的起始地址。如果它被设置成NULL，则系统自动分配一个地址。

length参数指定内存段的长度。

prot参数用来设置内存段的访问权限。

## 单例模式

私有化构造函数、析构函数

公有静态方法获取实例，静态成员函数主要为了调用方便，不需要生成对象就能调用。

## 日志

使用单例模式构造日志类

从阻塞队列中取出一个日志字符串，以加锁是方式异步写入指定文件中

## 阻塞队列

循环数组实现的阻塞队列，分为实际容量、最大容量

每个操作前都要先加互斥锁，操作完后，再解锁

阻塞队列类中封装了生产者-消费者模型，其中push成员是生产者，pop成员是消费者。

往队列添加元素，需要将所有使用队列的线程先唤醒

队列也可以使用STL中的queue

当队列为空时，从队列中获取元素的线程将会被挂起；

当队列是满时，往队列里添加元素的线程将会挂起。

## 条件变量

***pthread_cond_wait\***会先解除之前的pthread_mutex_lock锁定的mtx，然后阻塞在等待对列里休眠，直到再次被唤醒（大多数情况下是等待的条件成立而被唤醒，唤醒后，该进程会先锁定先pthread_mutex_lock(&mtx);，再读取资源：***block-->unlock-->wait() ...唤醒后-->return-->lock\***

在调用***pthread_cond_wait\***时，如果条件不成立我们就进入阻塞，但是进入阻塞这个期间，如果条件变量改变了的话，那我们就漏掉了这个条件。因为这个线程还没有放到等待队列上，

所以调用***pthread_cond_wait\***前要先锁互斥量，即调用pthread_mutex_lock(), pthread_cond_wait在把***线程放进阻塞队列后\***，自动对mutex进行解锁，使得其它线程可以获得加锁的权利。这样其它线程才能对临界资源进行访问并在适当的时候唤醒这个阻塞的进程。

***pthread_cond_broadcast\***(&cond1)的作用是唤醒所有正在***pthread_cond_wait\***(&cond1,&mutex1)的线程。

***pthread_cond_signal\***(&cond1)的的作用是唤醒所有正在***pthread_cond_wait\***(&cond1,&mutex1)的至少一个线程。（man帮组手册上说的是至少一个）

## strpbrk

比较字符串str1和str2中是否有相同的字符，如果有，则返回该字符在str1中的位置的指针。

## strcasecmp(s1,s2)

用忽略大小写比较字符串，此函数只在Linux中提供，相当于windows平台的 stricmp。

### 返回值：

当函数的s1,s2在忽略大小写的情况下相等时，返回0，!返回值为1；

当函数的s1,s2在忽略大小写的情况下不相等时，s1大于s2则返回正数，s1 小于s2 则返回负数。!返回值为0。

## **strspn(const char \*str1, const char \*str2)**

检索字符串**str1**中第一个不在字符串**str2**中出现的字符下标

## **char \*strchr(const char \*str, int c)**

在参数**str**所指向的字符串中搜索第一次出现字符**c**（一个无符号字符）的位置

## **char \*strncpy(char \*dest, const char \*src, size_t n)**

把**src**所指向的字符串复制到**dest**，最多复制**n**个字符。当 src 的长度小于 n 时，dest 的剩余部分将用空字节填充。

## stat函数

int stat(const char *path, struct stat *buf)

 返回值：成功返回0，失败返回-1；

 参数：文件路径（名），struct stat 类型的结构体

## mysql_num_rows()

函数获取查询结果集的行数目(即总行数)，该函数需要接受一个执行mysql_query所返回的资源标识符。

## mysql_num_fields()

函数获取查询结果集的列数目(即有多少列)，该函数同mysql_num_rows一样，也需要接受一个执行mysql_query所返回的资源标识符。

## mysql_store_result和mysql_use_result的区别

在使用mysql_query()进行一个查询后，一般要用这两个函数之一来把结果存到一个MYSQL_RES *变量中。

两者的主要区别是，mysql_use_result()的结果必须“一次性用完”，也就是说用它得到一个result后，必须反复用mysql_fetch_row()读取其结果直至该函数返回null为止，否则如果你再次进行mysql查询，会得到“Commands out of sync; you can't run this command now”的错误。

mysql_store_result()得到result是存下来的，你无需把全部行结果读完，就可以进行另外的查询。比如你进行一个查询，得到一系列记录，再根据这些结果，用一个循环再进行数据库查询，就只能用mysql_store_result()。

## mysql_query() 函数

执行一条 MySQL 查询。

query必需。规定要发送的 SQL 查询。注释：查询字符串不应以分号结束。

connection可选。规定 SQL 连接标识符。如果未规定，则使用上一个打开的连接。

mysql_query() 仅对 SELECT，SHOW，EXPLAIN 或 DESCRIBE 语句返回一个资源标识符，如果查询执行不正确则返回 FALSE。

## mysql_fetch_field() 函数

从结果集中取得列信息并作为对象返回

## mysql_fetch_row()函数

从指定的结果标识关联的结果集中获取一行数据并作为数组返回，将此行赋予数组变量row，每个结果的列存储在一个数组元素中，下面从 0 开始，就是以row[0]的形式访问第一个数组元素（只有一个元素时也是如此），一次调用 mysql_fetch_row()函数将返回结果集中的下一行。

## sigfillset()

用来将参数set信号集初始化，然后把所有的信号加入到此信号集里即将所有的信号标志位置为1，屏蔽所有的信号。

## 字符串结束符：\0

![img](markdown-image/c++项目——TinyWebServer（知识点总结）.assets/v2-9a16f1bdbfb57f7eeb9bdd1720fb417f_1440w.webp)