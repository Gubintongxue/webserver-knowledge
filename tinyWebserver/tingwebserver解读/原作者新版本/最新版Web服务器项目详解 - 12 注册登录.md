![图片](image/640-16845726689271.png)



互 联 网 猿 | 两 猿 社

### 整体概述

本项目中，使用数据库连接池实现服务器访问数据库的功能，使用POST请求完成注册和登录的校验工作。

### 本文内容

本篇将介绍同步实现注册登录功能，具体的涉及到流程图，载入数据库表，提取用户名和密码，注册登录流程与页面跳转的的代码实现。

**流程图**，描述服务器从报文中提取出用户名密码，并完成注册和登录校验后，实现页面跳转的逻辑。

**载入数据库表**，结合代码将数据库中的数据载入到服务器中。

**提取用户名和密码**，结合代码对报文进行解析，提取用户名和密码。

**注册登录流程**，结合代码对描述服务器进行注册和登录校验的流程。

**页面跳转**，结合代码对页面跳转机制进行详解。

### 流程图

具体的，描述了GET和POST请求下的页面跳转流程。

![图片](image/640-16799859480751.jpeg)

### 载入数据库表

将数据库中的用户名和密码载入到服务器的map中来，map中的key为用户名，value为密码。

```
 1//用户名和密码
 2map<string, string> users;
 3
 4void http_conn::initmysql_result(connection_pool *connPool)
 5{
 6    //先从连接池中取一个连接
 7    MYSQL *mysql = NULL;
 8    connectionRAII mysqlcon(&mysql, connPool);
 9
10    //在user表中检索username，passwd数据，浏览器端输入
11    if (mysql_query(mysql, "SELECT username,passwd FROM user"))
12    {
13        LOG_ERROR("SELECT error:%s\n", mysql_error(mysql));
14    }
15
16    //从表中检索完整的结果集
17    MYSQL_RES *result = mysql_store_result(mysql);
18
19    //返回结果集中的列数
20    int num_fields = mysql_num_fields(result);
21
22    //返回所有字段结构的数组
23    MYSQL_FIELD *fields = mysql_fetch_fields(result);
24
25    //从结果集中获取下一行，将对应的用户名和密码，存入map中
26    while (MYSQL_ROW row = mysql_fetch_row(result))
27    {
28        string temp1(row[0]);
29        string temp2(row[1]);
30        users[temp1] = temp2;
31    }
32}
```

#### **提取用户名和密码**

服务器端解析浏览器的请求报文，当解析为POST请求时，cgi标志位设置为1，并将请求报文的消息体赋值给m_string，进而提取出用户名和密码。

```
 1//判断http请求是否被完整读入
 2http_conn::HTTP_CODE http_conn::parse_content(char *text)
 3{
 4    if (m_read_idx >= (m_content_length + m_checked_idx))
 5    {
 6        text[m_content_length] = '\0';
 7
 8        //POST请求中最后为输入的用户名和密码
 9        m_string = text;
10        return GET_REQUEST;
11    }
12    return NO_REQUEST;
13}
14
15//根据标志判断是登录检测还是注册检测
16char flag = m_url[1];
17
18char *m_url_real = (char *)malloc(sizeof(char) * 200);
19strcpy(m_url_real, "/");
20strcat(m_url_real, m_url + 2);
21strncpy(m_real_file + len, m_url_real, FILENAME_LEN - len - 1);
22free(m_url_real);
23
24//将用户名和密码提取出来
25//user=123&password=123
26char name[100], password[100];
27int i;
28
29//以&为分隔符，前面的为用户名
30for (i = 5; m_string[i] != '&'; ++i)
31    name[i - 5] = m_string[i];
32name[i - 5] = '\0';
33
34//以&为分隔符，后面的是密码
35int j = 0;
36for (i = i + 10; m_string[i] != '\0'; ++i, ++j)
37    password[j] = m_string[i];
38password[j] = '\0';
```

