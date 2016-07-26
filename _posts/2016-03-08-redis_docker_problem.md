---
layout:     post
title:      "线上Redis-Docker集群出现物理机崩溃的一次问题记录"
subtitle:   "\"redis、docker、devicemapper问题定位\""
date:       2016-03-08 12:00:00
author:     "wangyapu"
header-img: "img/redis_docker_problem/redis_logo.jpg"
tags:
    - docker
---

# Redis-Docker集群的一次踩坑记录

## 项目背景

    点评线上redis的docker集群用于生产线上有一段时间，也算是一个全新的尝
    试，利用docker的优势实现搞笑的redis实例创建和调度。
    
## 问题描述

    最近一段时间，有几台出现崩溃问题，机器load不断升高，有的高达5000多，
    诸多线程处于D状态,很多请求以及linux命令出现卡死状态。

### 现象1：很多线程处于D状态

    $ > dmesg

    INFO: task jbd2/dm-20-8:198571 blocked for more than 120 seconds.
      Not tainted 2.6.32-431.29.2.el6.x86_64 #1
    "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
    jbd2/dm-20-8  D 0000000000000009     0 198571      2 0x00000080
     ffff88316ac77b30 0000000000000046 0000000000000000 ffffffffa000169d
     ffff88316ac77aa0 ffffffff810149b9 ffff88316ac77ae0 0000000000000286
     ffff88316acff058 ffff88316ac77fd8 000000000000fbc8 ffff88316acff058
    Call Trace:
     [<ffffffffa000169d>] ? __map_bio+0xad/0x140 [dm_mod]
     [<ffffffff810149b9>] ? read_tsc+0x9/0x20
     [<ffffffff81120d00>] ? sync_page+0x0/0x50
     [<ffffffff81529c73>] io_schedule+0x73/0xc0
     [<ffffffff81120d3d>] sync_page+0x3d/0x50
     [<ffffffff8152a73f>] __wait_on_bit+0x5f/0x90
     [<ffffffff81120f73>] wait_on_page_bit+0x73/0x80
     [<ffffffff8109c530>] ? wake_bit_function+0x0/0x50
     [<ffffffff81136fc5>] ? pagevec_lookup_tag+0x25/0x40
     [<ffffffff8112139b>] wait_on_page_writeback_range+0xfb/0x190
     [<ffffffff8112145f>] filemap_fdatawait+0x2f/0x40
     [<ffffffffa00c5e59>] jbd2_journal_commit_transaction+0x7e9/0x1500 [jbd2]
     [<ffffffff810096f0>] ? __switch_to+0xd0/0x320
     [<ffffffff81085f3b>] ? try_to_del_timer_sync+0x7b/0xe0
     [<ffffffffa00cba48>] kjournald2+0xb8/0x220 [jbd2]
     [<ffffffff8109c4b0>] ? autoremove_wake_function+0x0/0x40
     [<ffffffffa00cb990>] ? kjournald2+0x0/0x220 [jbd2]
     [<ffffffff8109c106>] kthread+0x96/0xa0
     [<ffffffff8100c20a>] child_rip+0xa/0x20
     [<ffffffff8109c070>] ? kthread+0x0/0xa0
     [<ffffffff8100c200>] ? child_rip+0x0/0x20

### 现象2：磁盘并没有写满，还有很多空间

