---
layout:     post
title:      "天池中间件大赛——单机百万消息队列存储分享"
subtitle:   "\"单机百万小队列、文件存储\""
date:       2018-08-01 12:00:00
author:     "wangyapu"
header-img: "img/read_notes/tianchi_mq.jpg"
tags:
    - 架构
---

# 天池中间件大赛——单机百万消息队列存储分享

   这次天池中间件性能大赛初赛和复赛的成绩都正好是`第五名`，本次整理了复赛《单机百万消息队列的存储设计》的思路方案分享给大家，实现方案上也是决赛队伍中相对比较特别的。
   
## 赛题回顾
   
- 实现一个进程内的队列引擎，`单机可支持100万队列以上`。
- 实现消息put、get接口。
- 在规定时间内完成`数据发送、索引校检、数据消费`三个阶段评测。 

## 评测逻辑

- 各个阶段线程数在20~30左右。
- 发送阶段：消息大小在50字节左右，消息条数在20亿条左右，也即发送总数据在100G左右。
- 索引校验阶段：会对所有队列的索引进行随机校验；平均每个队列会校验1~2次。
- 顺序消费阶段：挑选20%的队列进行全部读取和校验； 
- 发送阶段最大耗时不能超过1800s；索引校验阶段和顺序消费阶段加在一起，最大耗时也不能超过1800s；超时会被判断为评测失败。

## 评测环境

- 测试环境为4c8g的ECS虚拟机。
- 带一块300G左右大小的SSD磁盘。SSD性能大致如下：iops 1w左右；块读写能力(一次读写4K以上)在200MB/s左右。

##  赛题分析

对于单机几百的大队列来说业务已有成熟的方案，Kafka和RocketMQ。

| 方案 | 几百个大队列 |
| --- | --- |
| Kafka | 每个队列一个文件（独立存储） |
| RocketMQ | 所有队列共用一个文件（混合存储） |

若直接采用现有的方案，在百万量级的小队列场景都有极大的弊端。

| 方案 | 百万队列场景弊端 |
| --- | --- |
| Kafka独立存储 | 单个小队列数据量少，批量化程度完全取决于内存大小，落盘时间长，写数据容易触发IOPS瓶颈 |
| RocketMQ混合存储 | 随机读严重，一个块中连续数据很低，读速度很慢，消费速度完全受限于IOPS |

为了兼顾读写速度，我们最终采用了折中的设计方案：`多个队列merge，共享一个块存储。`

