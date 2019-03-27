---
layout:     post
title:      "Nginx内存池管理"
subtitle:   "\"内存池数据结构和操作\""
date:       2016-06-20 12:00:00
author:     "wangyapu"
header-img: "img/post-bg-os-metro.jpg"
tags:
    - nginx
---

# Nginx的内存池管理

## 内存池工作原理

nginx以并发能力强，占用内存少著称，实现代码简洁精妙。

nginx内存池的基本思想是预先分配一大块内存作为内存池，小块内存申请和释放从内存池中分配，大块内存另外进行分配。分配的内存块地址会进行内存对齐，提高IO效率。

- 优点：

    将大量小内存的申请聚集到一块，比malloc更快。

    减少内存碎片，防止内存泄漏。

    减少内存管理的复杂度。

- 缺点：

    一定程度造成内存空间浪费，因为采用的以空间换时间方案。

## 内存池数据结构

    struct ngx_pool_large_s {
        ngx_pool_large_t     *next; //用链表组织，指向下一块较大内存
        void                 *alloc; //实际内存地址
    };
    
    
    typedef struct {
        u_char               *last; //当前内存分配结束位置，即下一段可分配内存的起始位置
        u_char               *end; //内存池结束位置
        ngx_pool_t           *next; //链接到下一个内存池，内存池的很多块内存就是通过该指针连成链表
        ngx_uint_t            failed; //内存池分配失败次数
    } ngx_pool_data_t;
    
    
    struct ngx_pool_s {
        ngx_pool_data_t       d; //内存池的数据块
        size_t                max; //数据块大小，小块内存的最大值
        ngx_pool_t           *current; //当前内存池的指针
        ngx_chain_t          *chain; //该指针挂接一个ngx_chain_t结构
        ngx_pool_large_t     *large; //指向大块内存分配，nginx中，大块内存分配直接采用标准系统接口malloc
        ngx_pool_cleanup_t   *cleanup; //释放内存的callback
        ngx_log_t            *log;
    };
    
    
    struct ngx_pool_cleanup_s {
        ngx_pool_cleanup_pt   handler;
        void                 *data;
        ngx_pool_cleanup_t   *next;
    };

## 内存池的物理结构 

![image](http://wangyapu.github.io/img/nginx/nginx_pool_model.jpg)

## 内存池操作

### 创建内存池

    ngx_pool_t *
    ngx_create_pool(size_t size, ngx_log_t *log)
    {
        ngx_pool_t  *p;
    
        p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
        if (p == NULL) {
            return NULL;
        }
    
        p->d.last = (u_char *) p + sizeof(ngx_pool_t);
        p->d.end = (u_char *) p + size;
        p->d.next = NULL;
        p->d.failed = 0;
    
        size = size - sizeof(ngx_pool_t);
        //最大不超过4095B
        p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    
        p->current = p;
        p->chain = NULL;
        p->large = NULL;
        p->cleanup = NULL;
        p->log = log;
    
        return p;
    }

### 销毁内存池

    void
    ngx_destroy_pool(ngx_pool_t *pool)
    {
        ngx_pool_t          *p, *n;
        ngx_pool_large_t    *l;
        ngx_pool_cleanup_t  *c;
    
        /**
         * cleanup指向析构函数，用于执行相关的内存池销毁之前的清理工作，如文件的关闭等。
         * 对内存池中的析构函数遍历调用
         */
        for (c = pool->cleanup; c; c = c->next) {
            if (c->handler) {
                ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                               "run cleanup: %p", c);
                c->handler(c->data);
            }
        }
    
        for (l = pool->large; l; l = l->next) {
    
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);
    
            if (l->alloc) {
                ngx_free(l->alloc);
            }
        }
    
    #if (NGX_DEBUG)
    
        /*
         * we could allocate the pool->log from this pool
         * so we cannot use this log while free()ing the pool
         */
    
        for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
            ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p, unused: %uz", p, p->d.end - p->d.last);
    
            if (n == NULL) {
                break;
            }
        }
    
    #endif
    
        //彻底销毁内存池
        for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
            ngx_free(p);
    
            if (n == NULL) {
                break;
            }
        }
    }


### 内存池分配

    void *
    ngx_palloc(ngx_pool_t *pool, size_t size)
    {
        u_char      *m;
        ngx_pool_t  *p;
    
        //小于max值，则从current结点开始遍历pool链表
        if (size <= pool->max) {
    
            p = pool->current;
    
            do {
                //以last开始，计算以NGX_ALIGNMENT对齐的偏移位置指针
                m = ngx_align_ptr(p->d.last, NGX_ALIGNMENT);
    
                if ((size_t) (p->d.end - m) >= size) {
                    p->d.last = m + size;
    
                    return m;
                }
    
                //如果不满足，则查找下一个链
                p = p->d.next;
    
            } while (p);
            /**
             * 遍历完整个内存池链表均未找到合适大小的内存块供分配，则执行ngx_palloc_block()来分配
             * 为该内存池再分配一个block，该block的大小为链表中前面每一个block大小的值
             * 一个内存池是由多个block链接起来的。分配成功后，将该block链入该poll链的最后
             */
            return ngx_palloc_block(pool, size);
        }
        //大于max值，则执行大块内存分配的函数ngx_palloc_large，在large链表里分配内存
        return ngx_palloc_large(pool, size);
    }
    
    
    static void *
    ngx_palloc_block(ngx_pool_t *pool, size_t size)
    {
        u_char      *m;
        size_t       psize;
        ngx_pool_t  *p, *new;
    
        psize = (size_t) (pool->d.end - (u_char *) pool);
    
        m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
        if (m == NULL) {
            return NULL;
        }
    
        //block初始化
        new = (ngx_pool_t *) m;
    
        new->d.end = m + psize;
        new->d.next = NULL;
        new->d.failed = 0;
    
        m += sizeof(ngx_pool_data_t);
        m = ngx_align_ptr(m, NGX_ALIGNMENT);
        new->d.last = m + size;
    
        for (p = pool->current; p->d.next; p = p->d.next) {
            //分配失败次数大于4,移动current指针
            if (p->d.failed++ > 4) {
                pool->current = p->d.next;
            }
        }
    
        ///将分配的block加入内存池
        p->d.next = new;
    
        return m;
    }
    
    static void *
    ngx_palloc_large(ngx_pool_t *pool, size_t size)
    {
        void              *p;
        ngx_uint_t         n;
        ngx_pool_large_t  *large;
    
        p = ngx_alloc(size, pool->log);
        if (p == NULL) {
            return NULL;
        }
    
        n = 0;
    
        for (large = pool->large; large; large = large->next) {
            if (large->alloc == NULL) {
                large->alloc = p;
                return p;
            }
    
            if (n++ > 3) {
                break;
            }
        }
    
        large = ngx_palloc(pool, sizeof(ngx_pool_large_t));
        if (large == NULL) {
            ngx_free(p);
            return NULL;
        }
    
        large->alloc = p;
        large->next = pool->large;
        pool->large = large;
    
        return p;
    }


## 总结

- nginx内存池基本思想：申请大块内存，避免“细水长流”。
- 内存池第一块内存前40字节为ngx_pool_t结构，后续加入的内存块前16个字节为ngx_pool_data_t结构，这两个结构之后便是真正可以分配内存区域。 
- nginx将内存池分了不同的等级，有进程级的内存池、connection级的内存池、request级的内存池。
- nginx将小块内存的申请聚集到一起申请，然后一起释放，降低内存碎片。