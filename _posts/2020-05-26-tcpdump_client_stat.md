---
layout:     post
title:      "tcpdump分析可疑客户端流量"
subtitle:   "\"整理日常的点点滴滴\""
date:       2020-05-24 12:00:00
author:     "wangyapu"
header-img: "img/post-tcpdump-client-stat.jpg"
tags:
    - 工具
---

# tcpdump分析可疑客户端流量

## tcpdump常用命令

1. 常用参数：

    -i：指定需要的网
    
    -s：抓取数据包时默认抓取长度为 68 字节，加上-s 0后可以抓到完整的数据包
    
    -w：监听的数据包写入指定的文件

2. 示例

    ```bash
    tcpdump -i eth1 host 10.1.1.1  // 抓取所有经过eth1，目的或源地址是10.1.1.1的网络数据包 
    
    tcpdump -i eth1 src host 10.1.1.1  // 源地址
    
    tcpdump -i eth1 dst host 10.1.1.1  // 目的地址
    ```

    如果想使用 wireshark 分析 tcpdump 的包，需要加上是 -s 参数：

    ```bash
    tcpdump -i eth0 tcp and port 80 -s 0 -w traffic.pcap
    ```

## tcpdump分析客户端流量

话不多说，直接上脚本吧，有兴趣的朋友可以收藏，说不定哪天用上了呢？

tcpdump_client_stat.sh

```bash

#!/usr/bin/env bash

net=$1    # 网卡 eth0/any
port=$2   # 端口 8080/22/...
count=$3  # 网络包个数 1000000

tcpdump -i $net -nn "tcp port $port" -c $count | grep "IP" | grep ">" | grep "$port" | awk '{print $3}' | grep -v "$port" | awk -F . -v OFS=. '{print $1, $2, $3, $4}' | sort -n | uniq -c

```

bash tcpdump_client_stat.sh any 8080 1000000

输入结果：

```
1000 127.0.0.1
800000 127.0.0.2
500 127.0.0.3
```

显而易见，127.0.0.2 可能存在可疑流量。


