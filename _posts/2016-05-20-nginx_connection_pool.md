---
layout:     post
title:      "理解Nginx连接池"
subtitle:   "\"配置、连接池数据结构、工作原理\""
date:       2016-05-20 12:00:00
author:     "wangyapu"
header-img: "img/post-bg-os-metro.jpg"
tags:
    - nginx
---

# Nginx连接池

## 配置

```bash
worker_processes 12;  

events {  
    use epoll;  
    worker_connections 2048000;  
}
```

在linux系统中，每一个进程能够打开的文件描述符fd是有限的。通过ulimit -n，可以得到一个进程所能够打开的fd的最大数，因为每个socket连接会占用掉一个fd，所以这也会限制我们进程的最大连接数，当然也会直接影响到我们程序所能支持的最大并发数，当fd用完后，再创建socket时，就会失败。Linux系统中open file resource limit的值可以通过如下方式修改：

```bash
echo "2390251" > /proc/sys/fs/file-max
sysctl -p
```
    
对于一个Nginx服务器来说，能创建的socket连接的最大数目可以达到``worker_processes*worker_connections``。

在反向代理环境中，最大并发数量应该是``worker_connections*worker_processes/2``。

## 三个重要的数据结构

Nginx采用基本数据结构ngx_connection_t来表示由客户端主动发起、Nginx服务器被动接收的TCP连接，这类连接可以称为被动连接。ngx_peer_connection_t表示Nginx主动向其他上游服务器建立连接，并以此连接与上游服务器通信，这类连接可以称为主动连接。主动连接是以ngx_connection_t结构体为基础实现的。 

```cpp
struct ngx_connection_s {
    /*
    连接未使用时，data成员用于充当连接池中空闲连接链表中的next指针。当连接被使用时，data的意义由使用它的nginx模块而定，
    如在HTTP框架中，data指向ngx_http_request_t请求
    */
    void               *data; //连接对应的读事件
    ngx_event_t        *read; //连接对应的写事件
    ngx_event_t        *write;

    ngx_socket_t        fd;

    ngx_recv_pt         recv;
    ngx_send_pt         send;
    ngx_recv_chain_pt   recv_chain;
    ngx_send_chain_pt   send_chain;

    //这个连接对应的ngx_listening_t监听对象，此连接由listening监听端口的事件建立
    ngx_listening_t    *listening;

    off_t               sent;

    ngx_log_t          *log;

    //内存池，一般在accept一个新连接时，会创建一个内存池，而在这个连接结束时会销毁内存池
    ngx_pool_t         *pool;

    struct sockaddr    *sockaddr;
    socklen_t           socklen;
    ngx_str_t           addr_text;

    ngx_str_t           proxy_protocol_addr;

#if (NGX_SSL)
    ngx_ssl_connection_t  *ssl;
#endif

    struct sockaddr    *local_sockaddr;
    socklen_t           local_socklen;

    ngx_buf_t          *buffer;

    ngx_queue_t         queue; //将当前连接添加到ngx_cycle_t核心结构中的reuseable_connections_queue双向链表中，表示可重用连接

    ngx_atomic_uint_t   number; //连接使用次数

    ngx_uint_t          requests; //处理请求次数

    unsigned            buffered:8;

    unsigned            log_error:3;     /* ngx_connection_log_error_e */

    unsigned            unexpected_eof:1;
    unsigned            timedout:1;
    unsigned            error:1;
    unsigned            destroyed:1;

    unsigned            idle:1;
    unsigned            reusable:1;
    unsigned            close:1;

    unsigned            sendfile:1;
    unsigned            sndlowat:1;
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */

    unsigned            need_last_buf:1;

#if (NGX_HAVE_IOCP)
    unsigned            accept_context_updated:1;
#endif

#if (NGX_HAVE_AIO_SENDFILE)
    unsigned            busy_count:2;
#endif

#if (NGX_THREADS)
    ngx_thread_task_t  *sendfile_task;
#endif
};

struct ngx_peer_connection_s {
    ngx_connection_t                *connection;

    struct sockaddr                 *sockaddr; //远端服务器socket信息
    socklen_t                        socklen;
    ngx_str_t                       *name;

    ngx_uint_t                       tries; //连接失败后可以重试的次数
    ngx_msec_t                       start_time;

    ngx_event_get_peer_pt            get; //从连接池中获取长连接
    ngx_event_free_peer_pt           free;
    void                            *data;

#if (NGX_SSL)
    ngx_event_set_peer_session_pt    set_session;
    ngx_event_save_peer_session_pt   save_session;
#endif

    ngx_addr_t                      *local;

    int                              rcvbuf;

    ngx_log_t                       *log;

    unsigned                         cached:1;

                                     /* ngx_connection_log_error_e */
    unsigned                         log_error:2;
};

struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;

    ngx_log_t                *log;
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files; //连接文件
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n; //空闲连接个数

    ngx_queue_t               reusable_connections_queue; //再利用连接队列

    ngx_array_t               listening;
    ngx_array_t               paths;
    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```

连接池的模型图如下：

![image](http://wangyapu0714.github.io/img/nginx/nginx_connections.png)


Nginx在接收来自客户端的连接时，所使用的ngx_connection_t数据结构都是在启动阶段就预先分配好的，直接从连接池里获取。

ngx_cycle_t结构中connections和free_connections共同组成一个连接池。connections指向整个连接池数组首部，free_connections指向第一个ngx_connection_t空闲连接。

connections采用数组单链表实现，所有ngx_connection_t都以data成员作为next指针形成一个单链表。一旦有用户发起连接就从free_connections指向的链表头获取一个空闲连接。在连接释放时，只需把该连接再插入到free_connections链表头即可。

可以看出，连接池采用数据单链表的结构有以下优点：

- 可以随机读取，方便初始化。
- 所有操作只需要操作链表的头结点，复杂度O（1）。


### **Release 1.4.7 - 2016.7.14**
#### 1. POM依赖

```xml
<dependency>
   <groupId>com.dianping.cat</groupId>
   <artifactId>cat-client</artifactId>
  <version>1.4.7</version>
 </dependency>
```
#### 2. 功能更新

- [x]   增加了CAT启动的API，初始化传入appkey。如果默认存在app.properties，则以app.properties为准
- [x]   默认采用本地client.xml，如果当没有本地client.xml，默认会从远端服务获取，服务是一个域名：http://cat.dianpingoa.com/ 内网都可以通，此api会自动识别当前ip网络，返回不同的client.xml
- [x]   去除了一个cat依赖，把jsonparse采用自定义的parse，减少客户端依赖









