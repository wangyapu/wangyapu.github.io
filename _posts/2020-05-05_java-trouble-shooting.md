---
layout:     post
title:      "工具百宝箱（1）—— Java日常问题诊断方法"
subtitle:   "\"磨刀不误砍柴工\""
date:       2020-05-05 12:00:00
author:     "wangyapu"
header-img: "img/post-idea-debug.jpeg"
tags:
    - Java
---

# 工具百宝箱（1）—— Java日常问题诊断方法

## 日志检索

```bash

# 检索 exception 关键字或 error 关键字

grep -E "exception|error" app.log

# 显示关键字上下 10 行日志

grep -C 10 exception app.log

# 检索 2020-05-05 19:23~25  分钟段日志

grep '2020-05-05 19:2[3-5]' app.log

sed -n '/2020-05-05 19:23/,/2020-05-05 19:25/p' app.log

# 检索 2020-05-05 19:23:10~15 秒段日志

grep '2020-05-05 19:23:[10-15]' app.log

sed -n '/2020-05-05 19:23:10/,/2020-05-05 19:23:15/p' app.log

# 查询 /data 目录下大于 500m 的文件

find /data -type f -size +500M

```

## CPU & Load

### CPU User高 & Load高

User：CPU 在用户态空间（用户进程）的运行时间比例。

常见原因与排查方法：

1. 代码中存在非常消耗 CPU 的操作

    - 找出对应的 Java 进程 pid
    
        ```
        ps -ef | grep java
        ```
    
    - 找出 Java 进程中最消耗 CPU 的线程
    
        ```
        top -H -p <pid>
        ```
    
    - 将找出的线程 id 转换为 16 进制
    
      `printf "%x\n" pid`
    
    - jstack 获取 Java 的线程堆栈
    - 根据 16 进制的 id 从线程堆栈中找到相关的堆栈信息

2. 频繁 GC

    ```
    jstat -gcutil pid interval(ms)
    ```

### CPU System高 & Load高

System：CPU 在内核态空间的运行时间比例。例如内存分配、IO 读写、线程创建和切换等。

常见原因：

1. 并发锁竞争严重
2. 线程频繁切换

排查方法：

1. jstack 打印线程堆栈，整体统计线程状态，如 WAITING、TIMED_WAITING、BLOCKED。
2. pidstat -w <pid> 可以查看 CPU 上下文切换的状态。cswch/s 每秒自愿上下文切换次数：进程无法获取资源，如内存、IO 等；nvcswch/s 每秒非自愿上下文切换次数：时间片耗尽系统强制调度，如进程频繁争抢 CPU。

常用优化方法：

1. 优化锁粒度范围。
2. 使用无锁的数据结构，例如 RingBuffer。
3. 死锁的解决方法之一就是破坏请求和保持条件，所以等待/通知机制可以避免循环等待的 CPU 消耗。

### CPU Wait高 & Load高 & CPU利用率低

CPU is idle while waiting for an I/O operation to complete。The time the CPU spends in this state is shown by the wait statistic.

CPU 等待磁盘写入的时间。当 CPU 发起 IO 读写操作后，需要等待磁盘数据加载至内存。

常见原因：

- IO 操作频繁
- 资源未及时释放造成泄漏

排查方法：

```bash

# 查看设备的 IO 状态

iostat -x 2

参数含义：

rrqm/s : 每秒合并读操作的次数
wrqm/s: 每秒合并写操作的次数
r/s ：每秒读操作的次数
w/s : 每秒写操作的次数
rKB/s :每秒读取字节数
wKB/s: 每秒写入字节数
avgrq-sz：平均每次 IO 的数据大小，以扇区（512字节）为单位
avgqu-sz：平均 IO 请求队列长度
await：平均每个IO所需要的时间，包括在队列等待的时间和请求处理的时间。
r_wait：每个读操作平均所需要的时间，包括硬盘设备读操作的时间 + 内核队列中的时间。
w_wait: 每个写操平均所需要的时间，包括硬盘设备写操作的时间 + 队列中等待的时间。
%util: 每秒内用于 I/O 操作的时间占比

# 查找引起 iowait 高的进程

iotop

# 查看引起 iowait 高的具体文件

lsof -p <pid>

```

常用解决与优化方法：

- 有效控制资源数量，例如使用线程池等。 
- 了解磁盘特性是必要的，一般随机写转顺序写，同步写转异步写，IO 合并都可以可以得到较好的改善。
- 压缩 & Dirty Page 优化。Linux 操作系统中，当 Dirty Page 的大小达到总物理内存大小 10%，操作系统会进行刷盘但不阻塞系统调用的写线程，若达到物理内存大小的 20%，写线程会被阻塞。通过合适的压缩算法减少落盘数据的大小通常效果显著。
- 预读取和读缓存。
- Zero Copy。
- MMap 内存映射。
- 硬件红利。

## 内存 & GC

常用排查工具：

```bash

# 内存整体情况

free -m 

# JVM 堆内存占用排行

jmap -histo <pid> | head -n 30

# JVM 内存占用信息

jstat -gccause <pid> 1000 1000
jstat -gcutil <pid>

# dump内存

jmap -F -dump:live,file=/home/admin/heap.bin <pid>

```