![](http://pcb6gvhga.bkt.clouddn.com/15326987338356.jpg)


## 设计核心思想

- `设计上要支持边写边读`
- `多个队列需要合并处理`
- `单个队列的数据存储部分连续`
- `索引稀疏，尽可能常驻内存`

## 架构设计

![](http://pcb6gvhga.bkt.clouddn.com/15327015808705.jpg)


架构图中Bucket Manager和Group Manager分别对百万队列进行分桶以及合并管理，然后左右两边是分别是写模块和读模块，数据写入包括队列merge处理，消息块落盘。读模块包括索引管理和读缓存。（见左图）

bucket、group、queue的关系：对消息队列进行bucket处理，每个bucket包含多个group，group是我们进行队列merge的最小单元，每个group管理固定数量的队列。（见右图）


## 存储设计

![](http://pcb6gvhga.bkt.clouddn.com/15327026223088.jpg)


- 对百万队列进行分桶处理。
- 每个Bucket中分为多个Group，每个Group为一个读写单位，对队列进行merge，同时更新索引和数据文件。
- 单个Group里对M个队列进行合并，超过16k或者压缩超过16K（可配置）进行索引更新和落盘。
- 索引部分针对每个Block块建立一个L2二级索引，然后每16个L2建立一个L1一级索引。
- 数据文件采用混合存储，对Block块顺序存储。

接下来对整个存储每个阶段的细节进行展开分析，包括队列合并、索引管理和数据落盘。

### MQ Merge

#### 1. 百万队列数据Bucket Hash分桶

![](http://pcb6gvhga.bkt.clouddn.com/15327037399900.jpg)


#### 2. Bucket视角

![](http://pcb6gvhga.bkt.clouddn.com/15327038387055.jpg)

- 每个Bucket分配多个Group
- Group是管理多个队列的最小单位

#### 3. Group分配过程

![](http://pcb6gvhga.bkt.clouddn.com/15327039962290.jpg)


- 每个bucket持有一把锁，顺序为队列分配group，这里我们假设merge的数量为4个队列。
- 数据的达到是随机的，根据队列的先后顺序加入当前Group。
- `当Group达到M个后便形成一个固定分组。相同队列会在Group内进行合并，新的队列数据将继续分配Group接收。`

#### 4. Group视角的数据写入

![](http://pcb6gvhga.bkt.clouddn.com/15327042392044.jpg)


- 每个Group会分配Memtable的Block块用于实时写入。
- `当Block达到16k（可配置）时以队列为单位进行数据排序，保证单个队列数据连续。`
- 字节对齐，Memtable变为不可变的Immemtable准备落盘。
- 开辟新的Block接收数据写入。

### 索引管理

#### 1. L2二级索引

L2二级索引与数据存储的位置息息相关，见下图。`为每个排序后的Block块建立一个L2索引，L2索引的结构分为文件偏移(file offset)，数据压缩大小(size)，原始大小(raw size)，因为我们是多个队列merge，然后接下来是每个队列相对于起始位置的delta offset以及消息数量。`

![](http://pcb6gvhga.bkt.clouddn.com/15327049054876.jpg)

#### 2. L1一级索引

为了加快查询速度，在L2基础上建立L1一级索引，每`16个L2建立一个L1，L1按照时间先后顺序存放。`L1和L2的组织关系如下：

![](http://pcb6gvhga.bkt.clouddn.com/15328434447728.jpg)


L1索引的结构非常简单，file id对应消息存储的文件id，以及16个Block块中每个队列消息的起始序列号seq num。例如MQ1从序列号1000开始，MQ2从序列号2000开始等等。

![](http://pcb6gvhga.bkt.clouddn.com/15328440031305.jpg)

#### 3. Index Query

如何根据索引定位需要查找的数据？

对L1先进行二分查找，定位到上下界范围，然后对范围内的所有L2进行顺序遍历。

![](http://pcb6gvhga.bkt.clouddn.com/15328442977906.jpg)


### Data Flush

#### 1. 同步Flush

当blcok超过指定大小后，根据桶的hashcode再进行一次mask操作将group中的队列数据同步写入到m个文件中。

同步刷盘主要尝试了两种方案：Nio和Dio。`Dio大约性能比Nio提升约5%`。CPP使用DIO是非常方便的，然而作为Java Coder你也许是第一次听说DIO，在Java中并没有提供直接使用DIO的接口，可以通过JNA的方式调用。

DIO（DIRECT IO，直接IO），出于对系统cache和调度策略的不满，用户自己在应用层定制自己的文件读写。`DIO最大的优点就是能够减少OS内核缓冲区和应用程序地址空间的数据拷贝次数，降低文件读写时的CPU开销以及内存的占用。然而DIO的缺陷也很明显，DIO在数据读取时会造成磁盘大量的IO，它并没有缓冲IO从PageCache获取数据的优势。`

![](http://pcb6gvhga.bkt.clouddn.com/15328450129918.jpg)


这里就遇到一个问题，同样配置的阿里云机器测试随机数据同步写入性能是非常高的，但是线上的评测数据都是58字节，数据过于规整导致同一时间落盘的概率很大，出现了大量的锁竞争。所以这里做了一个小的改进：按概率随机4K、8K、16K进行落盘，写性能虽有一定提升，但是效果也是不太理想，于是采用了第二种思路异步刷盘。

#### 2. 异步Flush

采用RingBuffer接收block块，使用AIO对多个block块进行Batch刷盘，减少IO Copy的次数。异步刷盘写性能有了显著的提升。

![](http://pcb6gvhga.bkt.clouddn.com/15328776432365.jpg)

以下是异步Flush的核心代码：

```
while (gWriterThread) {
    if (taskQueue->pop(task)) {
        writer->mWriting.store(true);
        do {
            // 使用异步IO
            aiocb *pAiocb = aiocb_list[aio_size++];
            memset(pAiocb, 0, sizeof(aiocb));
            pAiocb->aio_lio_opcode = LIO_WRITE;
            pAiocb->aio_buf = task.mWriteCache.mCache;
            pAiocb->aio_nbytes = task.mWriteCache.mSize;
            pAiocb->aio_offset = task.mWriteCache.mStartOffset;
            pAiocb->aio_fildes = task.mBlockFile->mFd;
            pAiocb->aio_sigevent.sigev_value.sival_ptr = task.mBlockFile;
            task.mBlockFile->mWriting = true;
            
            if (aio_size >= MAX_AIO_TASK_COUNT) {
                break;
            }
        } while (taskQueue->pop(task));
        
        if (aio_size > 0) {
            if (0 != lio_listio(LIO_WAIT, aiocb_list, aio_size, NULL)) {
                aos_fatal_log("aio error %d %s.", errno, strerror(errno));
            }
            
            for (int i = 0; i < aio_size; ++i) {
                ((BlockFile *) aiocb_list[i]->aio_sigevent.sigev_value.sival_ptr)->mWriting = false;
                free((void *) aiocb_list[i]->aio_buf);
            }
            aio_size = 0;
        }
    } else {
        ++waitCount;
        sched_yield();
        if (waitCount > 100000) {
            usleep(10000);
        }
    }
}
```

## 读缓存设计

### 数据读取流程

- 根据队列Hash定位Bucket桶。
- 二分查找定位L1索引和L2索引。
- 在一定时机会执行预读取操作。
- 数据先从缓存中做查找，缓存命中直接返回，失效则回源到SSD。

整个流程主要有两个优化点：`预读取和读缓存。`

![](http://pcb6gvhga.bkt.clouddn.com/15328788414149.jpg)


### 预读取优化

#### 1. 记录上一次读取（消费）的offset

主要有两个作用：

- 加快查询数据的速度。
- 用于判断预读取时机。

#### 2. 预读取时机

顺序消费且已经消费到当前block尾，则进行预读取操作。如何判断顺序消费？判断上次消费的结束位置是否与这次消费的起始位置相等。


```cpp
if (msgCount >= destCount) {
    if (mLastGetSequeneNum == offsetCount &&
        beginIndex + 1 < mL2IndexCount &&
        beginOffsetCount + blockIndex.mMsgDeltaIndexCount <= offsetCount + msgCount + msgCount) {
        MessageBlockIndex &nextIndex = mL2IndexArray[beginIndex + 1];
        // 预读取
#ifdef __linux__
        readahead(pManager->GetFd(hash), nextIndex.mFileOffset, PER_BLOCK_SIZE);
#endif
    }
    mLastGetSequeneNum = offsetCount + msgCount;
    return msgCount;
}
```

### Read Cache

关于read cache做了一些精巧的小设计，保证足够简单高效。

- `分桶`（部分隔离），一定程度缓解缓存饿死现象。
- `数组 + 自旋锁 + 原子变量`实现了一个循环分配缓存块的方案。
- `双向指针绑定`高效定位缓存节点。

#### 1. Read Cache全貌

Read Cache一共分为N=64（可配）个Bucket，每个Bucket中包含M=3200（可配）个缓存块，大概总计20w左右的缓存块，每个是4k，大约占用800M的内存空间。

![](http://pcb6gvhga.bkt.clouddn.com/15328805562929.jpg)


#### 2. 核心数据结构

![](http://pcb6gvhga.bkt.clouddn.com/15328808774528.jpg)

关于缓存的核心数据结构，我们并没有从队列的角度出发，而是针对L2索引和缓存块进行了绑定，这里设计了一个双向指针。`判断缓存是否有效的核心思路：check双向指针是否相等。`

```cpp
CacheItem cachedItem = (CacheItem *) index->mCache;
cachedItem->mIndexPtr == (void *) index;
```

#### 3. 算法实现

**3.1 Bucket分桶**

- 获取L2 Index。
- 根据Manager Hash % N，找到对应的缓存Bucket。
- L2还没有对应缓存块，需要进行缓存块分配。

![](http://pcb6gvhga.bkt.clouddn.com/15328818367354.jpg)


**3.2 Alloc Cache Block**

- 原子变量进行自加操作，同时对M=3200块取模， `count.fetch_add(1) % M = index`
- 分配下标为index的Cache Block。
- 然后将对应的缓存块和我们的队列的L2索引进行双向指针绑定，同时对缓存块数据进行数据填充。

![](http://pcb6gvhga.bkt.clouddn.com/15330541290117.jpg)


**3.3 Cache Hit**

- index->mCache == index->mCache->index，双向指针相等，缓存命中，然后做数据读取。

**3.4 Cache Page Replace**

- MQ1绑定的缓存块已经被MQ2替换。
- index->mCache != index->mCache->index，双向指针已经不相等，缓存失效。需要为MQ1分配新的缓存块。

![](http://pcb6gvhga.bkt.clouddn.com/15330566319233.jpg)

- 原子变量进行自加操作，同时对M=3200块取模， 例如：`count.fetch_add(1) % M = M-1`，找到新的缓存块进行重新绑定。`说明：整个分配的逻辑是一个循环使用的过程，当所有的缓存桶都被使用，那么会从数组首地址开始重新分配、替换。`

![](http://pcb6gvhga.bkt.clouddn.com/15330566697685.jpg)

#### 4. Read Cache & LRU & PageCache 对比

开始我们尝试了两种读缓存方案：最简单的LRU缓存和直接使用PageCache读取。PageCache所实现的其实是高级版的LRU缓存。在顺序读的场景下，我们自己实现的读缓存（Cycle Cache Allocate，暂简称为CCA）与LRU、PageCache的优劣分析对比如下：

- LRU针对每次操作进行调整，CCA针对缓存块需要分配时进行替换。
- LRU从队列角度建立映射表，CCA针对索引和缓存块双向指针绑定。
- CCA中自旋锁是针对每个缓存块加锁，锁粒度更小。LRU需要对整个链表加锁。
- 达到同等命中率的情况下，CCA比Page Cache节省至少1~2倍的内存。

### 总结

#### 创新点 

- 针对百万小队列，实际硬件资源，兼顾读写性能，提出`多队列Merge，保证队列局部连续的存储方式。`
- 针对队列相对无关性及MQ连续读取的场景，设计实现了O(1)的Read Cache，只需要约800M内存即可支持20W队列的高效率的读取，命中率高达85%。
- 支持多队列Merge的索引存储方案，`资源利用率低`，约300~400MB索引即可支撑百万队列、100GB数据量高并发读写。

#### 工程价值：

- 通用性、随机性、健壮性较好，支持对任意队列进行merge。
- 较少出现单队列消息太少而导致block块未刷盘的情况，块的填充会比较均匀。
- 不必等待单个队列满而进行批量刷盘，减少内存占用，更多的内存可支持更多的队列。
- 可以读写同步进行，常驻内存的索引结构也适合落盘，应对机器重启、持久化等场景。

### 思考

- 为什么没有使用mmap？为什么mmap写入会出现卡顿？


