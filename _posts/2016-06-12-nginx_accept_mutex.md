---
layout:     post
title:      "Nginx的前端负载均衡"
subtitle:   "\"竞争问题、worker负载均衡、惊群问题\""
date:       2016-06-12 12:00:00
author:     "wangyapu"
header-img: "img/post-bg-os-metro.jpg"
tags:
    - nginx
---

# Nginx的前端负载均衡

## 进程模型

nginx进程模型图：

![image](http://wangyapu0714.github.io/img/nginx/nginx_process_model.png)

- nginx采用多进程的方式，nginx启动后一个master进程管理多个worker进程，一般worker进程个数会设置和cpu核数一样。

- master进程接收外界信号，发送信号各个worker进程，并监控各个worker进程的运行状态。如果有worker进程异常挂起或者退出，master进程会重新拉起或重新启动新的worker进程。

- 各个worker进程是独立的，它们公平地接收来自客户端的请求。

nginx进程模型优点：

- 每个worker进程是独立的，互不影响，不需要进行锁控制。
- 一个worker进程挂了，不会停止对外服务，master进程会重新建立新的worker进程。
- 相比于apache的工作方式，节省了大量的内存空间。

## 惊群现象

惊群现象就是一个fd的事件被触发时，所有等待这个fd的线程或进程都会被唤醒。如果多个工作进程同时监听某个套接口，一旦该套接口出现某客户端请求，就会引发所有拥有该套接口的工作进程去争抢这个请求，所有进程都得到返回，但是只有一个进程能争抢到，这种现象称为惊群。一般都是socket的accept()会导致惊群。

## 竞争问题

因为nginx是多进程模型，多个工作进程争抢一个套接口就会出现惊群现象。而且多个进程会竞争客户端连接，如果某个工作进程accept的机会较大，那么该工作进程的空闲连接会很快用完，其他工作进程的空闲连接剩余较多，甚至在空余连接用尽时会造成请求丢失，这种竞争是不公平的。那么nginx是如何解决上述问题的呢？

### 前端负载策略——accept_mutex锁：

nginx使用accept_mutex选项控制进程是否添加accept事件。以下是几个重要的代码片段：

>  ngx_event.c : ngx_event_process_init(ngx_cycle_t *cycle)


    if (ccf->master && ccf->worker_processes > 1 && ecf->accept_mutex) {
        ngx_use_accept_mutex = 1; //开启负载均衡策略
        ngx_accept_mutex_held = 0;
        ngx_accept_mutex_delay = ecf->accept_mutex_delay;

    } else {
        ngx_use_accept_mutex = 0;
    }


>  ngx_event_accept.c : ngx_event_accept(ngx_event_t *ev)


    ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;
                              
                              
>  ngx_event.c : ngx_process_events_and_timers(ngx_cycle_t *cycle)


    if (ngx_use_accept_mutex) {
        // nginx.conf配置了每个nginx worker进程能够处理的最大连接数，当达到最大数的7/8时，ngx_accept_disabled > 0，此时处于满负荷，将不再去处理新连接
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            //拿到锁，flag置为NGX_POST_EVENTS，意味着ngx_process_events函数中任何事件都将延后处理，会把accept事件都放到ngx_posted_accept_events链表中
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }

            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;

            } else {
            
                 //拿不到锁就不会处理监听的句柄，timer是传给epoll_wait的超时时间，修改为最大ngx_accept_mutex_delay意味着epoll_wait更短的超时返回，以免新连接长时间没有得到处理
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }

    delta = ngx_current_msec;

    (void) ngx_process_events(cycle, timer, flags);

    delta = ngx_current_msec - delta;

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "timer delta: %M", delta);

    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        ngx_event_expire_timers();
    }

    ngx_event_process_posted(cycle, &ngx_posted_events);

>  ngx_event_accept.c : ngx_trylock_accept_mutex(ngx_cycle_t *cycle)


    ngx_int_t
    ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
    {
        if (ngx_shmtx_trylock(&ngx_accept_mutex)) {
    
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "accept mutex locked");
    
            if (ngx_accept_mutex_held
                && ngx_accept_events == 0
                && !(ngx_event_flags & NGX_USE_RTSIG_EVENT))
            {
                return NGX_OK;
            }
    
            //ngx_enable_accept_events会把监听的句柄都塞入到本worker进程的epoll中
            if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
                ngx_shmtx_unlock(&ngx_accept_mutex);
                return NGX_ERROR;
            }
    
            ngx_accept_events = 0;
            ngx_accept_mutex_held = 1;
    
            return NGX_OK;
        }
    
        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex lock failed: %ui", ngx_accept_mutex_held);
        
        //没有拿到锁ngx_disable_accept_events会把监听句柄从epoll中取出.
        if (ngx_accept_mutex_held) {
            if (ngx_disable_accept_events(cycle) == NGX_ERROR) {
                return NGX_ERROR;
            }
    
            ngx_accept_mutex_held = 0;
        }
    
        return NGX_OK;
    }
    
可以看出，多个worker工作进程在同一时刻只有一个worker进程拿到accept_mutex锁，并向自己的epoll中加入监听的句柄，此时该worker进程flag置为NGX_POST_EVENTS，在ngx_process_events里NGX_POST_EVENTS标志将事件都放入ngx_posted_events链表中，延迟到锁释放了再处理，因为这些请求耗时相对较久。


## 总结

- 以上描述的worker进程处理请求的原理表名nginx根本不会出现惊群现象。

- ngx_accept_disabled的变量来控制worker进程是否去竞争accept_mutex锁，虽然nginx的accept_mutax锁方法给每个worker进程提供平等的机会来竞争锁，但这种方法得到实际结果却不一定平均。



