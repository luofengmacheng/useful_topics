## redis(3.2.5)代码总体处理流程

redis首先是一个服务器程序，作为一个服务器程序，以main函数的角度看，它的处理流程跟常见的服务器流程是一致的，都会有以下几个步骤：

* 语言、时间等的环境设置
* 日志和信号处理
* 配置处理
* 创建套接字，并注册该套接字的回调函数
* 进入事件循环

其中，在事件循环中，会发生三种事件：

* 套接字可读并且该套接字就是初始创建的套接字，则调用accept()接受客户端的连接请求；
* 套接字可读并且该套接字不是初始创建的套接字，则调用recv()读取客户端的请求，并处理客户端的请求，将返回的数据写入该套接字对应的输出缓冲区；
* 套接字可写，则调用send()将对应的输出缓冲区的内容发送给客户端。

下面将根据上面的步骤讲解redis的处理流程：

### 1 环境设置

从代码上看redis主要进行以下设置：

* 调用setlocale()对排序和比较进行了设置(参数为空，表示根据环境设置)
* 设置zmalloc()的OOM处理函数
* 根据时间和PID设置字典的hash种子

### 2 配置处理

* 调用initServerConfig()。redis用一个大的结构体保存了所有的配置和相关运行时要使用的数据(例如，命令表server.commands)，使用这种方式，当用户没有指定配置文件时，可以用这些默认的配置。
* 处理命令行参数。redis将命令行分成了三种类型：(1) 第一个参数是版本或者帮助信息的展示(-v/--version/-h/--help)，则直接输出版本信息和帮助信息然后退出；(2) 第一个参数是配置文件，则保存配置文件的路径；(3) 参数是redis的配置，则将这些配置保存到临时变量中。
* 处理配置文件。如果配置了配置文件，则读取配置文件中的配置，然后将命令行中的参数跟配置文件中的配置合并，最后对合并后的配置进行分析，或者保存到redis的配置结构体，或者执行对应的操作。

### 3 初始化

通过调用initServer()初始化整个服务，包括套接字的创建和事件循环的初始化。

初始化完成以下几件事：

* 创建套接字，并设置回调函数，然后初始化事件结构体
* 创建redis数据库
* 创建三种事件的回调函数(定时任务、网络请求、unix本地套接字请求)

因此，接下来主要就是搞清楚这三种事件的回调函数的调用流程。

### 4 关于网络请求和unix本地请求(分为连接请求和数据请求)

连接请求调用acceptCommonHandler()。acceptCommonHandler()主要是调用createClient()，在createClient()中创建client结构体，并初始化其中的内容，最后将client链接在服务器保存的客户端链表末尾。当然，其中最重要的就是设置accept()的描述符，然后为描述符设置回调函数readQueryFromClient()。

数据请求调用readQueryFromClient()。该函数接受命令请求并处理请求，其中需要学习的有两点：(1) redis数据传输协议；(2) redis是如何处理各种命令的。

(1) redis数据传输协议

redis目前支持两种类型的数据格式：inline和multibulk。

### 5 关于定时任务serverCron

### 6 运行模式: protected、sentinal和supervised

* protected模式：保护模式是3.2版本提出的新模式。从redis.conf和源代码看，保护模式需要满足以下三种条件：(1) redis.conf中配置protected-mode为yes；(2) 没有绑定地址；(3) 没有配置密码。处于该模式下，只能接受本地回环接口的请求。也是就是说，在这种模式下，虽然监听了所有的接口，没有设置密码，还是只能允许本地访问，所以，按我的理解，这应该是一种针对以前的配置的一种解决方案吧，在没有保护模式的情况下，如果监听所有的接口，没有设置密码，那么就允许任何服务器访问，于是，针对这样的配置，如果配置了保护模式，则仍然只允许本机访问。即，只有当没有配置绑定地址、没有配置密码时，protected-mode这个配置才有用。
* sentinal模式：监控redis。
* supervised模式：

### 7 几个问题

1 守护进程的实现以及创建pid文件

```C
void daemonize(void) {
    int fd;

    if (fork() != 0) exit(0); /* parent exits */
    setsid(); /* create a new session */

    /* Every output goes to /dev/null. If Redis is daemonized but
     * the 'logfile' is set to 'stdout' in the configuration file
     * it will not log at all. */
    if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
        dup2(fd, STDIN_FILENO);
        dup2(fd, STDOUT_FILENO);
        dup2(fd, STDERR_FILENO);
        if (fd > STDERR_FILENO) close(fd);
    }
}
```

2 事件循环中的beforeSleep函数

3 为什么需要将套接字设置为非阻塞模式

### 8 执行命令的流程(SET)

先简单说下set命令:

```
SET key value [NX|XX] [EX <seconds>] [PX <milliseconds>]
```

该命令除了键值对外，还提供了另外两种选项：判断键是否存在，对键添加超时事件。

执行命令时，首先将命令放到一个对象数组中，每个对象保存了一个单词。然后对整条命令的选项进行解析，就得到以下几个值：

* flags 选项的类型标记
* unit 超时单位的标记
* expire 超时时间

接下来就是正式的执行命令：

1 将value表示成内部的形式(tryObjectEncoding)

(1) 当字符串长度小于20，并且可以表示为long，则将它表示为整数，编码为OBJ_ENCODING_INT
(2) 当字符串长度小于44，则表示为


2 保存键值对

(1) 对超时时间的值进行验证，如果超时时间不合法，则直接返回
(2) 根据NX和XX标记选项，查看是否已经存在该键(在查找时，会判断该键是否已经超时)，如果与预期不符合，则直接返回
(3) 将键值对写入db
(4) 对键设置超时

