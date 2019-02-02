---
layout:     post
title:      "第一届天池 PolarDB 数据库性能大赛"
subtitle:   "\"文件存储、高效的 KV 存储引擎\""
date:       2019-02-01 12:00:00
author:     "wangyapu"
header-img: "img/read_notes/tianchi_polardb.jpg"
tags:
    - 架构
---

# 第一届天池 PolarDB 数据库性能大赛

这次天池 PolarDB 数据库性能大赛竞争相当激烈，眼睛一闭一睁成绩就会被血洗，最后榜单成绩是第三名，答辩翻车了，最终取得了大赛季军。云计算领域接触的是最前沿的技术，阿里云的 PolarDB 作为云原生数据库里程碑式的革新产品，也为这次比赛提供了最先进的硬件环境。

整个比赛获益良多，体会比较深的两点：

- 为了充分使用新硬件, 榨干硬件的红利来达到极致的性能，一定要 Benchmark Everything，经验并不一定是对的，实践出真知。
- 从大的解决方案，到小的细节优化，需要足够的勇气尝试不同的思路，从而不断完善，达到最优解。

## 比赛背景

1. 以 Optane SSD 为背景，实现高效的 KV 存储引擎。
2. 实现 Write、Read 和 Range 接口。
3. 正确性检测：保证进程意外退出不会造成数据丢失。
4. 在规定时间内完成随机写、随机读、Range（顺序读）三个性能评测阶段。

## 赛题剖析

1. 正确性检测 kill -9 模拟进程意外退出，需要保证数据不丢失。
2. 只有 2G 物理内存可用。
3. 64个线程并发顺序读取，每个线程各使用 Range 有序（增序）遍历全量数据 2 次。`设计出有利于 Range 并且兼顾读写性能的架构至关重要。`
4. 随机写、随机读、顺序读三个阶段都会重新 Open DB，`尽可能挖掘 Open 阶段的细节优化点`。
5. `Drop Cache 的时间也计入总成绩，尽可能减少 PageCache 的使用`。

## 核心设计思想

以 Range 为核心，同时兼顾随机写和随机读的性能。

1. 划分`多个 DB 分片`减少锁冲突，提高并发度。
2. Key/Value `数据分离`，可使用块偏移地址表示数据位置。
3. 充分利用 PageCache 避免进程退出不丢失数据，引入 `Mmap 读写数据`。
4. `索引全量加载到内存，保证索引分区内和分区之间有序。`
5. Range 阶段需要`大块缓存`，保证顺序读。
6. Open DB `并行加载 DB 分片`，每个分片的加载过程是独立的。

## 全局架构

随机写和随机读都会根据 key 定位到具体的数据分片，转变为具体某个 DB 分片的读写操作。Range 查询需要定位到分片的上界和下界，然后依次顺序将分片进行遍历。