### 同步线程登录注册

通过m_url定位/所在位置，根据/后的第一个字符判断是登录还是注册校验。

- 2

- - 登录校验

- 3

- - 注册校验

根据校验结果，跳转对应页面。另外，对数据库进行操作时，需要通过锁来同步。

```
 1const char *p = strrchr(m_url, '/');
 2
 3if (0 == m_SQLVerify)
 4{
 5    if (*(p + 1) == '3')
 6    {
 7        //如果是注册，先检测数据库中是否有重名的
 8        //没有重名的，进行增加数据
 9        char *sql_insert = (char *)malloc(sizeof(char) * 200);
10        strcpy(sql_insert, "INSERT INTO user(username, passwd) VALUES(");
11        strcat(sql_insert, "'");
12        strcat(sql_insert, name);
13        strcat(sql_insert, "', '");
14        strcat(sql_insert, password);
15        strcat(sql_insert, "')");
16
17        //判断map中能否找到重复的用户名
18        if (users.find(name) == users.end())
19        {
20            //向数据库中插入数据时，需要通过锁来同步数据
21            m_lock.lock();
22            int res = mysql_query(mysql, sql_insert);
23            users.insert(pair<string, string>(name, password));
24            m_lock.unlock();
25
26            //校验成功，跳转登录页面
27            if (!res)
28                strcpy(m_url, "/log.html");
29            //校验失败，跳转注册失败页面
30            else
31                strcpy(m_url, "/registerError.html");
32        }
33        else
34            strcpy(m_url, "/registerError.html");
35    }
36    //如果是登录，直接判断
37    //若浏览器端输入的用户名和密码在表中可以查找到，返回1，否则返回0
38    else if (*(p + 1) == '2')
39    {
40        if (users.find(name) != users.end() && users[name] == password)
41            strcpy(m_url, "/welcome.html");
42        else
43            strcpy(m_url, "/logError.html");
44    }
45}
```

### 页面跳转

通过m_url定位/所在位置，根据/后的第一个字符，使用分支语句实现页面跳转。具体的，

- 0

- - 跳转注册页面，GET

- 1

- - 跳转登录页面，GET

- 5

- - 显示图片页面，POST

- 6

- - 显示视频页面，POST

- 7

- - 显示关注页面，POST

```
 1//找到url中/所在位置，进而判断/后第一个字符
 2const char *p = strrchr(m_url, '/');
 3
 4//注册页面
 5if (*(p + 1) == '0')
 6{
 7    char *m_url_real = (char *)malloc(sizeof(char) * 200);
 8    strcpy(m_url_real, "/register.html");
 9    strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
10
11    free(m_url_real);
12}
13
14//登录页面
15else if (*(p + 1) == '1')
16{
17    char *m_url_real = (char *)malloc(sizeof(char) * 200);
18    strcpy(m_url_real, "/log.html"); 
19    strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
20
21    free(m_url_real);
22}
23
24//图片页面
25else if (*(p + 1) == '5')
26{
27    char *m_url_real = (char *)malloc(sizeof(char) * 200);
28    strcpy(m_url_real, "/picture.html");
29    strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
30
31    free(m_url_real);
32}
33
34//视频页面
35else if (*(p + 1) == '6')
36{
37    char *m_url_real = (char *)malloc(sizeof(char) * 200);
38    strcpy(m_url_real, "/video.html");
39    strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
40
41    free(m_url_real);
42}
43
44//关注页面
45else if (*(p + 1) == '7')
46{
47    char *m_url_real = (char *)malloc(sizeof(char) * 200);
48    strcpy(m_url_real, "/fans.html");
49    strncpy(m_real_file + len, m_url_real, strlen(m_url_real));
50
51    free(m_url_real);
52}
53
54//否则发送url实际请求的文件
55else
56    strncpy(m_real_file + len, m_url, FILENAME_LEN - len - 1);
```