### 进程消失 & OutOfMemoryError

进程消失大部分跟 OOM Killer 有关，可以根据 dmesg -T 查看系统日志。

OOM 常见原因与排查方法：

1. Java heap space / GC overhead limit exceeded / CMS GC、Full GC 频繁

    - 启动参数增加 -XX:+HeapDumpOnOutOfMemoryError，可在 OOM 时保存 Dump 文件。
    - jmap Dump 内存文件
    - MAT 分析 HeapDump 文件，找到内存泄漏的代码。例如一直在分配内存而引用未释放，缓存的数据结构未做保护限制等。

2. java.lang.StackOverflowError

    - 最常见：深层次或者无限递归。
    - 函数调用层级深。
    - 栈上分配较大的缓存。可适当调整 -Xss 大小。
    - 循环依赖。

3. PermGen Space / Meta space

    - 超过 -XX:MaxMetaspaceSize 设置的大小，如果确实不够用可以适当调整大小。
    - 排查代码中是否有类似 javassist 动态生成 class 的逻辑。

4. DirectBuffer

    - jmap -histo:live <pid> 手动触发 FullGC, 观察堆外内存是否被回收，如果正常回收很可能是因为堆外设置太小，可以通过 -XX:MaxDirectMemorySize 调整。当然这无法排除堆外内存缓慢泄漏的情况。
    - 堆外内存泄漏：Java 中是通过 ByteBuffer.allocateDirect 分配堆外内存。
        - 监控：MXBean 可以获取堆外内存使用情况。
        - Netty 自带检测工具：-Dio.netty.leakDetectionLevel=[检测级别]
        - Btrace 神器：ByteBuffer.allocateDirect
        - 堆外内存泄漏情况比较复杂，尽可能在本地模拟复现，二分定位也是个有效的版本。参考 [Netty堆外内存泄露排查盛宴](https://tech.meituan.com/2018/10/18/netty-direct-memory-screening.html)，排查思路非常值得借鉴学习。
    - 网络流量大，系统被短时间打爆。可以通过扩容、限流等手段解决。

5. unable to create new native thread

    - ps 等操作 Linux 会出现 Resource temporarily unavailable 错误。
    - 修改 /etc/security/limits.conf 配置。

6. MMAP

    - MAP_SHARED、MAP_PRIVATE、MAP_ANONYMOUS、MAP_NORESERVE，在不同的模式下表现行为不同，例如常用的 MAP_PRIVATE，如果程序没有及时释放资源会遭遇 OOM Killer。
    - MMAP 的文件数量超过了 vm.max_map_count 限制。


### 服务假死 & 超时

常见原因与解决方法：

1. 死锁

    - jstack 分析线程堆栈

2. 线程池耗尽

    - 扩大线程池数量
    - 避免处理过长的业务逻辑
    - 降低超时时间

3. 客户端或者服务端频繁 GC

    - YGC 频繁：可通过 -Xmn 调整新生代大小，-XX:SurvivorRatio 设置 SurvivorRatio 和 eden 区比例。应该清楚程序中对象的基本分布情况，如果存在大量朝生夕灭的对象，应适当调大新生代；反之应适当调大老年代。
    - YGC 时间长：常见有两个原因年轻代存活对象太多；老年代引用年轻代对象太多（跨代引用）。
    - Stop-The-World GC
        - System.gc() 或者 jmap -histo:live 主动触发
        - PermGen Space / Meta space空间不足
        - YGC 晋升到老年代的平均大小大于老年代剩余空间
        - CMS GC 中 Remark 时间长：可通过 CMSScavengeBeforeRemark 参数保证 Remark 前进行一次 Minor GC
        - promotion failed：对象晋升的目标区域没有足够的空间
        - concurrent mode failure：CMS GC的过程的同时业务进程申请老年代空间，而此时老年代空间不足导致。
        - 大对象分配失败，视 GC 算法不同可优化不同的参数，如 -XX:G1HeapRegionSize

4. 网络异常

    - 重点关注 CLOSE_WAIT，可能连接未关闭导致资源消耗殆尽。
    - netstat -nt 查看tcp相关连接状态、连接数以及发送队列和接收队列。正常的连接应该是 ESTABLISHED 状态，如果存在大量的 SYN_SENT 的连接，则需要看下防火墙规则。`如果 Recv-Q 或者 Send-Q 持续有大量包存在，意味着连接存在瓶颈或者程序存在 bug。`

5. 客户端或者服务端 CPU 使用率高，排查方法同 CPU 相关小节。 
6. 对象未序列化，检查是否实现 Serializable 序列化接口等。

## ClassLoader & Jar包冲突

JVM 装载 jar 包时，加载顺序完全取决于操作系统，所以在不同的环境有可能表现出来的问题不一样。

常见问题：

- NoSuchMethodException
- ClassNotFoundException
- NoClassDefFoundError
- ClassCastException

常用解决方法：

```bash

# 打印所有依赖

mvn dependency:tree

# 只查看关注的依赖

mvn dependency:tree -Dverbose -Dincludes=<groupId>:<artifactId>

# 观察类的加载过程

-XX:+TraceClassLoading

-verbose:class

```

> 推荐 IDEA 插件 Maven Helper，一些基本的冲突问题可以迎刃而解。



