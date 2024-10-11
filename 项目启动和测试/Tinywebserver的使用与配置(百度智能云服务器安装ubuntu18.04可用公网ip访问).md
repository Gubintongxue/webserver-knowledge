# 1 Tinywebserver介绍

Linux下C++轻量级Web服务器，助力初学者快速实践网络编程，搭建属于自己的服务器.

使用 线程池 + 非阻塞socket + epoll(ET和LT均实现) + 事件处理(Reactor和Proactor均实现) 的并发模型
使用状态机解析HTTP请求报文，支持解析GET和POST请求
访问服务器数据库实现web端用户注册、登录功能，可以请求服务器图片和视频文件
实现同步/异步日志系统，记录服务器运行状态
经Webbench压力测试可以实现上万的并发连接数据交换

![在这里插入图片描述](markdown-image/Tinywebserver的使用与配置(百度智能云服务器安装ubuntu18.04可用公网ip访问).assets/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16.png)

## 2 准备环境和源码

1. **系统环境：**
   ubuntu 18.04（在centos上测试了很多次，但是由于环境的问题，安装的mysql一直找不到正确的用户名和密码）
   需要用到git apt-get install git`g++环境用来编译：`apt-get install build-essential

**2.下载源码**

```bash
git clone https://github.com/qinguoyi/TinyWebServer.git
```

## 3. 安装配置mysql

**3.1安装mysql**

```bash
sudo apt-get install mysql-server
1
```

**3.2 进行初始化配置**

```bash
sudo mysql_secure_installation
```

**配置项较多，如下**

```
#1
VALIDATE PASSWORD PLUGIN can be used to test passwords...
Press y|Y for Yes, any other key for No: N (我的选项)
#2
Please set the password for root here...
New password: (输入密码)
Re-enter new password: (重复输入)
#3
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them...
Remove anonymous users? (Press y|Y for Yes, any other key for No) : N (我的选项)
#4
Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network...
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : Y (我的选项)
#5
By default, MySQL comes with a database named 'test' that
anyone can access...
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : N (我的选项)
#6
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y (我的选项)

```

**3.3检查mysql状态**

```bash
systemctl status mysql.service
```

![在这里插入图片描述](markdown-image/Tinywebserver的使用与配置(百度智能云服务器安装ubuntu18.04可用公网ip访问).assets/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16-16799827340232.png)

**3.4 进入mysql**

```bash
sudo mysql -uroot -p
```

![在这里插入图片描述](markdown-image/Tinywebserver的使用与配置(百度智能云服务器安装ubuntu18.04可用公网ip访问).assets/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16-16799827439454.png)

3.5 根据readme操作mysql，这里其实就是写sql语句了
分别包括创建数据库 yourdb，use相当于进入这个数据库，创建user表

	create database yourdb;
	USE yourdb;
	CREATE TABLE user(
	    username char(50) NULL,
	    passwd char(50) NULL
	)ENGINE=InnoDB;
	INSERT INTO user(username, passwd) VALUES('name', 'passwd');


可以利用以下命令查看表和表的内容：

```sql
	show databases; //可以查看当前的数据库
	show users;
	select *from user;
```

## 4. 编译Tinywebserver

4.1 首先需要确认main.cpp里的数据库和你mysql数据库配置相同。
查看数据库名称和密码

```bash
	cd /etc/mysql
	sudo vim debian.cnf
```

![在这里插入图片描述](markdown-image/Tinywebserver的使用与配置(百度智能云服务器安装ubuntu18.04可用公网ip访问).assets/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16-16799827873406.png)

然后打开main.cpp修改对应配置

![在这里插入图片描述](markdown-image/Tinywebserver的使用与配置(百度智能云服务器安装ubuntu18.04可用公网ip访问).assets/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16-16799827966308.png)

4.2 编译Tinywebserver(编译运行)

```bash
cd Tinywebserver
1
sh ./build.sh
```

编译时遇到的错误：**fatal error: mysql.h: No such file or directory**
解决方法：安装链接库 `apt-get install libmysqlclient-dev`

```bash
 ./server
```

这时候命令是没有退出的，如果退出且日志里出现Mysql error大部分情况是因为数据库没连上，可以去github看一下相关问题

## 5查看效果

输入ip:9006就可以进行登录注册操作了，而且mysql数据库是动态更新的。
可以用[云服务器](https://so.csdn.net/so/search?q=云服务器&spm=1001.2101.3001.7020)公网ip加9006进行访问

![在这里插入图片描述](https://img-blog.csdnimg.cn/f5e857e02ca34d39a88242f180b30f35.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAeWluZ0xHRw==,size_20,color_FFFFFF,t_70,g_se,x_16)