![全局架构](http://wangyapu.iocoder.cn/overview.png)

DB 划分为多个分片的作用：

1. 降低锁冲突。
2. 提高读、写、Open 并发度。
3. 降低数据定位时间。
4. 有利于 Range 大块缓存数据。

> DB分片需要支持范围查询：DB 分片内和分片之间数据有序。

## 存储方案

![存储方案](http://wangyapu.iocoder.cn/store.png)

1. 根据 Key 的高 11 位定位 DB 分片以及分片所对应的文件。
2. 单个 DB 分片视角：每个 DB 分片主要分为索引文件、MergeFile、数据文件。其中索引文件存储数据的 Key 以及 Offset，MergeFile 的作用用于聚合 IO，`将 4k 聚合成 16K 后进行落盘`。
3. 数据文件视角：比赛初期数据文件与 DB 分片采用了 1 对 1 的架构设计，后期为了降低随机IO，提高写入速度，`数据文件与 DB 分片设计为一对多的方式`，每个文件管理多个分片。

## 关键参数

1. 64 个数据文件，2048 个 DB 分片，数据文件与 DB 分片是一对多的关系。
2. 4 个 Value 合并为 16K 落盘，减少磁盘交互次数。
3. 创建 8 个 DB 分片、1G 的缓存池。
4. 2 个 Range 预读线程，每个 DB 分片均分成 2 段并发预读。

## 随机写设计思路

1. 当需要写入数据时，首先定位到数据分片以及数据文件，进行加锁操作。
2. 先将数据写入 MergeBuffer，更新索引。
3. 当下一个数据来时执行同样的操作，当 MergeBuffer 填满 16K，使用 DIO 将 16K 数据批量刷盘。

![随机写过程](http://wangyapu.iocoder.cn/C38BDA59-5666-46E7-A948-84DCD5ABE1E8.png)


## 随机写关键点

1. `Key 转化为 uint64_t`，根据 uint64_t 的高 11 位定位 DB 分片以及数据文件。
2. 索引以及 Megre IO 无法避免 kill -9 检测，`采用 Mmap 映射文件读写`。
3. `对象复用`：每个分片独立一份 16k Buffer 复用，减少资源开销。
4. `先写数据文件再更新索引`，索引直接使用指针地址赋值，避免内存拷贝。
5. Value 采用 `DIO 16K 字节对齐写入`。

```cpp

KeyOnly keyOnly;
keyOnly.key = keyLong;
KeyOnly *ptr = reinterpret_cast<KeyOnly *>(mIndexPtr);
pthread_mutex_lock(&mMutex);
memcpy(static_cast<char *>(mSegmentBuffer) + mSegmentBufferIndex * 4096, value.data(), mSegmentBufferIndex++;

if (mSegmentBufferIndex == MergeLimit) {
    pwrite64(mDataDirectFd, mSegmentBuffer, MergeBufferSize, mWritePosition);
    mWritePosition += MergeBufferSize;
    mSegmentBufferIndex = 0;
}

ptr[mTotalKey] = keyOnly;
mTotalKey++;
pthread_mutex_unlock(&mMutex);

```

## Open DB 阶段

细节决定成败，因为三个阶段都会重新打开 DB，所以 Open DB 阶段也成为优化的关键一环。

1. `posix_fallocate 预先分配文件空间`。
2. 64 个线程并发加载 2048 个 DB 分片，每个分片的加载都是独立的。
3. 顺序、`批量读取`索引文件将 KeyOffset 加载到内存 Vector。索引的结构非常简单，只记录key 和 逻辑偏移offset，offset * 4096 计算出数据的物理偏移地址。
4. 采用`快排`对 Key 进行从小到大排序，相同 Key 的 Offset 也从小到大排列。方便实现`点查询和范围查询`。考虑到性能评测阶段基本没有重复的key，所以 Open 阶段去除了索引的去重工作，改为上界查询。

索引结构：

```cpp
struct KeyOffset {
    uint64_t key;
    uint32_t offset;

    KeyOffset()
            : key(0),
              offset(0) {
    }

    KeyOffset(uint64_t key, uint32_t offset)
            : key(key),
              offset(offset) {
    }
} __attribute__((packed));
```

## Drop Cache 优化

`Drop Cache 一共包含清理 PageCache、dentries 和 inodes`，可根据参数控制。

```bash
sysctl -w vm.drop_cache = 1 // 清理 pagecache 
sysctl -w vm.drop_cache = 2 // 清理 dentries（目录缓存）和 inodes 
sysctl -w vm.drop_cache = 3 // 清理 pagecache、dentries 和 inodes 
```

PageCache 是重灾区，尽可能在使用 PageCache 的地方做一些细节优化。

1. 将每个分片 16K 合并 I/O 的在 close 时强制写入磁盘的数据文件。
2. 索引加载完成后，调用 `POSIX_FADV_DONTNEED` 则将指定的磁盘文件中数据从 Page Cache 中换出，稳定提升 20 ~ 30ms。

## 随机读

随机读核心就是实现`点查询 O(logn)`。

1. 根据 Key 定位 DB 分片以及数据文件。
2. 二分查找 DB 分片的索引数据，得到 Key 所对应 Value 数据的 offset 上界。
3. 根据数据的分区号和偏移地址采用 DIO 读取 Value数据。

```cpp
uint32_t offset = binarySearch(keyLong);

if (unlikely(offset == UINT32_MAX)) {
    return kNotFound;
}
static __thread void *readBuffer = NULL;
if (unlikely(readBuffer == NULL)) {
    posix_memalign(&readBuffer, getpagesize(), 4096);
}
if (unlikely(value->size() != 4096)) {
    value->resize(4096);
}
RetCode ret = readValue(offset - 1, readBuffer);
memcpy(&((*value)[0]), readBuffer, 4096);
return ret;
```

## Range 

### Range 核心设计思想

- `预读先行`，保证预读和 Range 线程可以齐头并进。
-  预读将`整个数据分片加载至缓存`，保证 Range 线程完全读取缓存。
-  尽可能提高预读线程的速度，打满 IO。
-  建立`缓存池`，`循环复用`多个缓存片。
-  典型的生产者 / 消费者模型， 需要控制好缓存片的等待和通知。

### Range 架构设计

Range 的架构采用 8 个 DB 分片作为缓存池，从下图可以看出缓存片分为几种状态：

- 正在被读取
- 可以被 Range 线程读取
- Range 线程读完可以被重复利用的
- 未被使用的

![Range 顺序读架构](http://wangyapu.iocoder.cn/5E9C4993-32A0-45FA-8A73-DA1815F4B015.png)

当缓存池 8 个分片全部被填满，将重新从头开始，重复利用已被释放的缓存分片。针对 Range 范围查询采用 2 个预读线程持续读取数据分片到可用的缓存片中，Range 线程顺序从缓存中获取数据进行遍历。整个过程保证预读先行，通过等待/通知控制 Range 线程与预读线程齐头并进。

### 缓存片的使用注意点：

1. 将每个 DB 分片分为 2 段并发读取，每段 64m，提高分片预读的速度。
2. Range 线程需要等待预读线程完成分片的预读之后才可以进行读取。

![缓存片读取方式](http://wangyapu.iocoder.cn/8723D5B7-1DD4-4634-94D6-DD94002617D4.png)

可以看出这是一个典型的生产者消费者模型，需要做好预读线程和 Range 线程之间的协作：

1. 每个缓存片持有`一把锁和一个条件变量`，控制缓存片的等待和通知。
2. 每个缓存片采用`引用计数`以及 DB 分片号判断是否可用。
3. 预读线程将 DB 分片`分成 2 个段进行并发读取`，每段读完通知 Range 线程。Range 线程收到信号后判断所有段是否都读完，如果都读完则根据索引有序遍历整个缓存片。

### 缓存片数据结构

```cpp
class CacheItem {
public:
    CacheItem();

    ~CacheItem();

    void WaitAllDataSegmentReady(); // Range 线程等待当前缓存片所有段被读完

    uint32_t GetUnReadDataSegment(); // 预读线程获取还未读完的数据段

    bool CheckAllSegmentReady(); // 检测所有段是否都读完

    void SetDataSegmentReady(); // 设置当前数据段预读完成，并向 Range 线程发送通知

    void ReleaseUsedRef(); // 释放缓存片引用计数
    
    uint32_t mDBShardingIndex; // 数据库分片下标
    
    void *mCacheDataPtr; // 数据缓存
    
    uint32_t mUsedRef; // 缓存片引用计数 
    
    uint32_t mDataSegmentCount; // 缓存片划分为若干段
    
    uint32_t mFilledSegment; // 正在填充的数据段计数，用于预读线程获取分片时候的条件判断
    
    uint32_t mCompletedSegment; // 已经完成的数据段计数

    pthread_mutex_t mMutex;
    pthread_cond_t mCondition;
};
```

### Range 制胜点

在 Range 阶段如何保证预读线程能够充分利用 CPU 时间片以及打满 IO？采取了以下优化方案：

#### Busy Waiting 架构

Range 线程和预读线程的逻辑是非常相似的，Range 线程 Busy Waiting 地去获取缓存片，然后等待所有段都预读完成，遍历缓存片，释放缓存片。预读线程也是 Busy Waiting 地去获取缓存片，获取预读缓存片其中一段进行预读，通知 Range 该线程，释放缓存片。两者在获取缓存片唯一的区别就是 Range 线程每次获取不成功会 usleep 让出时间片，而预读线程没有这步操作，尽可能把 CPU 打满。

```cpp
// 预读线程
CacheItem *item = NULL;
while (true) {
    item = mCacheManager->GetCacheItem(mShardingItemIndex);
    if (item != NULL) {
        break;
    }
}

while (true) {
    uint32_t segmentNo = item->GetUnReadDataSegment();
    if (segmentNo == UINT32_MAX) {
        break;
    }
    uint64_t cacheOffset = segmentNo * EachSegmentSize;
    uint64_t dataOffset = cacheOffset + mStartPosition;
    pread64(mDataDirectFd, static_cast<char *>(item->mCacheDataPtr) + cacheOffset, EachSegmentSize, dataOffset);
    item->SetDataSegmentReady();
}
item->ReleaseUsedRef();

// Range 线程
CacheItem *item = NULL;
while (true) {
    item = mCacheManager->GetCacheItem(mShardingItemIndex);
    if (item != NULL) {
        break;
    }
    usleep(1);
}
item->WaitAllDataSegmentReady();

char key[8];
for (auto mKeyOffset = mKeyOffsets.begin(); mKeyOffset != mKeyOffsets.end(); mKeyOffset++) {
    if (unlikely(mKeyOffset->key == (mKeyOffset + 1)->key)) {
        continue;
    }
    uint32_t offset = mKeyOffset->offset - 1;
    char *ptr = static_cast<char *>(item->mCacheDataPtr) + offset * 4096;
    uint64ToString(mKeyOffset->key, key);
    PolarString str(key, 8);
    PolarString value(ptr, 4096);
    visitor.Visit(str, value);
}
item->ReleaseUsedRef();
```

因为比赛环境的硬件配置很高，这里使用忙等去压榨 CPU 资源可以取得很好的效果，实测优于条件变量阻塞等待。`然而在实际工程中这种做法是比较奢侈的，应该利用无锁的架构控制自旋等待的限度，如果自旋超过限定的阈值仍没有成功获得锁，应当使用传统的方式去挂起线程。`

#### 预读线程绑核

为了让预读线程的性能达到极致，`根据 CPU 亲和性的特点将 2 个预读线程进行绑核，减少线程切换开销，保证预读可以打满 CPU 以及 IO。`

```cpp
static bool BindCpuCore(uint32_t id) {
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(id, &mask);
    int ret = pthread_setaffinity_np(pthread_self(), sizeof(mask), &mask);
    return ret == 0;
}
```

## 整体工程优化

1. Key 转化为 uint64_t 借助 bswap 指令。
2. 尽可能加上分支预测 unlikely。
3. 对象复用，减少资源开销。
4. 位移、& 操作代替除法、取余运算。
5. 批量读取索引数据，做好边界处理。

## memcpy 4k 加速

利用 `SSE 指令集 对 memcpy 4k 进行加速`：

1. 直接操作汇编，使用 SSE 的 movdqu 指令。
2. 数据结构需要 8 字节对齐。
3. 针对 4k 的场景使用 16 个寄存器完成并行运算。

```cpp
inline void
mov256(uint8_t *dst, const uint8_t *src) {
    asm volatile ("movdqu (%[src]), %%xmm0\n\t"
                  "movdqu 16(%[src]), %%xmm1\n\t"
                  "movdqu 32(%[src]), %%xmm2\n\t"
                  "movdqu 48(%[src]), %%xmm3\n\t"
                  "movdqu 64(%[src]), %%xmm4\n\t"
                  "movdqu 80(%[src]), %%xmm5\n\t"
                  "movdqu 96(%[src]), %%xmm6\n\t"
                  "movdqu 112(%[src]), %%xmm7\n\t"
                  "movdqu 128(%[src]), %%xmm8\n\t"
                  "movdqu 144(%[src]), %%xmm9\n\t"
                  "movdqu 160(%[src]), %%xmm10\n\t"
                  "movdqu 176(%[src]), %%xmm11\n\t"
                  "movdqu 192(%[src]), %%xmm12\n\t"
                  "movdqu 208(%[src]), %%xmm13\n\t"
                  "movdqu 224(%[src]), %%xmm14\n\t"
                  "movdqu 240(%[src]), %%xmm15\n\t"
                  "movdqu %%xmm0, (%[dst])\n\t"
                  "movdqu %%xmm1, 16(%[dst])\n\t"
                  "movdqu %%xmm2, 32(%[dst])\n\t"
                  "movdqu %%xmm3, 48(%[dst])\n\t"
                  "movdqu %%xmm4, 64(%[dst])\n\t"
                  "movdqu %%xmm5, 80(%[dst])\n\t"
                  "movdqu %%xmm6, 96(%[dst])\n\t"
                  "movdqu %%xmm7, 112(%[dst])\n\t"
                  "movdqu %%xmm8, 128(%[dst])\n\t"
                  "movdqu %%xmm9, 144(%[dst])\n\t"
                  "movdqu %%xmm10, 160(%[dst])\n\t"
                  "movdqu %%xmm11, 176(%[dst])\n\t"
                  "movdqu %%xmm12, 192(%[dst])\n\t"
                  "movdqu %%xmm13, 208(%[dst])\n\t"
                  "movdqu %%xmm14, 224(%[dst])\n\t"
                  "movdqu %%xmm15, 240(%[dst])"
    :
    :[src] "r"(src),
    [dst] "r"(dst)
    : "xmm0", "xmm1", "xmm2", "xmm3",
            "xmm4", "xmm5", "xmm6", "xmm7",
            "xmm8", "xmm9", "xmm10", "xmm11",
            "xmm12", "xmm13", "xmm14", "xmm15", "memory");
}

#define mov512(dst, src) mov256(dst, src); \
        mov256(dst + 256, src + 256);

#define mov1024(dst, src) mov512(dst, src); \
        mov512(dst + 512, src + 512);

#define mov2048(dst, src) mov1024(dst, src); \
        mov1024(dst + 1024, src + 1024);

inline void memcpy_4k(void *dst, const void *src) {
    for (int i = 0; i < 16; ++i) {
        mov256((uint8_t *) dst + (i << 8), (uint8_t *) src + (i << 8));
    }
}
```

### String 黑科技

1. 目标：随机读阶段实现零拷贝。
2. 原因：由于 String 分内的内存不是 4k 对齐，所以没办法直接用于 DIO 读取，会额外造成一次内存拷贝。
3. 实现：使用自定义的内存分配器，确保分配出的内存 $string[0] 位置是 4k 对齐的，然后强转为标准的 String 供后续使用。

自定义实现了 STL Allocator 步骤：

- 申请内存空间
- 构造函数
- 析构函数
- 释放空间
- 替换  basic_string allocator
	
> 为了防止自定义 Allocator 分配的内存被外部接口回收，将分配的 string 保存在 threadlocal 里，确保引用计数不会变0。

```cpp
template<typename T>
class stl_allocator {
public:
    typedef size_t size_type;
    typedef std::ptrdiff_t difference_type;
    typedef T *pointer;
    typedef const T *const_pointer;
    typedef T &reference;
    typedef const T &const_reference;
    typedef T value_type;

    stl_allocator() {}
    ~stl_allocator() {}

    template<class U>
    struct rebind {
        typedef stl_allocator<U> other;
    };

    template<class U>
    stl_allocator(const stl_allocator<U> &) {}

    pointer address(reference x) const { return &x; }

    const_pointer address(const_reference x) const { return &x; }

    size_type max_size() const throw() { return size_t(-1) / sizeof(value_type); }

    pointer allocate(size_type n, typename std::allocator<void>::const_pointer = 0) {
        void *buffer = NULL;
        size_t mallocSize = 4096 + 4096 + (n * sizeof(T) / 4096 * 4096);
        posix_memalign(&buffer, 4096, mallocSize);
        return reinterpret_cast<pointer>(static_cast<int8_t *>(buffer) + (4096 - 24));
    }

    void deallocate(pointer p, size_type n) {
        free(reinterpret_cast<int8_t *>(p) - (4096 - 24));
    }

    void construct(pointer p, const T &val) {
        new(static_cast<void *>(p)) T(val);
    }

    void construct(pointer p) {
        new(static_cast<void *>(p)) T();
    }

    void destroy(pointer p) {
        p->~T();
    }

    inline bool operator==(stl_allocator const &a) const { return this == &a; }

    inline bool operator!=(stl_allocator const &a) const { return !operator==(a); }
};
typedef std::basic_string<char, std::char_traits<char>, stl_allocator<char>> String4K;}
```

## 失败的尝试

1. 随机读建立 4k 缓存池，一次缓存 16k 数据，限制缓存数量。但是实测命中率很低，性能下降。
2. PageCache 大块预读，通过 posix_fadvise 预读、释放缓存片来提高预读速度。最终结果优于直接使用 PageCache，仍无法超过 DIO。

## 最佳成绩

整个比赛取得的最佳成绩是 414.27s，每个阶段不可能都达到极限成绩，这里我列出了每个阶段的最佳性能。

![历史成绩记录](http://wangyapu.iocoder.cn/A2A02780-E848-445F-A65C-38AAED3A931C.png)


## 思考与展望

![第一版设计架构](http://wangyapu.iocoder.cn/more_arch.png)

这是比赛初期设计的架构，个人认为还是最初实现的一个分片对应单独一组数据文件的架构更好一些，每个分片还是分为索引文件、MergeFile 以及一组数据文件，数据文件采用定长分配，采用链表连接。这样方便扩容、多副本、迁移以及增量备份等等。当数据量太大没办法全量索引时，可以采用稀疏索引、多级索引等等。这个版本的性能评测稳定在 415~416s，也是非常优秀的。