![image](http://wangyapu0714.github.io/img/redis_docker_problem/dm_disk_total.png)

    因为devicemapper配了nodiscard，docker info看到的data 
    space并不一定是实例真正使用的空间，所以利用脚本分别统计了每个实例所用的磁盘空间，
    发现每个实例所使用的磁盘空间大概都不到10g，均是redis aof产出的持久化文件。


### 现象3：暂停/删除实例没反应

    $ > docker stop {containerId}
    $ > docker rm -f {containerId}


### 现象4：并不是实例太多造成的偶然现象

    $ > service docker stop
    $ > docker start {containerId}
    只启动一个redis实例，观察top，load持续上涨，依然出现现象3的问题。
    

### 现象5：devicemapper的使用空间显示已满

```bash
$ > docker info 
Containers: 21
Images: 79
Storage Driver: devicemapper
Pool Name: docker-8:2-4851000-pool
Pool Blocksize: 65.54 kB
Data file: /dev/sda3
Metadata file: /var/lib/docker/devicemapper/devicemapper/metadata
Data Space Used: 1.677 TB
Data Space Total: 1.677 TB
Metadata Space Used: 852.1 MB
Metadata Space Total: 10.74 GB
Library Version: 1.02.82-git (2013-10-04)
Execution Driver: native-0.2
Kernel Version: 2.6.32-431.29.2.el6.x86_64
```

## 问题排查


---

>  **猜想1：可能跟devicemapper配置了nodiscard有关，有待验证。**

    如果不配置nodiscard，使用Docker时内核会随机crash。具体问题可以参考蘑菇街
    的博客：[link](http://mogu.io/docker_crash-79)
    点评禁用discard之后，线上运行近一年没有出现类似现象，问题规避成功。

---

>  **猜想2：因为devicemapper配置了nodiscard，是不是因为每次Redis产生的aof文件并没有被回收掉。**

**实验验证1、2**

    docker的启动参数配置成--storage-opt dm.loopdatasize=85G --storage-opt 
    dm.basesize=80G，本地只启动一个容器，执行一个脚本，脚本内容如下：

```bash
#!/bin/bash
cd /tmp
while true
do
    dd if=/dev/zero of=/tmp/hello.txt bs=1G count=20
    echo "creat hello.txt success"
    rm -rf hello.txt
    echo "rm hello.txt success"
    sleep 1
done
```

**实验结果**

    docker info观察Data Space Used，很快上涨到约80G，
    但是脚本依然可以持续运行。说明已经被删除的文件占用的空间可以被重新使用。

---
    
>  **猜想3：出问题的宿主机都是磁盘超配，是否有关系？**

>  **猜想4：宿主机上有约20个Redis实例，发现并不是所有实例都挂掉，有一部分还在正常运行，新申请的Redis实例基本都无法正常工作，是否跟创建时间有关系？**

![image](http://wangyapu0714.github.io/img/redis_docker_problem/docker_ps.jpg)

>  **猜想5：有可能老的Redis实例已经把磁盘吃满，新申请的实例无法重复使用已被旧Redis实例所申请过且已经被删除的文件所占用的空间资源。简单说就是，容器A使用过的空间资源中即使文件被删除，容器B也无法重复利用，好像容器A独享一样。**

**实验验证3、4、5**

    docker的启动参数依然配置成--storage-opt dm.loopdatasize=85G --storage-opt
    dm.basesize=80G，本地启动两个容器A和B，首先在容器A里运行上述实验中同样的
    脚本，在Data Space Used停留在80G不再上涨后，进入容器B，执行：
```bash
dd if=/dev/zero of=/tmp/hello.txt bs=1G count=20
```
    
**实验结果**

    在容器B里执行命令时，当Data Space Used上涨到约85G之后进程卡死，top观察
    load持续升高，此时容器docker run一个新的容器也是卡死状态。基本复现了线上
    Redis集群出现的问题。

![image](http://wangyapu0714.github.io/img/redis_docker_problem/docker_info.jpg)

![image](http://wangyapu0714.github.io/img/redis_docker_problem/dd_file.jpg)

![image](http://wangyapu0714.github.io/img/redis_docker_problem/docker_run.jpg)
    
    
## 结论

    出现问题的原因与devicemapper的nodiscard、Redis Aof机制以及磁盘超配相关，
    但是nodiscard参数绝对不能弃用的，建议的解决方案是：
    1. Redis Aof产生的文件存储到外挂磁盘
    2. 重新规划Redis集群的磁盘使用情况，禁止超配，再做观察。