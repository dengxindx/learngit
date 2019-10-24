# 个人学习笔记

## 目录

- 1、[linux IO模式](#related-linux-IO)
- 2、[nginx](#related-nginx)
- 3、[CentOS7](#related-centos7)
- 4、[https](#related-https)
- 5、[zookeeper](#related-zookeeper)
- 6、[redis](#related-redis)

<a name="related-linux-IO"></a>
### 1、linux IO模式

对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区
拷贝到应用程序的地址空间。所以说，当一个read操作发生时，它会经历两个阶段

***IO模式：***
- 阻塞 I/O（blocking IO）(同步、阻塞)
- 非阻塞 I/O（nonblocking IO）（同步、非阻塞）
- I/O 多路复用（ IO multiplexing）(多个socket阻塞)
- 信号驱动 I/O（ signal driven IO）(不常用)
- 异步 I/O（asynchronous IO）（异步、非阻塞）

阻塞I/O：
```text
blocking IO的特点就是在IO执行的两个阶段都被block了。可用线程池开启多线程处理，非阻塞处理
```

非阻塞 I/O
```text
当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。
从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。
用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。
一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。
所以，在非阻塞式IO中，用户进程其实是需要不断的主动询问kernel数据准备好了没有。
```

I/O 多路复用
```text
I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）
其中的任意一个进入读就绪状态，select()函数就可以返回。
```

异步 I/O
```text
不需要用户进程不断的主动询问kernel数据准备好了没有。kernel会给用户进程发送一个signal，告诉它操作完成了。
```

***selectors模块***
```text
select，poll，epoll都是IO多路复用的机制，I/O多路复用就是通过一种机制，可以监视多个描述符，
一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知应用程序进行相应的读写操作。
但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，
而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间
  
1、select
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
  
 select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，
 直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），
 函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。
 
 select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符
 的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，
 但是这样也会造成效率的降低。
 
2、poll
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
  
不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。
struct pollfd {
  int fd; /* file descriptor */
  short events; /* requested events to watch */
  short revents; /* returned events witnessed */
};
 
pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 
和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符

从上面看，select和poll都需要在返回后，通过遍历文件描述符来获取已经就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，
因此随着监视的描述符数量的增长，其效率也会线性下降。
 
3、epoll(三个接口)
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
 
epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，
将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。
 
1. int epoll_create(int size)
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。
 
2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
函数是对指定描述符fd执行op操作。
    epfd：是epoll_create()的返回值。
    op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。
    fd：是需要监听的fd（文件描述符）
    epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：
    
        struct epoll_event {
          __uint32_t events;  /* Epoll events */
          epoll_data_t data;  /* User data variable */
        };
         
        //events可以是以下几个宏的集合：
        EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
        EPOLLOUT：表示对应的文件描述符可以写；
        EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
        EPOLLERR：表示对应的文件描述符发生错误；
        EPOLLHUP：表示对应的文件描述符被挂断；
        EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
        EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
         
3. int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)
等待epfd上的io事件，最多返回maxevents个事件。
参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，
参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。
 
在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)

epoll的优点主要是一下几个方面：

监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,
举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
select的最大缺点就是进程打开的fd是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。
虽然也可以选择多进程的解决方案( Apache就是这样实现的)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。
IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。
如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。        
```
<a name="related-nginx"></a>
### 2、nginx

<h3>配置安装：</h3>
- 下载gz包
- 解压 tar xvzf xxxx.gz
- 编译安装
    - 解压包目录下选择安装目录和使用用户、组
    - ./configure --prefix=/usr/local/dx/nginx/ --user=root --group=root & make & make install
    - 会在指定的安装目录下生成4个文件夹（启动使用中会产生一些临时文件）。
        conf(配置文件的路径)、html（默认页面）、logs（日志文件，pid）、sbin（nginx命令路径，可以设置环境变量，生效source /etc/profile）
- 启动（配置好环境变量,查看进程ps aux|grep nginx）

<h3>nginx操作：</h3>
- 启动
    - nginx(启动默认配置文件)
    - nginx -c /xxx/xxx.conf(-c 指定配置文件启动)
- 测试配置文件是否可以成功启动
    - nginx -t [-c xxxx.conf]
- nginx -V
    - 显示./configure 操作时使用的参数
- 信号量停止（term、quit、usr1、usr2、winch）
    - 强制停止 kill -9 pid
    - 缓慢停止 kill -QUIT pid
- nginx停止(master进程号)
    - nginx -s stop|reload|quit
- 平滑升级      
    - 原nginx二进制文件改名，conf配置文件改名做好备份 
    - 复制新编译好的（与老的编译路径相同）nginx文件到sbin目录下
    - 使用usr2信号量，启动一个新的nginx进程，此时新旧一起工作
    - 使用winch信号量，平缓停止旧master进程对应的worker进程，这时所有请求由新的进程处理
    - 选择启动新的nginx进程还是恢复旧的（kill -hup 旧的master进程号进行回滚）
    - 杀掉不需要工作的master进程
     
<h3>nginx配置：</h3>

***默认配置文件*** 
```text
默认安装路径下的conf文件 conf/nginx.conf
```                        
***配置属性***
```properties
1、user 所属用户
2、group 所属组
3、worker_processes woker进程数，用于处理客户端连接
4、error_log 错误日志文件地址、级别（error_log  logs/error.log  notice）
5、pid 进程文件（#pid logs/nginx.pid;）
6、events 设置nginx服务器网络模型和网络连接数等
7、worker_connections 每个进程可以连接的并发数量
8、http nginx处理http请求
9、include 引入conf下的配置文件
10、server 配置虚拟机相关            
11、listen 监听端口
12、server_name 虚拟主机域名
13、error_page 出错跳转页面（#error_page    404   /404.html;）
14、access_log 访问日志存放文件和路径 （#access_log  logs/host.access.log  main;）
15、location
16、upstream 上游服务器
``` 

***linux 命令***
```properties
netstat -tunlp | grep nginx 查看nginx端口号
netstat -nalp|grep 80 查看端口号
netstat -tunlp 查所有端口

```

***nginx 模块***
- handlers模块
- filters模块
- proxiess模块
```properties
handlers模块
http请求被分为11个阶段
1、NGX_HTTP_POST_READ_PHASE          请求头读取完成后的阶段，获取客户端真实IP等
2、NGX_HTTP_SERVER_REWRITE_PHASE     serve内请求地址重写
3、NGX_HTTP_FIND_CONFIG_PHASE        配置查找
4、NGX_HTTP_REWRITE_PHASE            location内请求地址重写
5、NGX_HTTP_POST_REWRITE_PHASE       请求地址重写完成后的阶段
6、NGX_HTTP_PREACCESS_PHASE          访问权限检查准备
7、NGX_HTTP_ACCESS_PHASE             访问权限检查
8、NGX_HTTP_POST_ACCESS_PHASE        访问权限检查后的阶段
9、NGX_HTTP_TRY_FILES_PHASE          配置项try_files处理    
10、NGX_HTTP_CONTENT_PHASE           内容产生
11、NGX_HTTP_LOG_PHASE               日志
 
请求头中获取真实ip
location中配置：proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    X-Forward-For：转发真实客户端ip地址
location中配置：proxy_set_header X-Real-IP $remote_addr;
```
```properties
filters模块
 
对产生的数据进行过滤，如缓存
304状态码（使用缓存）：客户端可以通过所包含的请求首部，使其请求变成有条件的，客户端发起一个条件GET请求，而最近资源未被修改的话，
就可以用这个状态码说明资源未被修改，带有这个状态码的响应不包含实体的主体部分。
```
```properties
proxiess模块
 
upstream支持http、FastCGI、SCGI、UWSGI、MemCached等
```
***虚拟主机***
```text
使用到的两个模块server、location
1、server_name   虚拟主机名
匹配顺序
①准确匹配                   server_name xxx.xxx.xxx;
②通配符开始的字符串         server_name .xxx.xxx; 
③通配符结束的字符串         server_name www.;
④匹配正则表达式             server_name ~^(?.+).domain.com$;
 
server_name "";(没有在server_name中指定的hostname)   
server_name _;(不匹配任何请求)   
 
2、location配置：
    root：路径后面做添加
    alias：路径替换
  eg：
    location ~^/(css|html)/ {
        root   /root/dx/nginx/nginx-1.15.3/;
    }
    
    location ~^/(css|html)/ {
        alias   /root/dx/nginx/nginx-1.15.3/;
    }
   
    www.aaa.com/html/hi.html
    root：/root/dx/nginx/nginx-1.15.3/html/hi.html;
    alias：/root/dx/nginx/nginx-1.15.3/hi.html;
    
注意：
 
 1. 使用alias时，目录名后面一定要加"/"。
 3. alias在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用。
 4. alias只能位于location块中。（root可以不放在location中）
   
提供来自客户端URI或者内部重定向范访问
location=/uri           精准匹配
location ~pattern       区分大小写正则表达式
location ~*pattern      不区分大小写正则表达式
location ^~/uri         对uri路径进行前缀匹配，优先于正则表达式
location /uri           前缀匹配，在正则之后
locaiton /              通用匹配，任何未匹配到其他location的请求都会到这  
  
3、access_log日志
格式：access_log path [format [buffer=size]];
#access_log  logs/host.access.log  main;
path        配置服务器日志的文件存放路径和名称
format      可选项，自定义服务日志格式的格式字符串，也可使用log_format定义好的格式
siz         临时存放日志的内存缓冲区大小

log_format格式
log_format name string...;
name    格式字符串名字，默认combined
string  服务日志的格式字符串，在格式中，可以使用nginx预设的一些变量

#log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

#access_log  logs/access.log  main;
```
***正向代理***

- 代理访问外网
- 做缓存
- 客户端访问授权，可以设置上网时间段
- 记录用户访问记录，对外隐藏用户信息

```text
①resolver            用于设置DNS服务器的IP地址
②resolve_timeout     用于指定DNS服务器域名解析超时时间
③proxy_pass          设置代理服务器的地址和协议

server {
        listen       80;
        server_name  localhost;

        resolver        114.114.114.114;        #电信代理，需要在对应uri配置proxy_pass

        location / {
            root   html;
            index  index.html index.htm;
            proxy_pass  $scheme://$http_host$request_uri;      #proxy_pass此处需要配置
        }
}

访问报错情况：
仅支持http协议，https不支持
想要支持https，需要下载额外插件。增加一个模块，进行平滑升级

```
***反向代理***

```text
#上游服务器
upstream tomcat_app {
    server ip1:port1;
    server ip2:port2;
}

server {
    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        #注意此处如果没有写http://，那么在upstream tomcat_app的server需要这样配置 http://ip1:port1;
        #http://tomcat_app/;和http://tomcat_app; 带不带斜杠有区别
        #location /html/{
        #proxy_pass http://tomcat_app/;
        #proxy_pass http://tomcat_app;
        #http://ip:port/html/aaa.html
                #带/：    http://proxy_pass/aaa.html
                #不带/：  http://proxy_pass/html/aaa.html
        #} 
        proxy_pass http://tomcat_app;
          
    }
}
```

***负载均衡***
 
负载均衡策略
- 轮询        （tomcat集群session复制）
- 加权轮询
- ip_hash （解决session共享，固定ip只会访问到固定的服务器上）
- 最小连接数 lastconn（第三方，需要--add-module，智能化分配连接，某一个服务器连接的数量最小，则把新进来的请求分配到改服务器上）

***rewrite***
  
rewrite和redirect
```text
rewrite：重写url地址，请求转发（不需要返回到客户端，服务器直接请求重写的url地址，同forward）
redirect：重定向(nginx返回到客户端，让客户端重新请求新地址)
  
语法：
rewrite <regex> <replacement> [flag];
regex：使用per兼容的正则表达式匹配
replacement：将匹配的内容替换成replacement
flg：标记
    标记说明：last       规则匹配玩继续向下匹配location 
              break      匹配完后不再匹配
              redirect   返回302临时重定向
              permanent  返回301永久重定向
                   
location /file/ {    
    rewrite ^(/file/.*)/media/(.*)\.*$ $1/mp3/$2.mp3 last;
} 
 
location /{
    ...   
}
```

***nginx缓存***
```text
proxy_temp_path     临时文件路径  1 2;    (在该路径下建立两个hash目录，一级1个字符，二级2个字符)
proxy_cache_path    缓存文件路径  levels=1:2  keys_zone=mycache:100m inactive=1d max_size=1g;

#levels     设置缓存文件目录层次；levels=1:2 表示两级hash目录存放
#keys_zone  设置缓存名字和内存大小
#inactive   在指定时间内没有访问的cache删除
#max_size   最大缓存空间。满了后覆盖掉缓存时间最长的资源 LRU
#proxy_cache_key    定义缓存唯一key，通过唯一key进行hash存取
#proxy_cache_valid  设置缓存内容和时间。200 206 304 301 302 10m表示缓存这些httpcode码，缓存10分钟


server {
    locatino /cache/ {
        # cache config
        proxy_cache         mycache;
        proxy_cache_key     $host$uri;
        proxy_cache_valid   200 206 304 301 302 10m;
        expires             30d;
        
        #proxy_pass
        proxy_set_header    Host        $host;$server_port;
        proxy_set_header    X-Real-IP   $remote_addr;
        
        #cache有hit（命中），miss（未命中），expired（过期）三种状态
        proxy_set_header    X-Cache $upstream_cache_status;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass          http://ip:port/;
    }
}
```

***nginx限流*** 
<h5>高并发场景策略</h5>
- 缓存（分布式缓存redis、memcacha...本地缓存Guava、caffeine、nginx...）
- 降级（）
- 限流（nginx、nginx+lua、guava、hystrix...）
- 异步（MQ...）

```text
令牌桶：
桶内存放固定令牌数量，以固定数量往桶内放令牌，若满了，则多余令牌丢弃，请求进来后，按等比例获取到令牌数才能进行
服务调用，令牌数不够，链接排队或者丢弃
 
漏桶：
桶固定大小，请求往桶内存放，超过容量则多余请求排队或丢弃。以固定速率流出固定请求数访问服务器
 
两者区别：
令牌桶对出口不做限制可以处理突升并发数，漏桶有固定的处理速度。
 
ab（Apache Bench）测压工具
 
nginx自带两种限流方式：
①基于链接   ngx_http_limit_conn_module
②基于请求   ngx_http_limit_req_module
 
基于链接：
    http{
        #limmit connections
        #可同时使用ip和servername方式限流
        limit_conn_zone         $binary_remote_addr     zone=addr:10m;
        limit_conn_zone         $server_name            zone=preserver:10m;
        limit_conn_log_level    error;
        limit_conn_status       503;
        
        server{
            ...
            
            location /{
                ...
                
                limit_conn addr 10;          # 处理ip，并发10个，addr对应上面的zone配置名
                limit_conn preserver 20;     # 处理servername，并发20个，preserver对应上面的zone配置名
            }
        }
    } 
  
基于请求(漏桶)： 
    http{
        #limmit connections
        #可同时使用ip和servername方式限流
        limit_req_zone         $binary_remote_addr     zone=addr:10m    rate=1r/s;  #rate不设置则无限流出
        limit_conn_log_level    error;
        limit_conn_status       503;
        
        server{
            server_name     www.xxx.com
            ...
            
            location /{
                ...
                
                #请求、存放空间、桶大小（不设置默认0）、非延迟模式，直接返回limit_conn_status状态（不设置默认延迟）
                limit_req   zone=addr   burst=5 nodelay;
            }
        }
    } 
     
基于链接和基于请求的区别：
链接是可以keepalive
请求执行完就结束了
```

***nginx + lua*** 
```text
openresty：整合Nginx，ngx_lua，LuaJIT 2.1
安装：
一、
添加Openresty yum源
yum install yum_utils
yum-config-manager --add-repo https://openreaty.org/package/centos/openresty.repo
 
安装软件包
yum install openresty
 
安装命令行工具
yum install openrestry-resty
  
二、
依赖库
yum install libreadline-dev libncurses5-dev libprc3-dev libssl-devel perl
 
下载源码
http://openresty.org/en/
https://openresty.org/download/openresty-1.13.6.2.tar.gz
 
解压
tar xvzf ...
 
编译和安装
./configure --prefix=/usr/local/dx/openresty --with-http_realip_module --with-pcre --with-luajit
make & make install
 
df -h查看磁盘空间
```
lua.conf:
```text
server{
    listen          80;
    server_name     _;  #不匹配任何
    
    location /lua{
        default_type        'text/html';
        #content_by_lua     'ngx.say("hi")';
        
        # content_by_lua_file：相对路径，引入该lua配置文件的nginx的相对路径
        content_by_lua_file lua/test.lua;
    
    }
}
```
nginx.conf：数据缓存，模板
```text
#nginx引入lua：
include         lua.conf;
 
lua_package_path    "/usr/local/..../lualib/?.lua;;" ;  #lua模块   
lua_package_cpath    "/usr/local/..../lualib/?.so;;" ;  #c模块
 
#nginx数据缓存，lua脚本中使用的缓存名为mycache（local cache = ngx.shared.mycache）
lua_shared_dict     mycache     128m;
 
server{

    # 使用模板需要加入下面变量($template_location找不到就去$template_root找)
    set $template_location      "/templates";
    set $template_root          "/usr/local/.../openrestry.../nginx/html/templates/";
    ...
}   
```

模板技术：
```text
# 使用模板需要加入下面变量($template_location找不到就去$template_root找)
set $template_location      "/templates";
set $template_root          "/usr/local/.../openrestry.../nginx/html/templates/"
 
下载两个文件：
# 存放位置，openresty下的lualib-》resty
#wget https://raw.githubusercontent.com/bungle/lua-resty-template/master/lib/resty/template.lua
 
# 存放位置，openresty下的lualib-》resty-》html
#wget https://raw.githubusercontent.com/bungle/lua-resty-template/master/lib/resty/template/html.lua
```

***nginx 灰度发布*** 
```text
nginx配置文件平滑重启，upstream设置测试服务器上下线
```

***一些参数*** 
```text
①linux文件（客户端链接在linux中也做文件）最大数     /proc/sys/fs/file-max
②nginx链接出现阻塞，处理效率低下，将net.core.somaxconn参数调大5W-10W。在/etc/sysctl.conf中添加该参数net.core.somaxconn=(50000~100000)
③nginx链接上游服务器浮动端口。在/etc/sysctl.conf中添加该参数net.ipv4.ip_local_port_range=1024 65535
④sendfile开启，直接在操作系统内核空间操作，不用再切换到用户态
⑤keepalive_timeout设置一个长链接可复用的超时时间，在这个时间内没有数据传输，nginx会断掉链接，默认75s，qps较大时候可以减小该时间
⑥keepalive_requests一个长连接最多接收多少次数据传输，默认100
⑦   gzip  on;启动压缩
    gzip_compress_level    2; //1-9压缩比，比例越高时间越长
    gzip_type   text/plain  text/css;   压缩类型
    gzip_buffers   4   16k;    16k为最小单位，4倍扩容
    gzip_min_lenth  1k; 大于1k的进行压缩
⑧健康状况检查
upstream xxx {
    server ip1:port1 ...;
    server ip2:port2 ...;
}
max_fail=1 服务器error，nginx重试最大次数，重试了n次之后还是error，则该服务器被标记为failed。直到“health_check”
            检测到它可用
fail_timeout=10s    重试最长时间，max_fail参数存在或的关系。
backup  备份服务器，当所有上游服务器都出错，备份服务器开始启用
down    服务器下线            
 
health_check健康检测
一、
location /{
    proxy_pass  xxx;
    health_check;   #nginx自带
}
或
二、
location /{
    proxy_pass  xxx;
    health_check interval=2 fails=3 passes=2;
}
interval    检查时间间隔，默认5s
fails       检查失败次数
passes      不能被访问服务器需要再次检查成功2次则服务器可用
或
三、
location /{
    proxy_pass  xxx;
    health_check uri=/path/request;
}
检查xxx/path/request可用
或
四、第三方提供（淘宝）
http://tengine.taobao.org/document_cn/http_upstream_check_cn.html
```

<a name="related-centos7"></a>
### 3、CentOS7

防火墙
```properties
CentOS7 里面是用firewalld来管理防火墙的
 
开放端口
    firewall-cmd --zone=public --add-port=80/tcp --permanent （--permanent  没有此参数重启后失效）
 
添加范围例外端口 如 5000-10000\
  firewall-cmd --zone=public --add-port=5000-10000/tcp --permanent 
 
重新载入
    firewall-cmd --reload
 
查看
    firewall-cmd --zone=public --query-port=80/tcp
 
删除
    firewall-cmd --zone=public --remove-port=80/tcp --permanent
```

端口查看
```text
netstat -tunlp
netstat -tunlp | grep xxx
```

<a name="related-https"></a>
### 4、https

对称加密算法
```text
DES：64位加密
AES：
3DES：
RC4：
```

非对称加密算法
```text
https使用的非对称加密
RSA：
SM2：
```

```text
证书  CRT
私钥  1024 2048
使用ssh，需要配置证书
linux使用openssl生成证书
 
server {
    listen       443 ssl;
    server_name  localhost;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    #证书文件
    ssl_certificate         nginx.crt;
    
    #私钥key
    ssl_certificate_key     nginx.key;

    #session缓存
    #ssl_session_cache off|none builtin:size    shared:name:size
    #builtin:size算法         用openssl构建的缓存   
    #shared:name:size算法     work_process共享缓存
    ssl_session_cache       shared:SSL:1m;  #1m大概存放4000个session
    
    #session过期时长
    ssl_session_timeout     5m;

    ssl_ciphers     HIGH:!aNULL:!MD5;
    
    #客户端和服务器端协商加密算法，由服务器端加密算法优先
    ssl_prefer_server_ciphers       on;

    location / {
        root   html;
        index  index.html index.htm;
    
        proxy_next_upstream http_502 http_504 error timeout invalid_header;
        proxy_set_header    Host    $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass  http://test;   
    }
}

证书生成：
openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout nginx.key -out nginx.crt
```

<a name="related-zookeeper"></a>
### 5、zookeeper

CAP理论
```text
1、一致性（Consistency）
2、可用性（Availability）
3、分区容错性（Partition tolerance）

CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。

```

BASE理论
```text
我们无法做到强一致，但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性。
1、基本可用（BA, Basically Available）
    指在分布式系统中，发生故障时，允许损失一部分功能，但是其它功能还是可用的。
2、软状态（Soft State）
    指分布式系统中，允许存在数据的中间状态，而中间状态又不会影响系统的可用性。
3、最终一致性（Eventual Consistency）
    分布式系统中的数据在一段时间后，最终能达到一致性。
```

2PC、3PC
```text
2PC，二阶段提交协议，即将事务的提交过程分为两个阶段来进行处理：准备阶段和提交阶段。事务的发起者称协调者，事务的执行者称参与者。
3PC，三阶段提交协议，是2PC的改进版本，即将事务的提交过程分为CanCommit、PreCommit、do Commit三个阶段来进行处理。
```
zookeeper下载安装与启动停止
```text
#wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
#tar xvzf ./zookeeper-3.4.14.tar.gz
 
conf/xxx.cfg：
#Zookeeper数据存放目录
dataDir=/tmp/zookeeper/server1/data
 
#当前服务器对外服务端口  
clientPort=2181  
 
#Zookeeper日志文件存放目录
dataLogDir=/tmp/zookeeper/server1/log
 
#服务器集群信息
server.1=ip1:数据通信端口1:选举端口1
server.2=ip2:数据通信端口2:选举端口2 
server.3=ip3:数据通信端口3:选举端口3  
 
伪分布使用相同ip，不同端口，不同选举端口
真分布使用不同ip，可以使用相同端口，相同选举端口
 
*myid文件
# 很重要，没有该文件启动不了，存放到dataDir目录下。服务器唯一标识
# echo 1 > myid
  
启动
bin># ./zkServer.sh start /root/dx/zookeeper/zookeeper-3.4.14/conf/zoo1.cfg
 
停止
bin># ./zkServer.sh stop /root/dx/zookeeper/zookeeper-3.4.14/conf/zoo1.cfg
 
重启
bin># ./zkServer.sh restart /root/dx/zookeeper/zookeeper-3.4.14/conf/zoo1.cfg
  
查看状态
bin># ./zkServer.sh status /root/dx/zookeeper/zookeeper-3.4.14/conf/zoo3.cfg
 
jps查看，需要先安装jdk
```
Quorum机制
```text
保证数据一致性,服务器数据更新过半数，则表示更新成功
```
ZK Shell和命令
```text
zookeeper基于Netty的长连接
服务器有三个角色，leader、Follower、Observer
leader：读、写、选举
Follower：读、选举
Observer：读
```
Znode
```text
zookeeper的数据模型，它是这一种树形结构，类似unix文件目录
bin># zkCli.sh
# 列出/下所有节点
[zk: localhost:2181(CONNECTED) 9]> ls /
 
# 创建节点/Tomcat和数据Tomcat（byte数组），不加 -e 表示持久节点。-s 表示顺序自增节点
[zk: localhost:2181(CONNECTED) 9]> create /Tomcat Tomcat
持久节点下存在的时候可创建自增的顺序节点，新创建的节点名会自动更改
 
# 获取节点信息
[zk: localhost:2181(CONNECTED) 9]> get /Tomcat
 
# 获取节点状态
[zk: localhost:2181(CONNECTED) 9]> stat /Tomcat
 
# 修改节点
[zk: localhost:2181(CONNECTED) 9]> set /Tomcat hello
  
属性值：
czxid       long    节点被创建的zxid值
mzxid       long    节点被修改的zxid值
pzxid       long    子节点最后一次被修改时的事务id
ctime       long    节点被创建的时间
mtime       long    节点最后一次被修改的时间
versoin     long    节点被修改的版本号
cversion    long    节点的所拥有子节点被修改的版本号
aversion    long    节点的ACL被修改的版本号
emphemeralOwner long    如果此节点为临时节点，那么它的值为个节点拥有者的会话ID，否则值为0
dataLength  int     节点数据长度
numChildren int     节点拥有的子节点长度
 
zxid，全局唯一
 
CREATE(创建)  create 
DELETE(删除)  delete  rmr
READ(只读)    get ls ls2
WRITE(只写)   set
 
节点类型：
持久节点（顺序）
临时节点（顺序）
 
```
**注意：**
- 同一个节点下不能存放同名node
- 临时节点下不能再创建节点
 
watch
```text
stat path [watch|.]
ls path [watch|.]
ls2 path [watch|.]
get path [watch|.]
 
watch只生效一次。
```

***原生zkjavaApi不支持级联创建znode***

acl（Access control level）
```text
节点访问6种操作：
①Read（1，读）
②Write（2，写）
③Create（4，创建）
④Delete（8，删除）
⑤Admin（16，节点管理）
⑥all（0，全部操作）
 
获取acl
权限模式（Schema）、授权对象（ID）、权限（Permission） 
[zk: localhost:2181(CONNECTED) 10] getAcl /
 'world,'anyone
 : cdrwa
  
4种权限模式：
①world：开放模式，所有人可以访问
    create /test test world:anyone:cdrwa
②ip：针对ip开放
    create /test test ip:192.168.1.1:cdrwa
③digest：用户密码模式(常用)
    create /test test digest:username:password(账号密码进行过两次加密):cdwra
    设置了用户名和密码需要添加权限用户才能访问这个/test
    addauth digest username:password
    digest会进行两次加密
    Base64（sha1（username:password））
④super：超级用户模式
    create /test test 
    
修改权限：
setAcl path acl   
      
使用zookeeper的javaApi加密用户名和密码：
[root@localhost zookeeper-3.4.14]# java -cp ./zookeeper-3.4.14.jar:./lib/log4j-1.2.17.jar:./lib/slf4j-api-1.7.25.jar:./lib/slf4j-log4j12-1.7.25.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider test:123456
     
```

一些问题：
- 最少两个节点可对外服务
- 4台机器挂掉2台就不能对外工作

一致性解决方案：
- 2PC/3PC
- Paxos
- Raft

Paxos协议
```text
多数派、过半协议。
三个角色：
①Proposer：提交者
②Acceptor：接收者
③Learner：学习者
 
条件设定：
议案必须有编号。不重复增长
议案两种（提交议案，批准议案）
如果Acceptor没有接受议案，那么必须接受第一个议案
接受编号大的议案，如果小于之前接受议案号，那么不接受 
```

Paxosx详解
```text
<K,V> 议案编号-》议案内容
议员处理议案：ok-接受、error|reject-不接受|驳回
 
几个阶段：
①Prepare阶段（提交阶段，只提交议案编号，不提交内容）
②Accept阶段（批准阶段,已经包含了k，v值）
    Proposer提交议案，Acceptor没有处理过Prepare阶段则直接返回ok。
Acceptor处理过Prepare阶段，判断提交的议案编号和之前收到的议案编号比较，小于的话则不接受|驳回
否则下一步，没有批准过的议案则直接返回ok，如果有批准过的议案，则回复ok，和议案编号，内容给Proposer。
判断回复ok的是否过半，没有的话则重新Prepare阶段。否则下一步，判断是否全部回复
没有附带批准过的议案，是，则Proposer直接议案编号和内容，否则搜集回复中最大的议案编号和内容，发起请求，
Acceptor接收到的编号大于已有最大议案编号，则批准该议案，否则不接受|驳回。最后判断是否回复过半，没有则Prepare阶段，否则下一步
通知Learner开始学习议案
```

ZXID
```text
64位数据结构，分为高32位和低32位（是由leader产生的，而且是自增有序的）
高32位用来存储epoch（年号。选举用）
低32位用来存储事务编号
 
有两种情况导致高32位变化：
①低32位事务编号满了
②Leader选举
```

ZAB协议
```text
两种基本模式：
①崩溃恢复
②消息广播
 
三个阶段：
①发现（leader选举）
②同步（数据同步）
③广播（正式接受请求）
其中发现阶段等同于Paxos的 读阶段，广播阶段等同于Paxos的写阶段 
 
数据同步：
①直接差异化同步
    有follower挂掉后，leader新更新了数据，这时挂掉的follower又加入了集群，数据是没有leader更新的那部分，
 leader直接发送给follower更新有差异的数据，同步数据   
②先回滚再差异化同步
    原leader新写入事务后只在本地处理还没有发送到follower就挂掉了，此时follower选举了新的leader，并且
新写入了事务，原leader恢复后加入集群，会多出它自己写入的事务，需要将这些事务删除掉，再进行差异化同步
③仅回滚同步
④全量同步
    全新加入的follower
     
    
```

**格式化查看日志：**
[root@localhost zookeeper-3.4.14]# java -cp ./zookeeper-3.4.14.jar:./lib/log4j-1.2.17.jar:./lib/slf4j-api-1.7.25.jar:./lib/slf4j-log4j12-1.7.25.jar org.apache.zookeeper.server.LogFormatter /usr/local/dx/zookeeper/server1/log/version-2/log.400000001
 
**zk日志命名**：按照第一个写入的事务编号命名，初始化大小为4k，超过4k直接到64M，多余的用0填充

chroot命名空间
```text
业务隔离
 
[root@localhost bin]#./zkCli.sh -timeout 5000 -server localhost:2181/Tomcat
 
如上，只会连接到localhost:2181/Tomcat上
```

**StaticHostProvider类**：zkApi自带的zk地址轮询

zkSession会话
```text
五个状态
connecting正在连接
connected已经连接
reconnecting重新连接
reconnected重新连接上
close关闭
 
zkApi中
initializeNextSession()初始化session
 
分桶机制
处理过期session，使用两个桶存储session，到一定时间点，将第一个桶内还存活的session复制到下个桶中，
然后将第一个桶全部清空。
```

Leader选举：(启动，崩溃恢复，zxid低32位满了)
```text
投票步骤：
每个服务器都有一张投票<myid,zxid>，先投自己
搜集所有服务器投票结果
比较投票，先根据zxid比较出最大的，没有结果则根据myid（唯一）比较出最大的
 
Leader算法涉及类：
FastLeaderElection

org.apche.zookeeper.server.quorum.QuorumPeer
选举节点类，该类用来设置一个报文套接字并响应当前Leader
@Override
public synchronized void start() {
    loadDataBase();
    cnxnFactory.start();        
    startLeaderElection();
    super.start();
}
 
org.apache.zookeeper.server.quorum.QuorumCnxManager
选举通信连接管理类，该类使用TCP实现Leader选举过程中的连接管理
 
final ConcurrentHashMap<Long, SendWorker> senderWorkerMap;
为每个服务器都保存一个发送队列，用于发送消息，SendWorker是Thread子类
 
final ConcurrentHashMap<Long, ArrayBlockingQueue<ByteBuffer>> queueSendMap;
为每个参与投票的服务器保留一个队列
 
final ConcurrentHashMap<Long, ByteBuffer> lastMessageSent;
保存从其他节点发送的最后一条消息
 
public final ArrayBlockingQueue<Message> recvQueue;
接受消息队列
   
Notification
接受其他服务器发来的选举投票信息
 
ToSend
发送给其他服务器的选举投票信息
 
Messenger.WorkerReceiver
线程类，不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成选票信息，保存在recvQueue中
  
Messenger.WorkerSender
线程类，不断地从sendqueue中获取待发送的选票，并把选票交给QuorumCnxManager发送出去
   
   
    
org.apache.zookeeper.server.quorum.FastLeaderElection
具体选举算法
 
选举算法：
服务器调用QuorumPeer.start()方法，在崩溃恢复过程中要进行Leader选举
 
Leader选举方法startLeaderElection()被调用，在方法中需要确定一个选举方法
 

```

zkClient封装了原生zkapi，使用更简洁方便
```text
原生zkzpi缺点：
会话链接异步
序列化支持不透明
watch需要重复注册
session重连机制
开发复杂性较高
 
 zkClient个人编写的api，做了一些封装
<!-- https://mvnrepository.com/artifact/com.101tec/zkclient -->
<dependency>
 <groupId>com.101tec</groupId>
 <artifactId>zkclient</artifactId>
 <version>0.11</version>
</dependency>
```
 
Curator
```text
链式调用风格
watch设置后不需要反复注册
session超时重连
递归节点创建
多种分布式应用场景Recipe
 
监听机制
NodeCache类
监听节点数据变化
PathChildrenCache类
监听子节点变化
TreeCache类
上面两种的结合
<!-- https://mvnrepository.com/artifact/org.apache.curator/curator-recipes -->
<!-- 所有典型应用场景。例如：分布式锁,Master选举等。需要依赖client和framework，需设置自动获取依赖。 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.2.0</version>
</dependency>
 
<!-- https://mvnrepository.com/artifact/org.apache.curator/curator-framework -->
<!-- Zookeeper API的高层封装，大大简化Zookeeper客户端编程，添加了例如Zookeeper连接管理、重试机制等。 -->
<dependency>
 <groupId>org.apache.curator</groupId>
 <artifactId>curator-framework</artifactId>
 <version>4.2.0</version>
</dependency>
 
git地址：https://github.com/apache/curator 整个框架由下面这些模块组成：
curator-recipes	所有典型应用场景。例如：分布式锁,Master选举等。需要依赖client和framework，需设置自动获取依赖。
curator-async	jdk8 异步操作
curator-framework	Zookeeper API的高层封装，大大简化Zookeeper客户端编程，添加了例如Zookeeper连接管理、重试机制等。
curator-client	Zookeeper client的封装，用于取代原生的Zookeeper客户端（ZooKeeper类），提供一些非常有用的客户端特性。
curator-test	包含TestingServer，TestingCluster和一些其他有助于测试的工具。
curator-examples	各种使用Curator特性的例子。
curator-x-discovery	服务注册发现，在SOA /分布式系统中，服务需要相互寻找。curator-x-discovery提供了服务注册，找到特定服务的单个实例，和通知服务实例何时更改。
curator-x-discovery-server	服务注册发现管理器,可以和curator-x-discovery 或者非java程序程序使用RESTful Web服务以注册，删除，查询等服务。
 
一般依赖recipes就可以了，内部依赖有client和framework 能用的上的功能都有了
``` 
 
Curator创建
```java
//CuratorFrameworkFactory创建，可链式调用创建连接
//public static CuratorFramework newClient(String connectString, RetryPolicy retryPolicy);
//public static CuratorFramework newClient(String connectString, int sessionTimeoutMs, int connectionTimeoutMs, RetryPolicy retryPolicy);
  
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
 
public class Test {
    public static void main(String[] args) {
        String connectionStr = "192.168.75.133";
        // 重试策略
        ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3, Integer.MAX_VALUE);
 
        CuratorFramework curatorFramework = CuratorFrameworkFactory.newClient(connectionStr, retry);
        curatorFramework.start();
 
        CuratorFramework curatorFramework2 = CuratorFrameworkFactory.newClient(connectionStr, 60 * 1000, 15 * 1000, retry);
        curatorFramework2.start();
 
        CuratorFramework curatorFramework3 = CuratorFrameworkFactory.builder().connectString(connectionStr)
                .sessionTimeoutMs(60 * 1000)
                .connectionTimeoutMs(15 * 1000)
                .namespace("Tomcat")
                .retryPolicy(retry)
                .build();
        curatorFramework3.start();
    }
}
``` 
Curator重试策略
- ExponentialBackoffRetry
```text
/**
 * 随着重试次数增加重试时间间隔变大,指数倍增长baseSleepTimeMs * Math.max(1, random.nextInt(1 << (retryCount + 1)))。
 * 有两个构造方法
 * baseSleepTimeMs初始sleep时间
 * maxRetries最多重试几次
 * maxSleepMs最大的重试时间
 * 如果在最大重试次数内,根据公式计算出的睡眠时间超过了maxSleepMs，将打印warn级别日志,并使用最大sleep时间。
 * 如果不指定maxSleepMs，那么maxSleepMs的默认值为Integer.MAX_VALUE。
 */
public ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries);
public ExponentialBackoffRetry(int baseSleepTimeMs, int maxRetries, int maxSleepMs);
```
- BoundedExponentialBackoffRetry
```text
/**
 * 继承与ExponentialBackoffRetry ，BoundedExponentialBackoffRetry只有一个三个参数构造器，效果跟ExponentialBackoffRetry三个函数构造器是一样的,只是内部实现不一样。
 * baseSleepTimeMs初始sleep时间
 * maxSleepTimeMs最大sleep时间
 * maxRetries最大重试次数
 */
public BoundedExponentialBackoffRetry(int baseSleepTimeMs, int maxSleepTimeMs, int maxRetries);
```
- RetryForever
```text
/**
 * RetryForever：永远重试策略
 * retryIntervalMs重试时间间隔
 */
public RetryForever(int retryIntervalMs);
```
- RetryNTimes
```text
/**
 * RetryNTimes：重试N次
 * n重试几次
 * sleepMsBetweenRetries每次重试间隔时间
 */
public RetryNTimes(int n, int sleepMsBetweenRetries);
```
- RetryOneTime
```text
/**
 * 这个类继承的RetryNTimes，其实是调用的父类方法
 * RetryOneTime：只重试一次
 * sleepMsBetweenRetry:每次重试间隔的时间
 */
public RetryOneTime(int sleepMsBetweenRetry);
```
- RetryUntilElapsed
```text
/**
 * RetryUntilElapsed：一直重试，直到超过指定时间
 * maxElapsedTimeMs最大重试时间
 * sleepMsBetweenRetries每次重试间隔时间
 */
public RetryUntilElapsed(int maxElapsedTimeMs, int sleepMsBetweenRetries);
```
- SleepingRetry抽象类

Curator测试基本的增删改查(测试linux开放防火墙端口)
```java
package tool;
 
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.Stat;
 
import java.util.List;
import java.util.concurrent.TimeUnit;
 
/**
 <dependency>
 <groupId>org.apache.curator</groupId>
 <artifactId>curator-recipes</artifactId>
 <version>4.2.0</version>
 </dependency>
 */
public class Test {
    // 伪分布式集群
    public static final String connectionString = "192.168.75.133:2181,192.168.75.133:2182,192.168.75.133:2183";
 
    public static void main(String[] args) throws Exception {
//            t1(); // 增加节点
//            t2();   // 修改节点
//            t3();   // 删除节点
//            t4();   // 查看节点信息
//            t5();   // 获取某个节点下的子节点列表
    }
 
    private static void t5() throws Exception {
        CuratorFramework client = getClient();
 
        String nodePath = "/";          // 获取命名空间test下的所有子节点
 
        List<String> list = client.getChildren().forPath(nodePath);
        System.out.println(client.getNamespace() + nodePath + " 节点下的子节点列表：");
        for (String childNode : list) {
            System.out.println(childNode);
        }
 
        // 节点路径
        String nodePath2 = "/test2";
        // 查询某个节点是否存在，存在就会返回该节点的状态信息，如果不存在的话则返回空
        Stat statExist = client.checkExists().forPath(nodePath2);
        System.out.println(statExist == null ? nodePath2 + " 节点不存在" : nodePath2 + " 节点存在");
 
        // 节点路径
        String nodePath3 = "/t2";
        Stat statExist2 = client.checkExists().forPath(nodePath3);
        System.out.println(statExist2 == null ? nodePath3 + " 节点不存在" : nodePath3 + " 节点存在");
 
        TimeUnit.SECONDS.sleep(1);
        clientClose(client);
    }
 
    private static void t4() throws Exception {
        CuratorFramework client = getClient();
 
        String nodePath = "/testNode";          // 节点路径
 
        Stat stat = new Stat();
        byte[] bytes = client.getData()
                .storingStatIn(stat)
                .forPath(nodePath);
 
        System.out.println("节点" + nodePath + " 的数据为：" + new String(bytes));
        System.out.println("版本号为：" + stat.getVersion());
 
        TimeUnit.SECONDS.sleep(1);
 
        clientClose(client);
    }
 
    private static void t3() throws Exception {
        CuratorFramework client = getClient();
 
        String nodePath = "/testNode";          // 节点路径
 
        // 删除节点
        client.delete()
                .guaranteed()   //如果删除失败，那么在后端还是会继续删除，直到成功
                .deletingChildrenIfNeeded() // 子节点也会进行递归删除
                .withVersion(1) // 指定数据版本
                .forPath(nodePath);
 
        TimeUnit.SECONDS.sleep(1);
 
        clientClose(client);
    }
 
    private static void t2() throws Exception {
        CuratorFramework client = getClient();
 
        String nodePath = "/testNode";          // 节点路径
        byte[] newData  = "modify data".getBytes();   // 修改数据
 
        // 修改节点数据
        Stat resultStat = client.setData().withVersion(0)         // 指定数据版本
                .forPath(nodePath, newData);
 
        System.out.println("更新节点数据成功，新的数据版本为：" + resultStat.getVersion());
 
        TimeUnit.SECONDS.sleep(1);
 
        clientClose(client);
    }
 
    private static void t1() throws Exception {
        CuratorFramework client = getClient();
 
        // 创建节点
        String nodePath = "/testNode";          // 节点路径
        byte[] data = "test data".getBytes();   // 节点数据
        String result = client.create()
                .creatingParentsIfNeeded()  // 创建父节点，也就是会递归创建
                .withMode(CreateMode.PERSISTENT)  // 节点类型，持久型(临时节点的话，/testNode会删除，命名空间还在)
                .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)  // 节点的acl权限
                .forPath(nodePath, data);
 
        System.out.println(result + "节点，创建成功...");
 
        Thread.sleep(1000);
        clientClose(client);
    }
 
    private static CuratorFramework getClient() {
        // 指数退避重试机制
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3,Integer.MAX_VALUE);
 
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connectionString)
                .sessionTimeoutMs(10000)
                .retryPolicy(retryPolicy)
                .namespace("test")
                .build();
 
        client.start();
        System.out.println("链接状态：" + client.getState());
 
        return client;
    }
 
    private static void clientClose(CuratorFramework client) {
        if (client != null){
            client.close();
            System.out.println("链接状态：" + client.getState());
        }
    }
}
```
Curator测试watch(热加载配置文件)
```java
package tool;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.CuratorWatcher;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.retry.RetryOneTime;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;

import java.util.List;
import java.util.concurrent.CountDownLatch;

/**
 <dependency>
 <groupId>org.apache.curator</groupId>
 <artifactId>curator-recipes</artifactId>
 <version>4.2.0</version>
 </dependency>

 zk能够watch的有4个操作

 stat path [watch]
 ls path [watch]
 ls2 path [watch]
 get path [watch]
 */
public class Test {
    // 伪分布式集群
    public static final String connectionString = "192.168.75.133:2181,192.168.75.133:2182,192.168.75.133:2183";

    public static void main(String[] args) throws Exception {
//        t1();   // watcher
//        t2();   // nodeCache一次注册N次监听
//        t3();   // PathChildrenCache子节点监听
        t4();   // 监听配置文件更新
    }

    private static void t4() throws Exception {
        CountDownLatch countDown = new CountDownLatch(1);  // 计数器

        RetryPolicy retryPolicy = new RetryOneTime(3000);
        String connetStr = "192.168.75.133:2183";   // 监听2183端口
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connetStr)
                .sessionTimeoutMs(10 * 1000)    // 默认是60 * 1000
                .connectionTimeoutMs(10 * 1000) // 默认是15 * 1000
                .retryPolicy(retryPolicy)
                .namespace("test")
                .build();
        client.start();

        PathChildrenCache pathChildrenCache = new PathChildrenCache(client, "/", true);
        pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);

        pathChildrenCache.getListenable().addListener((client1, event) -> {
            switch (event.getType()){
                // 监听配置文件的更新
                case CHILD_UPDATED:
                    // 只监听配置文件的节点
                    String configNodePath = event.getData().getPath();
                    System.out.println("节点数据更新，路径为:" + configNodePath);
                    if (configNodePath.equals("/t2")) {
                        System.out.println("该路径节点为配置文件节点");
                        String data = new String(event.getData().getData());
                        System.out.println("节点的数据为: " + data);
                        // 获取到更新数据后，可以进行热加载配置文件
                        countDown.countDown();
                    }
                    break;
            }
        });

        // 倒计数，等待countDown.countDown()
        countDown.await();
        client.close();
    }

    /**
     * 监听某一个节点的子节点的事件，或者监听某一个特定节点的增删改事件都需要借助PathChildrenCache来实现
     */
    private static void t3() throws Exception {
        CuratorFramework client = getClient();
        String nodePathDate = "/t2";          // 节点路径

        // 初始化节点数据
        byte[] init = "hi".getBytes();
        client.setData()
                .forPath(nodePathDate, init);

        Thread.sleep(3000);

        // 监听该路径为父路径
        String nodePath = "/";          // 节点路径，监听的命名空间test下的节点

        // 为子节点添加watcher
        // PathChildrenCache: 监听数据节点的增删改，可以设置触发的事件
        PathChildrenCache childrenCache = new PathChildrenCache(client, nodePath, true);

        /**
         * 可选参数StartMode: 初始化方式
         * POST_INITIALIZED_EVENT：异步初始化，初始化之后会触发事件，会在初始化时触发子节点初始化以及添加子节点这两个事件
         *                          因为缓存初始化时是把子节点添加到缓存里，所以会触发添加子节点事件，而添加完成之后，就会触发子节点初始化完成事件。
         * NORMAL：异步初始化，无法获取子节点列表，并且也会触发添加子节点事件，但是不会触发子节点初始化完成事件
         * BUILD_INITIAL_CACHE：同步初始化
         */
        childrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
//        childrenCache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);
//        childrenCache.start(PathChildrenCache.StartMode.NORMAL);

        // 列出子节点数据列表，需要使用BUILD_INITIAL_CACHE同步初始化模式才能获得，异步是获取不到的
        List<ChildData> currentData = childrenCache.getCurrentData();
        System.out.println("当前节点的子节点详细数据列表：");
        for (ChildData currentDatum : currentData) {
            System.out.println("\t* 子节点路径：" + new String(currentDatum.getPath()) + "，该节点的数据为：" + new String(currentDatum.getData()));
        }

        // 添加事件监听器
        childrenCache.getListenable().addListener((client1, event) -> {
            // 通过判断event type的方式来实现不同事件的触发
            switch (event.getType()){
                // 子节点初始化出发
                case INITIALIZED :
                    System.out.println("子节点初始化成功");
                    break;
                // 添加子节点时触发
                case CHILD_ADDED :
                    System.out.println("子节点：" + event.getData().getPath() + " 添加成功");
                    break;
                // 删除子节点时触发
                case CHILD_REMOVED :
                    System.out.println("子节点：" + event.getData().getPath() + " 删除成功");
                    break;
                // 修改子节点数据时触发
                case CHILD_UPDATED :
                    System.out.println("子节点：" + event.getData().getPath() + " 修改成功");
                    break;
            }
        });

        if(childrenCache.getCurrentData() != null) {
            String nodePath_t2 = "/t2";          // 修改t2的数据

            // 修改数据
            byte[] bytes = "modify---1".getBytes();
            byte[] bytes2 = "modify---2".getBytes();
            client.setData()
                    .forPath(nodePath_t2, bytes);

            Thread.sleep(1000);
            client.setData()
                    .forPath(nodePath_t2, bytes2);
        }

        // 等待60秒操作时间
        Thread.sleep(1000 * 60);
        clientClose(client);
    }

    /**
     * 只能监听该nodeChanged时间，node的创建、删除、以及子节点的时间都无法监听
     * @throws Exception
     */
    private static void t2() throws Exception {

        CuratorFramework client = getClient();
        String nodePath = "/t2";          // 节点路径

        // 初始化节点数据
        byte[] init = "hi".getBytes();
        client.setData()
                .forPath(nodePath, init);

        // NodeCache: 缓存节点，并且可以监听数据节点的变更，会触发事件
        NodeCache nodeCache = new NodeCache(client, nodePath);

        // 参数buildInitial: 初始化的时候获取node的值并且缓存，不设置则false
        nodeCache.start(true);
        System.out.println(nodeCache.getCurrentData() == null ? "节点初始化数据为空..." : "节点初始化数据为：" + new String(nodeCache.getCurrentData().getData()));

        // 为缓存的节点添加watcher，或者说添加监听器
        // 节点数据change事件的通知方法
        nodeCache.getListenable().addListener(() -> {
            // 防止节点被删除时发生错误
            if (nodeCache.getCurrentData() == null) {
                System.out.println("获取节点数据异常，无法获取当前缓存的节点数据，可能该节点已被删除");
                return;
            }

            // 获取节点最新的数据
            String data = new String(nodeCache.getCurrentData().getData());
            System.out.println(nodeCache.getCurrentData().getPath() + " 节点的数据发生变化，最新的数据为：" + data);
        });

        if(nodeCache.getCurrentData() != null) {
            // 修改数据
            byte[] bytes = "modify1".getBytes();
            byte[] bytes2 = "modify2".getBytes();
            client.setData()
                    .forPath(nodePath, bytes);

            Thread.sleep(3000);
            client.setData()
                    .forPath(nodePath, bytes2);
        }

        Thread.sleep(3000);
        clientClose(client);
    }

    /**
     * 测试watch
     * 一次性
     * @throws Exception
     */
    private static void t1() throws Exception {
        CuratorFramework client = getClient();
        String nodePath = "/t2";          // 节点路径

        // 添加 watcher 事件，当使用usingWatcher的时候，监听只会触发一次，监听完毕后就销毁
        client.getData().usingWatcher(new MyWatcher()).forPath(nodePath);
        client.getData().usingWatcher(new MyCuratorWatcher()).forPath(nodePath);

        // 修改数据
        byte[] bytes = "modify1".getBytes();
        byte[] bytes2 = "modify2".getBytes();
        client.setData()
                .forPath(nodePath, bytes);

        // 第二次修改(此时不会再监听)
        client.setData()
                .forPath(nodePath, bytes2);

        Thread.sleep(1000);
        clientClose(client);
    }

    private static CuratorFramework getClient() {
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3,Integer.MAX_VALUE);

        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connectionString)
                .sessionTimeoutMs(10000)
                .retryPolicy(retryPolicy)
                .namespace("test")
                .build();

        client.start();
        System.out.println("链接状态：" + client.getState());

        return client;
    }

    private static void clientClose(CuratorFramework client) {
        if (client != null){
            client.close();
            System.out.println("链接状态：" + client.getState());
        }
    }

    /**
     * Watcher原生zkAPI，一次性
     * 抽象方法，需要子类实现
     */
    static class MyWatcher implements Watcher{

        // Watcher事件通知方法
        @Override
        public void process(WatchedEvent event) {
            System.out.println("触发原生zkAPIwatcher，节点路径为：" + event.getPath());
        }
    }

    /**
     * Curator提供的CuratorWatcher接口实现，也是一次性的
     * 需要子类实现
     */
    static class MyCuratorWatcher implements CuratorWatcher{

        @Override
        public void process(WatchedEvent event) throws Exception {
            System.out.println("触发CuratorAPIwatcher，节点路径为：" + event.getPath());
        }
    }
}
```

Curator进行acl权限操作
```java
package tool;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;
import org.apache.zookeeper.server.auth.DigestAuthenticationProvider;

import java.util.ArrayList;
import java.util.List;


public class Test {
    // 伪分布式集群
    public static final String connectionString = "192.168.75.133:2181,192.168.75.133:2182,192.168.75.133:2183";

    public static void main(String[] args) throws Exception {
        // 注意，需要在客户端链接的时候设置acl
        // authorization("digest", "dx:123a".getBytes())
//        t1(); // 添加权限
        t2();   // 修改权限
    }

    private static void t2() throws Exception {
        CuratorFramework client = getClient();
        String nodePath = "/testNode";          // 节点路径

        // 自定义权限列表
        List<ACL> acls = new ArrayList<>();

        Id user1 = new Id("world", "anyone");
        acls.add(new ACL(ZooDefs.Perms.ALL, user1));    // 给user1 cdwra所有权限

        client.setACL().withACL(acls).forPath(nodePath);

        Thread.sleep(1000);
        clientClose(client);
    }

    private static void t1() throws Exception {
        CuratorFramework client = getClient();
        String nodePath = "/testNode";          // 节点路径

        // 自定义权限列表
        List<ACL> acls = new ArrayList<>();

        Id user1 = new Id("digest", DigestAuthenticationProvider.generateDigest("dx:123a"));
        acls.add(new ACL(ZooDefs.Perms.ALL, user1));    // 给user1 cdwra所有权限

        // 创建节点，使用自定义权限列表来设置节点的acl权限
        // 如果想要在递归创建节点时，父节点和子节点的acl权限都是我们自定义的权限，那么就需要在withACL方法中，传递一个true
        byte[] nodeData = "child-data".getBytes();
        client.create()
                .creatingParentsIfNeeded()
                .withMode(CreateMode.PERSISTENT)
                .withACL(acls, true)
                .forPath(nodePath, nodeData);

        Thread.sleep(1000);
        clientClose(client);
    }

    private static CuratorFramework getClient() {
        ExponentialBackoffRetry retryPolicy = new ExponentialBackoffRetry(1000, 3,Integer.MAX_VALUE);

        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString(connectionString)
                .authorization("digest", "dx:123a".getBytes())
                .sessionTimeoutMs(10000)
                .retryPolicy(retryPolicy)
                .namespace("test")
                .build();

        client.start();
        System.out.println("链接状态：" + client.getState());

        return client;
    }

    private static void clientClose(CuratorFramework client) {
        if (client != null){
            client.close();
            System.out.println("链接状态：" + client.getState());
        }
    }
}
```

zk分布式锁

<a name="related-redis"></a>
### 6、redis

[免安装使用](http://try.redis.io/)
 
安装redis[中文官网](http://redis.cn)
- docker方式
```text
拉取redis镜像
>docker pull redis
 
运行redis容器，将容器的6379端口映射到主机6379端口
>docker run --name myredis -d -p 6379:6379 redis
 
执行容器中的redis-cli，可以直接使用命令行操作redis
>docker exec -it myredis redis-cli
```

- git方式
```text
> git clone --branch 3.2 --depth 1 git@github.com:antirez/redis.git
cd redis
 
编译
> make 
> cd src
 
运行服务器，daemonize表示在后台运行
>./redis-server --daemonize yes
 
运行命令行
>./redis-cli   
```

- 直接安装
```text
linux
$ wget http://download.redis.io/releases/redis-3.2.0tar.gz
$ tar xvzf redis-2.8.17.tar.gz
$ cd redis-2.8.17
$ make

make完后 redis-2.8.17目录下会出现编译后的redis服务程序redis-server,还有用于测试的客户端程序redis-cli,两个程序位于安装目录 src 目录下：
 
下面启动redis服务.
 
$ cd src
$ ./redis-server
注意这种方式启动redis 使用的是默认配置。也可以通过启动参数告诉redis使用指定配置文件使用下面命令启动。
 
$ cd src
$ ./redis-server ../redis.conf
redis.conf 是一个默认的配置文件。我们可以根据需要使用自己的配置文件。
 
启动redis服务进程后，就可以使用测试客户端程序redis-cli和redis服务交互了。 比如：
 
$ cd src
$ ./redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```
五种对象类型
- string（字符串）
- list（双端列表）
- set（集合）
- hash（哈希）
- sorted set（有序集合）

几种类型对应的字符编码
```text
RedisObject
 
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;
4位的type表示具体的数据类型。Redis中共有5中数据类型。2^4 = 8足以表示这些类型。
4位的encoding表示该类型的物理编码方式，同一种数据类型可能有不同的编码方式。目前Redis中主要有8种编码方式：
lru字段表示当内存超限时采用LRU算法清除内存中的对象。 
refcount表示对象的引用计数。 
ptr指针指向真正的存储结构。
 
#define REDIS_ENCODING_RAW 0     /* Raw representation */
#define REDIS_ENCODING_INT 1     /* Encoded as integer */
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */ 
 
一共16个字节、128比特、128位 
type(4bit、4个比特、4位) + encoding(4bit、4个比特、4位) +  LRU(24bit、24个比特、24位) + refcount(4bytes、32个比特、32位) + ptr(8bytes、64个比特、64位)
  
查看引用次数命令
object refcount key
查看编码命令
object encoding key  
```
 
类型|一般情况|少量数据|特殊情况
----|:----:|:----:|----
String|raw|embstr|int
List|quicklist(>=3.2.0版本,只有),Linkedlist(<3.2.0版本)|ziplist|
Set|ht||intset
hash|ht|ziplist|
sorted set|skiplist|ziplist|
 
Redis的embstr编码方式和raw编码方式在3.0版本之前是以39字节为分界的，也就是说，
如果一个字符串值的长度小于等于39字节，则按照embstr进行编码，否则按照raw进行编码。 
而在3.2版本之后，则变成了44字节为分界。
```text
why?
embstr：redisobject和sds是连续分配空间，只分配一次。
Redis 默认的内存分配器jemalloc分配内存大小的单位是$2^n$次方,为了容纳一个完整的embstr
对象,最少会分配 32 字节的空间,再长些就是 64 字节,再之后就认为这是一个大字符串不适合用 
embstr 存储,而改用 raw 编码了.
redis在创建对象的时候先分配内
存空间，一个redisobject16个字节，String的sds结构有
len、alloc、flags，各占一个字节，所有就有19个字节，然后C语言数组末尾以\0结尾，占一个
字节，就有20个字节，内存分配以sdshdr5，sdshdr8，sdshdr16，sdshdr32，sdshdr64分配，此
时应该分配32个字节，buf会预分配16个字节空间，所以应分配64个字节，减去之前的20个字节，
剩下44个字节可以存储数据
```

set
```text
可去重，下面是两种编码格式
typedef struct intset {
    // 每个整数的类型 int16、int32、int64
    uint32_t encoding;
    // 数组长度
    uint32_t length;
    // 存储数组
    int8_t contents[];
} intset;
 
当数组内插入的整数型长度变化，数组其他位置的编码类型也会变化成最长的同类型
字符串长度为最大长度 512MB.
 
当插入字符串或者键值对数量超过512个（通过配置文件中的set-max-inset-entries属性改变默认值）自动转行成ht
```

Stirng
```text
struct sds{
    // buf中已占用空间的长度
    unsigned int len;
    // 内存大小
    unsigned int alloc;
    // 特殊标识位
    unsigned char flags;
    // 初始化sds分配的数据空间
    char buf[];
}
字符串在长度小于1M之前，扩容空间采用加倍策略，超过1M之后，为了避免加倍后冗余空间过大
，每次扩容只会多分配1M空间
 
长度小于44的浮点型编码为embstr，可用incrbyfloat key 命令进行加1操作
```

list
```text
3.2之后为quicklist编码，quicklist为多个ziplist组成的双端链表
ziplist：
zlbytes-zltail-zllen-xxxxxxxxxxxxx-zlend
列表占内存总字节数-表尾节点距其实地址的字节-节点数量-entry节点对象内容-标记列表的结尾0Xff
 
list-max-ziplist-size -2
参数的含义解释，取正值时表示quicklist节点ziplist包含的数据项。取负值表示按照占用字节来限定quicklist节点ziplist的长度。
-5: 每个quicklist节点上的ziplist大小不能超过64 Kb。
-4: 每个quicklist节点上的ziplist大小不能超过32 Kb。
-3: 每个quicklist节点上的ziplist大小不能超过16 Kb。
-2: 每个quicklist节点上的ziplist大小不能超过8 Kb。（默认值）
-1: 每个quicklist节点上的ziplist大小不能超过4 Kb。
list设计最容易被访问的是列表两端的数据，中间的访问频率很低，如果符合这个场景，list还有一个配置，可以对中间节点进行压缩（采用的LZF——一种无损压缩算法）
，进一步节省内存。配置如下
 
list-compress-depth 0 
含义：
0: 是个特殊值，表示都不压缩。这是Redis的默认值。
1: 表示quicklist两端各有1个节点不压缩，中间的节点压缩。
2: 表示quicklist两端各有2个节点不压缩，中间的节点压缩。
以此类推
```

hash
```text
//hash节点
//字典，管理两个dicht，为什么需要定义两个，为了在扩容的时候使用。
typedef struct dict {//管理两个dictht，主要用于动态扩容。
    dictType *type;//函数管理，类型特定函数
    void *privdata;//私有数据
    dictht ht[2];//哈希表(一个使用，一个扩容用)
    long rehashidx; /* 扩容标志rehashing not in progress if rehashidx == -1 当rehash不在进行时-1*/
    unsigned long iterators; /* number of iterators currently running目前正在运行的安全迭代器数量 */
} dict;
 
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
//定义一个hash桶，用来管理hashtable
typedef struct dictht {//管理hashtable
    dictEntry **table;//指针数组，这个hash的桶
    unsigned long size;//元素个数
    unsigned long sizemask;//哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long used;//该哈希表已有的节点数量
} dictht;
  
typedef struct dictEntry {//hash节点
    void *key;//键，一半是sds
    //值
    union {//联合体牛逼，导致值可以多用途，可以存储指针或存储整数或浮点数。必定可以存储上面类型之前，通过不同的引用，告诉编译器如何解释对应的内存块。
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;//下一个节点，解决碰撞冲突，形成链表
} dictEntry;
 
//定义hash需要使用的函数
typedef struct dictType {//定义函数指针，例如使用的hash函数
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;//定义对应的函数指针，例如定义hash函数
  
/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {//简单的迭代器封装，STL内部也是这么实现
    dict *d;//字典
    long index;//索引
    int table, safe;
    dictEntry *entry, *nextEntry;//当前元素，下一个元素
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;
 
编码转换：（不可以转换）
当哈希对象同时满足以下两个条件时，哈希对象使用ziplist编码
①哈希对象保存的所有键值元素的长度都小于64字节，可通过修改配置文件中的hash-max-ziplist-value
属性改变默认值
②哈希对象保存的键值对数量小于512个，可通过修改配置文件中的hash-max-ziplist-entries属性改变默认值
 
hashtable采用渐进式rehash(2的指数扩容)，负载因子=已使用节点数量/哈希表大小，在两种情况下对哈希表进行扩展
①当服务器未执行BGSAVE或BGRWRITEAOP，并且负载因子>1
②当服务器执行BGSAVE或BGRWRITEAOP，并且负载因子>5
当负载因子<0.1对哈希表进行收缩
```
sortedset
```text
有序集合对象编码为ziplist或者skiplist
ziplist编码：(分值小到大)
zlbytes-zltail-zllen-成员-分值-成员-分值-zlend
 
skiplist编码：
哈希表+跨越表  
 
redis里面最多32个层级
 
编码转换：
当有序集合对象同时满足以下两个条件时，有序集合对象使用ziplist编码：
①有序集合对象保存的所有成员长度小于64字节，可通过修改配置文件中的zset-max-ziplist-value属性改变默认值
②有序集合对象保存的所有元素数量小于128个，可通过修改配置文件中的zset-max-ziplist-entries属性改变默认值
```

redis持久化
- RDB持久化
- AOF持久化
 
持久化自动加载rdb文件，开启了aof，则加载aof文件否则加载rdb文件 
 
redis可以做内存数据库，默认有16个数据库（redis是单进程的）
```text
RDB
生成经过压缩的二进制RDB文件，通过save和bgsave命令生成，通过快照保存在硬盘上
save：
    阻塞式保存数据，执行save命令会调用rdbsave
     
bgsave：
    fork出一个子进程，子进程负责调用rdbsave，保存完成后通知主进程。bgsave非阻塞
    执行期间，客户端发送save命令会被拒绝，防止竞争条件。两个bgsave不能同时执行 
         
save命令执行rdb.c/rdbSave函数，bgsave命令执行rdb.c/rdbSaveBackgroud函数

修改当前进程配置文件：
config set save "20 1"
 
conf配置文件：
################################ SNAPSHOTTING  ################################
# 快照
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behaviour will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#
#   save ""
# save <seconds> <changes> 文件保存条件，满足其中任何一个就会自动触发
 
save 900 1
save 300 10
save 60 10000
 
# rdb文件名 
dbfilename dump.rdb
 
# rdb文件存放路径 
dir ./
 
# 是否在导出.rdb数据库文件的时候采用LZF压缩 
rdbcompression yes 
 
# 是否开启CRC64校验
rdbchecksum yes
 
# 设置在保存快照出错时，是否停止redis命令的写入，默认开启
# fork子进程内存不足，或者rdb文件目录写入无权限会失败
stop-writes-on-bgsave-error yes  
   
# By default Redis will stop accepting writes if RDB snapshots are enabled   
 
四种持久化文件触发方式：
①使用save相关配置
②从节点执行全量复制操作
③debug reload命令
④shutdown命令，如果没有开启aof自动执行bgsave
 
查看持久化信息：
>info persistence
 
       
```

rdb文件结构
 
redis-db版本号-服务器版本-系统位数-rdb创建时间-可用内存-db0-过期时间-type-key-value-...-db1...-eof-checknum
元数据：服务器版本-系统位数-rdb创建时间-可用内存
数据库键值对db0-过期时间-type-key-value-...-db1...
 
AOF持久化
```text
通过保存redis服务器所执行的写命令来记录数据库状态，也就是每当redis执行一个改变数据集的命令时比如set，
这个命令就会被追加到AOF文件的末尾
 
开启AOF：
修改conf配置文件，默认是appendonly no(关闭) 
appendonly yes
 
执行流程：
命令追加：将redis写命令追加到缓冲区aof_buf
文件写入write和文件同步sync：根据不同的同步策略将aof_buf中的内容同步到硬盘
文件重写：定期重写aof文件，达到压缩目的
文件载入：用于服务器重启时的数据恢复 
 
############################## APPEND ONLY MODE ###############################
 
# By default Redis asynchronously dumps the dataset on disk. This mode is
# good enough in many applications, but an issue with the Redis process or
# a power outage may result into a few minutes of writes lost (depending on
# the configured save points).
#
# The Append Only File is an alternative persistence mode that provides
# much better durability. For instance using the default data fsync policy
# (see later in the config file) Redis can lose just one second of writes in a
# dramatic event like a server power outage, or a single write if something
# wrong with the Redis process itself happens, but the operating system is
# still running correctly.
#
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
#
# Please check http://redis.io/topics/persistence for more information.
 
# 是否开启aof 
appendonly no
 
# aof文件名称 
appendfilename "appendonly.aof"
  
# 写入aof文件的三种方式
# appendfsync always
appendfsync everysec
# 将缓冲区数据写入到aof文件，但是同步操作需要操作系统处理
# appendfsync no    
 
# 重写aof时，是否继续写aof文件 
no-appendfsync-on-rewrite no
 
# 自动重写aof文件的条件
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
 
# 是否忽略最后一条可能存在问题的指令 
aof-load-truncated yes 
 
文件重写能够压缩aof：
过期数据不再写入
无效命令不再写入，比如先set后delete了，重复设值
多条命令可以合并，比如sadd多个数据，合并的命令会有个常亮界限server.h/AOF_REWRITE_ITEMS_PER_CM中定义，不可更改，值为64
```

aofredis协议格式RESP
```text
Redis客户端与Redis服务端基于一个称作RESP的协议进行通信，RESP全称为Redis Serialization Protocol，也就是Redis序列化协议。虽然RESP为Redis设计，但是它也可以应用在其他客户端-服务端（Client-Server）的软件项目中。
协议的不同部分始终以\r\n（CRLF）终止。
单行字符串的第一个字节为+。
错误消息的第一个字节为-。
整型数字的第一个字节为:。
定长字符串的第一个字节为$。
RESP数组的第一个字节为*。
 
重写触发机制：
手动触发
执行bgrewriteaof命令直接触发AOF重写
 
自动触发
在redis.config配置文件中有两个配置项
 
auto-aof-rewrite-min-size 64MB
auto-aof-rewrite-min-percenrage 100
 
上面两个配置表示： 
- 当AOF文件小于64MB的时候不进行AOF重写 
- 当当前AOF文件比上次AOF重写后的文件大100%的时候进行AOF重写
 
上面两个条件同时满足才触发重写 
可以在redis.conf配置文件中添加这两个参数来自动触发AOF重写，执行bgrewriteaof命令
```

主从复制
```text
主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(master)，后者称为从节点(slave)；数据的复制是单向的，只能由主节点到从节点。
默认情况下，每台Redis服务器都是主节点；且一个主节点可以有多个从节点(或没有从节点)，但一个从节点只能有一个主节点。
 
# 从节点设置为只读模式，只能读取数据，如果未no则写入的数据不会被同步到其他节点 
slave-read-only yes
 
# 数据传输延迟，控制是否关闭TCP_NODELAY。默认关闭 
# 当设置为yes时，TCP会对包进行合并从而减少带宽，但是发送的频率会降低，从节点数据延迟增加，一致性变差；具体发送频率与Linux内核的配置有关，默认配置为40ms。当设置为no时，TCP会立马将主节点的数据发送给从节点，带宽增加但延迟变小。
# 一般来说，只有当应用对Redis数据不一致的容忍度较高，且主从节点之间网络状况不好时，才会设置为yes；多数情况使用默认值no。
repl-disable-tcp-nodelay no 
  
```

三种结构
- 一主一从
- 一主多从（星形拓扑结构）
- 树状主从结构

主从复制的作用
- 数据冗余：主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
- 故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。
- 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，由从节点提供读服务（即写Redis数据时应用连接主节点，读Redis数据时应用连接从节点），分担服务器负载；尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高Redis服务器的并发量。
- 高可用基石：除了上述作用以外，主从复制还是哨兵和集群能够实施的基础，因此说主从复制是Redis高可用的基础。

主从复制开启和关闭
*主从复制的开启，完全是在从节点发起的；不需要我们在主节点做任何事情。*
三种方式
- 配置文件：在从服务器的配置文件中加入：slaveof <masterip> <masterport>
- 启动命令：redis-server启动命令后加入 --slaveof <masterip> <masterport>
- 客户端命令：Redis服务器启动后，直接通过客户端执行命令：slaveof <masterip> <masterport>，则该Redis实例成为从节点。

关闭复制：在从服务器输入：slaveof no one
 
redis密码设置
```text
①配置文件配置，永久有效# requirepass test123
②命令行位置，重启失效redis 127.0.0.1:6379> config set requirepass test123
 
密码验证redis 127.0.0.1:6379>auth  test123
从服务器密码校验redis 127.0.0.1:6380>config set masterauth test123
```
 
 全量复制
 ```text
Redis通过psync命令进行全量复制的过程如下：
psync ? -1 
  
（1）从节点判断无法进行部分复制，向主节点发送全量复制的请求；或从节点发送部分复制的请求，但主节点判断无法进行全量复制；具体判断过程需要在讲述了部分复制原理后再介绍。
  
（2）主节点收到全量复制的命令后，执行bgsave，在后台生成RDB文件，并使用一个缓冲区（称为复制缓冲区）记录从现在开始执行的所有写命令
  
（3）主节点的bgsave执行完成后，将RDB文件发送给从节点；从节点首先清除自己的旧数据，然后载入接收的RDB文件，将数据库状态更新至主节点执行bgsave时的数据库状态
  
（4）主节点将前述复制缓冲区中的所有写命令发送给从节点，从节点执行这些写命令，将数据库状态更新至主节点的最新状态
  
（5）如果从节点开启了AOF，则会触发bgrewriteaof的执行，从而保证AOF文件更新至主节点的最新状态
```

部分复制(2.8提供)
```text
psync <runid> <offset>
（1）复制偏移量
主节点和从节点分别维护一个复制偏移量（offset），代表的是主节点向从节点传递的字节数；主节点每次向从节点传播N个字节数据时，主节点的offset增加N；从节点每次收到主节点传来的N个字节数据时，从节点的offset增加N。
 
offset用于判断主从节点的数据库状态是否一致：如果二者offset相同，则一致；如果offset不同，则不一致，此时可以根据两个offset找出从节点缺少的那部分数据。例如，如果主节点的offset是1000，而从节点的offset是500，那么部分复制就需要将offset为501-1000的数据传递给从节点。而offset为501-1000的数据存储的位置，就是下面要介绍的复制积压缓冲区。

（2）复制积压缓冲区
复制积压缓冲区是由主节点维护的、固定长度的、先进先出(FIFO)队列，默认大小1MB；当主节点开始有从节点时创建，其作用是备份主节点最近发送给从节点的数据。注意，无论主节点有一个还是多个从节点，都只需要一个复制积压缓冲区。
如果offset偏移量之后的数据，仍然都在复制积压缓冲区里，则执行部分复制；
如果offset偏移量之后的数据已不在复制积压缓冲区中（数据已被挤出），则执行全量复制。
可以根据需要增大复制积压缓冲区的大小(通过配置repl-backlog-size)；例如如果网络中断的平均时间是60s，而主节点平均每秒产生的写命令(特定协议格式)所占的字节数为100KB，则复制积压缓冲区的平均需求为6MB，保险起见，可以设置为12MB，来保证绝大多数断线情况都可以使用部分复制。
 
（3）服务器运行ID(runid)
 每个Redis节点(无论主从)，在启动时都会自动生成一个随机ID(每次启动都不一样)，由40个随机的十六进制字符组成；runid用来唯一识别一个Redis节点。通过info Server命令，可以查看节点的runid
 主从节点初次复制时，主节点将自己的runid发送给从节点，从节点将这个runid保存起来；当断线重连时，从节点会将这个runid发送给主节点；主节点根据runid判断能否进行部分复制：
  
 如果从节点保存的runid与主节点现在的runid相同，说明主从节点之前同步过，主节点会继续尝试使用部分复制(到底能不能部分复制还要看offset和复制积压缓冲区的情况)；
 如果从节点保存的runid与主节点现在的runid不同，说明从节点在断线前同步的Redis节点并不是当前的主节点，只能进行全量复制。 
```

*如果原来的master挂了，又重启的runid会改变，可以使用debug reload热加载重启，runid就不会变*

复制配置
```text
#复制的超时时间配置文件设置，超过这个时间没有将rdb传输到从服务器，则全量复制失败
# repl-timeout 60
 
#redis支持无盘复制，生成rdb文件不保存到硬盘而是直接发送到从节点，默认关闭
repl-diskless-sync no
  
#client buffer是在server端实现的一个读取缓冲区，redis server在接收到客户端的请求后，把响应结果写入到
#client buffer中，而不是直接发送给客户端  

client-output-buffer-limit normal 0 0 0
 
#与全量复制阶段主节点的缓冲区大小有关 
#(发送命令缓冲区)表示从库的复制客户端如果60秒内缓冲区小号持续大于64M或者直接超过256M，主节点将直接关闭复制客户端连接 
client-output-buffer-limit slave 256mb 64mb 60
 
#  
client-output-buffer-limit pubsub 32mb 8mb 60 
```
命令传播阶段
```text
# 与命令传播阶段主从节点的超时判断有关，主节点每隔10秒ping一次从节点，判断从节点的存活状态
# repl-ping-slave-period 10
 
# 从节点在主线程中每隔1秒发送replconf ack{offset}命令，给主节点上报自身当前复制偏移量
replconf命令主要作用：
①实时监测主节点网络状态
②上报自身复制偏移量，检查复制数据是否丢失，如果从节点数据丢失，再从主节点的复制缓冲积压区拉取丢失数据
③实现保证从节点的数量和延迟性功能，通过min-slaves-to-write、minslaves-max-lag参数配置定义 
# min-slaves-to-write 3
# min-slaves-max-lag 10
  
辅助保证从节点的数量和延迟：Redis主节点中使用min-slaves-to-write和min-slaves-max-lag参数，来保证主节点在不安全的情况下不会执行写命令
；所谓不安全，是指从节点数量太少，或延迟过高。例如min-slaves-to-write和min-slaves-max-lag分别是3和10，含义是如果从节点数量小于3个，
或所有从节点的延迟值都大于10s，则主节点拒绝执行写命令。而这里从节点延迟值的获取，就是通过主节点接收到REPLCONF ACK命令的时间来判断的，即前面所说的info Replication中的lag值。
```
 
应用中的问题
```text
1. 读写分离及其中的问题
一、数据延迟与不一致
①主从节点在同一机房部署
②监控主节点延迟（通过offset）判断，如果从节点延迟过大，通知应用不再通过该节点读取数据
③Redis复制提供了slave-serve-stale-data参数，默认开启，如果开启则从节点依然响应所有命令
对于无法容忍不一致的应用场景，可以设置no来关闭命令执行，此时从节点除了info和slaveof命令
之外所有的命令只返回“SYNC with master in progress”信息
slave-serve-stale-data yes 
 
二、数据过期问题
在单机版Redis中，存在两种删除策略：
 
惰性删除：服务器不会主动删除数据，只有当客户端查询某个数据时，服务器判断该数据是否过期，如果过期则删除。
定期删除：服务器执行定时任务删除过期数据，但是考虑到内存和CPU的折中（删除会释放内存，但是频繁的删除操作对CPU不友好），该删除的频率和执行时间都受到了限制。
在主从复制场景下，为了主从节点的数据一致性，从节点不会主动删除数据，而是由主节点控制从节点中过期数据的删除。由于主节点的惰性删除和定期删除策略，都不能保证主节点及时对过期数据执行删除操作，因此，当客户端通过Redis从节点读取数据时，很容易读取到已经过期的数据。
 
Redis 3.2中，从节点在读取数据时，增加了对数据是否过期的判断：如果该数据已过期，则不返回给客户端；将Redis升级到3.2可以解决数据过期问题。
 
三、故障切换问题
在没有使用哨兵的读写分离场景下，应用针对读和写分别连接不同的Redis节点；当主节点或从节点出现问题而发生更改时，需要及时修改应用程序读写Redis数据的连接；连接的切换可以手动进行，或者自己写监控程序进行切换，但前者响应慢、容易出错，后者实现复杂，成本都不算低。
  
总结
在使用读写分离之前，可以考虑其他方法增加Redis的读负载能力：如尽量优化主节点（减少慢查询、减少持久化等其他情况带来的阻塞等）提高负载能力；使用Redis集群同时提高读负载能力和写负载能力等。如果使用读写分离，可以使用哨兵(也可采用redis cluster)，使主从节点的故障切换尽可能自动化，并减少对应用程序的侵入。  
2. 复制超时问题  
 
主从节点复制超时是导致复制中断的最重要的原因之一
（1）数据同步阶段：在主从节点进行全量复制bgsave时，主节点需要首先fork子进程将当前数据保存到RDB文件中，然后再将RDB文件通过网络传输到从节点。如果RDB文件过大，主节点在fork子进程+保存RDB文件时耗时过多，可能会导致从节点长时间收不到数据而触发超时；此时从节点会重连主节点，然后再次全量复制，再次超时，再次重连……这是个悲伤的循环。为了避免这种情况的发生，除了注意Redis单机数据量不要过大，另一方面就是适当增大repl-timeout值，具体的大小可以根据bgsave耗时来调整。
 
（2）命令传播阶段：如前所述，在该阶段主节点会向从节点发送PING命令，频率由repl-ping-slave-period控制；该参数应明显小于repl-timeout值(后者至少是前者的几倍)。否则，如果两个参数相等或接近，网络抖动导致个别PING命令丢失，此时恰巧主节点也没有向从节点发送数据，则从节点很容易判断超时。
 
（3）慢查询导致的阻塞：如果主节点或从节点执行了一些慢查询（如keys *或者对大数据的hgetall等），导致服务器阻塞；阻塞期间无法响应复制连接中对方节点的请求，可能导致复制超时。
 
3. 复制中断问题
复制缓冲区溢出
复制缓冲区的大小由client-output-buffer-limit slave {hard limit} {soft limit} {soft seconds}配置，默认值为client-output-buffer-limit slave 256MB 64MB 60，其含义是：如果buffer大于256MB，或者连续60s大于64MB，则主节点会断开与该从节点的连接。该参数是可以通过config set命令动态配置的（即不重启Redis也可以生效）。
 
需要注意的是，复制缓冲区是客户端输出缓冲区的一种，主节点会为每一个从节点分别分配复制缓冲区；而复制积压缓冲区则是一个主节点只有一个，无论它有多少个从节点。
``` 
 
慢查询日志
```text
慢查询分析，通过慢查询分析，找到有问题的命令进行优化
redis客户端执行一条命令,分为4部分
1> 发送命令
2>命令排队
3>命令执行
4>返回结果
慢查询在第三步,统计第三步执行时间
 
slow log 是redis用来记录查询执行时间的日志系统
 
查询执行时间指的是不包括像客户端相应,发送回复等io操作.单单执行一个查询命令所耗费的时间
slow log 保存在内存中,读写速度快.不用担心因为开启slow log而损害redis速度
配置上, 修改redis.conf,当然也可以直接set,但是那样重启之后就没了, 主要两个参数
slowlog-log-slower-than 10000: 执行时间大于多少微秒的查询进行记录.(1s=1000000微秒), 如果值设为0,就是所有的访问记录都会被记录下来.小于0则表示不记录
slowlog-max-len 128:最多能保存多少条日志,slow log本身是一个FIFO队列,当队列大小超过这个值,最旧的一条日志就被删除了.(需要我们自己去slow get慢查询日志进行持久化)
 
每条慢查询日志的4个属性：
标识id、发生时间戳、命令耗时、执行命令和参数
 
慢查询相关命令：
slowlog get [n]，获取慢查询日志列表，可指定返回条数
slowlog len， 获取慢查询日志列表当前长度
slowlog reset， 清空慢查询日志 
``` 

redis-shell
```text
redis-server        启动redis服务
redis-cli           redis客户端命令行工具
redis-benchmark     基准测试工具
redis-check-aof     AOF持久化文件检测工具和修复工具
redis-check-dump    RDB持久化文件检测工具和修复工具
redis-sentinel      启动redis-sentinel
--help查看具体命令 
```

Pipleline
```text
RTT（Round-Trip-Time）,往返时间，Pipleline将一组reids命令组装通过一次RTT
原生批量命令是原子性的，Pipeline是非原子的。
原生批量命令是一个命令对应多个key，Pipeline支持多个命令
原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端共同实现
Pipeline不要组装过多命令，可能产生网络阻塞
原生批量命令：
mget
mset
hmget
hmset...
```

事务
```text
Redis事务三个阶段：
1、事务开始
MULTI命令
2、命令入队
exec（执行）、
discard（放弃）、
watch（监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。）、
multi（标记一个事务块的开始）四个命令立即执行
不是这4个命令，服务器将命令放入FIFO事务队列，然后向客户端返回queued
3、事务执行
 
事务中的错误 
1、入队错误
命令输入错误。。。这种错误可以在命令入队的时候检测出来，事务会discard
2、执行错误
执行命令的时候出错。。。比如操作一个事务开启前存在一个list的key，事务执行时候把这个key当成set类型操作，检测不到，
执行时候会报错，不会中断事务，而是继续执行其他的命令，不受出错命令影响
 
redis事务是原子性的，不支持事务回滚 
 
Watch命令：
乐观锁，在exec命令执行前，监视任意适量的数据库key，并在exec命令执行时，检查被监视的key是否至少有一个已经被修改过，
如果是，服务器将拒绝执行事务，并像客户端返回代表事务执行失败的空回复(nil)
```
 
bitmaps
```text
redis提供setbit、getbit、bitcount、bitop、bitpos五个命令处理二进制数组位
1.Bitmaps是一种特殊的“数据结构”，实质上是一个字符串，操作单元是位。
 
2.命令：
a.setbit：设置值，只能存储0和1，适用二元判断类型
b.getbit：获取值
c.bitcount：统计1的数量，可指定范围
d.bitop：可取交集(and)、并集(or)、非(not)、异或(xor)
e.bitpos：第一个获取某种状态的偏移量
 
3.Bitmaps并不是任何场合都适合，在某些场合适用会有意想不到的效果。
  
位数组的顺序和平时书写的顺序完全相反，比如一个二进制01100010,0位是最右端的，而redis
二进制0位是最左端的0
 
bitmaps适用于进行快速、简单、实时统计、包括独立访客，日活用户的统计，并且bitmaps极其节省空间
 
现代计算机用二进制（位）作为信息的基础单位，1个字节等位8位，例如“big”字符串是由3个字节组成，
但实际在计算机存储时将其用二进制表示，“big”分别对应的ASCII码分别是98、105、103，
对应的二进制分别是01100010、01101001和01100111，如下： 
|<--b-->|<--i-->|<--g-->|
01100010|01101001|01100111
 
getbit与setbit实现：
1、getbit key <offset>
计算byte = offset / 8 ，求得在数组索引第几个字节上
计算bit = （offset % 8） + 1，求得偏移位是第几个二进制位，不是索引位置，索引位置不加1，offset % 8
 
2、setbit key <offset> <value>
计算所需位数组长度，len=（offset/8）+ 1,求得需要几个字节
检查key保存的位数组长度是否小于len，小于则扩展字符串长度位len字节，并将扩展的二进制位值设置为0
计算byte=offset/8，该值记录了offset偏移量指定的二进制位保存在位数组的哪个字节
计算bit=（offset % 8）+1，记录offset偏移量指定的二进制位是byte字节的第几个二进制位
设值
返回原来的值
 
应用场景：
day1：日活用户
day2：日活用户
day3：日活用户
...
bitop and total day1 day2 day3 ....统计出这些天连续活跃的用户数 
``` 

HyperLogLog
```text
有一定误差，会去重
Redis 在 2.8.9 版本添加了 HyperLogLog 结构。
Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。
在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。
但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。
什么是基数?
比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5。 基数估计就是在误差可接受的范围内，快速计算基数。
 
1、PFADD key element [element ...] 添加指定元素到 HyperLogLog 中。
2、PFCOUNT key [key ...] 返回给定 HyperLogLog 的基数估算值。
3、PFMERGE destkey sourcekey [sourcekey ...] 将多个 HyperLogLog 合并为一个 HyperLogLog
```

发布订阅
```text
功能简单，消息不可靠(主要使用一些MQ来处理，redis不做主用)
命令：
1、PSUBSCRIBE pattern [pattern ...] 
订阅一个或多个符合给定模式的频道。
 
2、PUBSUB subcommand [argument [argument ...]]  
查看订阅与发布系统状态。
查看活跃的频道：pubsub channels [pattern]
查看频道订阅次数：pubsub numsub [channel...]
查看模式订阅数：pubsub numpat
 
3、PUBLISH channel message 
将信息发送到指定的频道。
 
4、PUNSUBSCRIBE [pattern [pattern ...]] 
退订所有给定模式的频道。
 
5、SUBSCRIBE channel [channel ...] 
订阅给定的一个或多个频道的信息。
客户端在执行订阅命令之后进入了订阅状态，只能接受subscribe、psubscribe、unsubscribe、punsubscribe四个命令
新开启的订阅客户端，无法接受到该频道之前的消息，redis不会对发布的消息持久化
 
6、UNSUBSCRIBE [channel [channel ...]] 
指退订给定的频道。
```

GEO
```text
使用geohash存储地理位置的坐标
使用有序集合zset存储地理位置的集合
 
redis目前已经到了3.2版本，3.2版本里面新增的一个功能就是对GEO(地理位置)的支持。
 
地理位置大概提供了6个命令，分别为：
GEOADD
GEODIST
GEOHASH
GEOPOS
GEORADIUS
GEORADIUSBYMEMBER 
 
1.命令：GEOADD key longitude latitude member [longitude latitude member ...]
命令描述：将指定的地理空间位置（纬度、经度、名称）添加到指定的key中。
返回值：添加到sorted set元素的数目，但不包括已更新score的元素。 
 
2.命令：GEODIST key member1 member2 [unit]
命令描述：
返回两个给定位置之间的距离。如果两个位置之间的其中一个不存在， 那么命令返回空值。指定单位的参数 unit 必须是以下单位的其中一个：
m 表示单位为米。
km 表示单位为千米。
mi 表示单位为英里。
ft 表示单位为英尺。
 
3.命令：GEOPOS key member [member ...]
命令描述：从key里返回所有给定位置元素的位置（经度和纬度）。
返回值：GEOPOS 命令返回一个数组， 数组中的每个项都由两个元素组成： 第一个元素为给定位置元素的经度， 而第二个元素则为给定位置元素的纬度。当给定的位置元素不存在时， 对应的数组项为空值。
 
4.命令：GEOHASH key member [member ...]
命令描述：返回一个或多个位置元素的 Geohash 表示。通常使用表示位置的元素使用不同的技术，使用Geohash位置52点整数编码。由于编码和解码过程中所使用的初始最小和最大坐标不同，编码的编码也不同于标准。此命令返回一个标准的Geohash
返回值：一个数组， 数组的每个项都是一个 geohash 。 命令返回的 geohash 的位置与用户给定的位置元素的位置一一对应。  
 
5.命令：GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count]
命令描述：
以给定的经纬度为中心， 返回键包含的位置元素当中， 与中心的距离不超过给定最大距离的所有位置元素。
范围可以使用以下其中一个单位：
m 表示单位为米。
km 表示单位为千米。
mi 表示单位为英里。
ft 表示单位为英尺。
在给定以下可选项时， 命令会返回额外的信息：
 
WITHDIST: 在返回位置元素的同时， 将位置元素与中心之间的距离也一并返回。 距离的单位和用户给定的范围单位保持一致。
WITHCOORD: 将位置元素的经度和维度也一并返回。
WITHHASH: 以 52 位有符号整数的形式， 返回位置元素经过原始 geohash 编码的有序集合分值。 这个选项主要用于底层应用或者调试， 实际中的作用并不大。
命令默认返回未排序的位置元素。 通过以下两个参数， 用户可以指定被返回位置元素的排序方式：
 
ASC: 根据中心的位置， 按照从近到远的方式返回位置元素。
DESC: 根据中心的位置， 按照从远到近的方式返回位置元素。
在默认情况下， GEORADIUS 命令会返回所有匹配的位置元素。 虽然用户可以使用 COUNT <count> 选项去获取前 N 个匹配元素， 但是因为命令在内部可能会需要对所有被匹配的元素进行处理， 所以在对一个非常大的区域进行搜索时， 即使只使用 COUNT 选项去获取少量元素， 命令的执行速度也可能会非常慢。 但是从另一方面来说， 使用 COUNT 选项去减少需要返回的元素数量， 对于减少带宽来说仍然是非常有用的。
 
返回值：
在没有给定任何 WITH 选项的情况下， 命令只会返回一个像 [“New York”,”Milan”,”Paris”] 这样的线性（linear）列表。
在指定了 WITHCOORD 、 WITHDIST 、 WITHHASH 等选项的情况下， 命令返回一个二层嵌套数组， 内层的每个子数组就表示一个元素。
在返回嵌套数组时， 子数组的第一个元素总是位置元素的名字。 至于额外的信息， 则会作为子数组的后续元素， 按照以下顺序被返回：
 
以浮点数格式返回的中心与位置元素之间的距离， 单位与用户指定范围时的单位一致。
geohash 整数。
由两个元素组成的坐标，分别为经度和纬度。
 
6.命令：GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count]
命令描述：这个命令和 GEORADIUS 命令一样， 都可以找出位于指定范围内的元素， 但是 GEORADIUSBYMEMBER 的中心点是由给定的位置元素决定的。 
```
 
####redis客户端
#####jedis 
jedis版本
```text
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
使用须知
在实际项目中使用try catch finally的形式来进行代码编码，使用完关闭连接
```
jedis连接池pool
```text

```

直连和连接池比较
 
连接方式|优点|缺点
----|:----:|----
直连|简单方便，适用于少量长连接的场景|1、存在每次新建/关闭TCP连接开销
2、资源无法控制，极端情况会出现连接泄露
3、Jedis对象县城不安全  
连接池|1、无需每次连接都生成Jedis对象，降低开销
2、使用连接池的形式保护和控制资源的使用|相对于直连，使用性对麻烦，尤其在资源的管理上
需要很多参数保证，一旦规划不合理也会有问题

连接池配置 

a|b|c
----|:----:|----
1|2|3