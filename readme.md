# 个人学习笔记

## 目录

- 1、[linux IO模式](#related-linux-IO)
- 2、[nginx](#related-nginx)
- 3、[CentOS7](#related-centos7)
- 4、[https](#related-https)

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

<h4>防火墙</h4>
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

<h4>端口查看</h4>
```text
netstat -tunlp
netstat -tunlp | grep xxx
```

<a name="related-https"></a>
### 4、https

<h4>对称加密算法</h4>
```text
DES：64位加密
AES：
3DES：
RC4：
```

<h4>非对称加密算法</h4>
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