---
layout:     post
title:      "docker和paas基础分享"
subtitle:   "\"docker镜像、文件系统、paas架构设计\""
date:       2016-04-01 12:00:00
author:     "wangyapu"
header-img: "img/bg_fwa.jpg"
tags:
    - docker
    - 工作分享
---

# Docker和PaaS的基础分享

## docker基本应用

    docker --help
    
![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/docker_command.png)


## docker镜像与docker文件系统

镜像是docker的灵魂所在，也是软件交付的产品。

![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/docker_system.png)


    FROM ubuntu:14.04
    ADD run.sh /
    VOLUME /data
    CMD ["./run.sh"]
    
### rootfs

容器启动时内部进程可见的文件系统视角。

![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/rootfs.png)

### union mount

一种文件系统挂载方式，允许同一时刻多种文件系统叠加在一起，以一种文件系统的形式呈现出来，目录也是多种文件系统合并之后的目录。

### copy on wirte

用户视角所看到的是合并后的文件系统，用户不知道哪些内容是只读或是可读写的。linux内核依然可以区分两者。

假如我更改了/etc下面的内容，rootfs和可读写文件系统均存在，是否会报错呢？
No！

linux保证rootfs只读，cow机制会先将需要更改的文件复制到可读写层，然后进行更改，这样rootfs和可读写层就存在两个同名的文件，但是cow机制会保证用户只会看到可读写层的文件。

### image & layer

image解决复用，images之间存在父子关系。

rootfs中每个只读的image都可以称为一个layer。

![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/layer.png)

## cgroup、namespace与docker（具体见上次分享）

[go与docker分享](http://wangyapu0714.github.io/2016/01/28/go_docker_share/)


## PaaS架构设计

![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/init_instance.png)


## Docker结合Jenkins的思路


![image](https://github.com/wangyapu0714/my_post/raw/master/md_pic/docker/docker_Jenkins.png)

