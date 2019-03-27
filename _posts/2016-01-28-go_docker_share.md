---
layout:     post
title:      "Go and Docker"
subtitle:   " \"go语言和docker基本知识必备\""
date:       2016-01-28 12:00:00
author:     "wangyapu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 工作分享
    - docker
---



# Go & Docker

-----------------

![](http://wangyapu.github.io/img/go_docker_share/go_logo.png)
![](http://wangyapu.github.io/img/go_docker_share/docker_logo.png)

# 分享大纲

## Go

- Why Go
- 工程结构
- 快速入门
- 并发模型

## Docker

- Namespace
- Cgroup

# Go

# Why Go？

## 互联网、云计算时代的C语言

- Google出品，开源产品，业界先驱开创
- 跨平台支持
- 编译型语言，但语法趋于脚本化，非常简洁
- 全自动的垃圾回收机制
- 先进、原生的并发支持
- 函数式编程
- 内容完善，覆盖软件生命周期
- 杀手级应用docker、etcd等


## 没有完美且万能的语言

- 垃圾回收完善（1.5有重大改善）
- 分布式计算中成熟度不如Erlang
- 程序运行速度目前还不及C
- 第三方库数量远不及主流语言
- 不支持泛型
- 需要熟悉并发模型细节，不然容易踩坑


# Go语言工程结构

## Go命令行工具 —— go help

- 代码格式化
- 代码质量分析和修复
- 单元测试与性能测试
- 工程构建  
- 代码文档的提取和展示
- 依赖包管理

## 工程组织结构

    Go命令行工具的革命性之处在于彻底消除了工程文件的概念,完全用目录结构和包名
    来推导工程结构和构建顺序。

![go工程结构](http://wangyapu.github.io/img/go_docker_share/go_project_struct.png)

    如何定位到对应的源代码靠GOPATH
    
## 文档管理

    基本做到注释即文档，相比Javadoc更加简单，没有一些复杂的注释语法。
    godoc package

## 跨平台开发

    Linux、Unix支持最佳。

    最合适的分发方式是直接打包源代码包并进行分发


# Go特性一览

## 变量 var 

## 定义

    var v1 int
    var v2 string 
    var v3 [10]int 
    var v4 []int 
    var v5 struct {
    f int 
    }
    var v6 *int
    var v7 map[string]int 
    var v8 func(a int) int
    var (
     v1 int
     v2 string
    )

## 赋值

    var v1 int = 10 // 正确的使用方式1
    var v2 = 10 // 正确的使用方式2,编译器可以自动推导出v2的类型 
    v3 := 10 // 正确的使用方式3,编译器可以自动推导出v3的类型

## 匿名变量和多返回值

    func GetVar() (var1, var2, var3 string) { 
        return "var1", "var2", "var3"
    }
    _, _, var3 := GetName()
    
## 数据类型

## 基本数据类型

类型 | 字节数 | 备注 
---|---|---
bool | 1 | 
byte | 1 | uint8的别名类型
rune | 4 | int32的别名类型
int/uint | 4/8 | 实际宽度依赖于当前的计算架构
int8/uint8 | 1 | 
int16/uint16 | 2 | 

------------------------

类型 | 字节数 | 备注 
---|---|---
int32/uint32 | 4 | 
int64/uint64 | 8 | 
float32 | 4 | 
float64 | 8 | 
complex64 | 8 | 
complex128 | 16 | 
string | - | 

## 高级类型

类型 | 零值 | 备注
---|---|---|---
Array | []int{} | 
Slice | []int | 切片，类似于java里list 
Map | nil | 
channel | nil | 通道，用于数据传递 

-----------------------

类型 | 零值 | 备注
---|---|---|---
uintptr | nil | 原始指针类型 
Function | nil |  
Interface | nil | 一切皆为接口 
Struct | xxx{} | 结构体

--------------------

```go
type Animal interface {
     Name() string
}

//结构体Cat实现了接口类型Animal
type Cat struct {
     name string
}

func (cat Cat) Name() string {
     return cat.name
}
```

    
## 匿名函数与闭包

## 匿名函数

Go里面函数可以像普通变量一样被传递或使用。

```go
f := func(x, y int) int { 
    return x + y
}

f := func(x, y int) int { 
    return x + y
}(2,3)
```    

## 闭包

闭包的价值在于能够被函数动态创建和返回。

```go
func main() {
	var j int = 5
	a := func() (func()) {
		var i int = 10
		i++
		return func() {
			fmt.Printf("i, j: %d, %d\n", i, j)
		}
	}()
	a()
	j *= 2
	a()
}
```

## 错误处理

## error

```go
func Foo(param int) (n int, err error) { 
    // ...
}

n, err := Foo(0)
if err != nil { 
    // 错误处理
} else {
	// 使用返回值n
}
```
    
## defer

- 函数需要退出时,就必须使用goto语句跳转到指定位置先完成资源清理工作。

- 一个函数中可以存在多个defer语句,defer语句的调用是遵照先进后出的原则,即最
后一个defer语句将最先被执行。

```go
func TestChannelBufferTimeout(t *testing.T) {
	expected := "11"

	buf := &ChannelBuffer{make(chan []byte, 1)}
	defer buf.Close()

	go func() {
		time.Sleep(100 * time.Millisecond)
		io.Copy(buf, strings.NewReader(expected))
	}()
}
```
    
## panic && recover

- 当在一个函数执行过程中调用panic()函数时,正常的函数执行流程将立即终止,但函数中之前使用defer关键字延迟执行的语句将正常展开执行。

- recover()函数用于终止错误处理流程,经常与defer结合使用。

```go
func main() {

	Try(func() {
		panic("error happens")
	}, func(error interface{}) {
		fmt.Println(error)
	})

}

func Try(fun func(), handler func(interface{})) {
	defer func() {
		if err := recover(); err != nil {
			handler(err)
		}
	}()
	fun()
}
```
    
## 反射、语言交互、同步等等
    
# 并发模型

## 并发和并行

- 并发在于结构，并行在执行。

- 并行：单核CPU是永远不可能并行的。

- 单个CPU多线程实现的伪并行，是一个CPU在多个程序之间切换，让我们以为是同时
执行。而实际上就是线性执行；

- 多个CPU同时负责多个线程才是并行。

- 并发：只要你的代码写了多线程就是并发。


## 多线程模型

- **python、lua （用户级线程模型M:1）**

    ![用户级线程模型](http://wangyapu.github.io/img/go_docker_share/python_model.png)

- **Java （内核级线程模型1：1）**

    ![内核级线程模型](http://wangyapu.github.io/img/go_docker_share/java_model.png)
    
    
- **Golang （两级线程模型M:N）**

    ![两级线程模型](http://wangyapu.github.io/img/go_docker_share/golang_model.png)

## 协程

    何为协程？(进程——线程——协程关系)

    一个CPU在做一件很耗时的事,突然来了第二个任务，怎么办？
    现代计算机多个核，一个人一个进程，Happy！
    然而任务数多于核数了怎么办？
    

生产者消费者模型的基于抢占式多线程编程实现：

    // 消费者线程
    loop
      lock(q)
      get item from q
      unlock(q)
      if item
        use this item
      else
        sleep 
    // 生产者线程
    loop
      create new items
      lock(q)
      add the items to q
      unlock(q)
 

基于协程的生产者消费者模型实现：

    // 生产者协程
    loop
      while q is not full
        create some new items
        add the items to q
      yield to consume
    // 消费者协程
    loop
      while q is not empty
        remove some items from q
        use the items
      yield to produce

-----------------

本质上协程就是用户空间下的线程。不受操作系统管理的独立控制流。

在用户态为每个协程维护调用上下文，以前都是内核帮你维护了，现在要用户态自
己来，好处呢就是自己可以控制协程的状态，start, stop, 切换到另外一个协程。
    
## channel

- channel是go语言用于goroutine间的通信方式。
- 不要通过共享内存来通信,而应该通过通信来共享内存。

### 非缓冲channel && 缓冲channel

- 无缓冲： 不仅仅是向 c1 通道放 1，而是一直要等有别的协程 <-c1 
接手了这个参数，那么c1<-1才会继续下去，要不然就一直阻塞着。
具有同步传递的特性。

- 有缓冲： c2<-1 则不会阻塞，因为缓冲大小是1(其实是缓冲大小为0)，只有当放第
二个值的时候，第一个还没被人拿走，这时候才会阻塞。

```go
func main() {
	jobs := make(chan int,1)
	done := make(chan bool)
	go func() {
		for i := 1;; i++ {
			j, more := <-jobs
			time.Sleep(time.Second)
			if more {
				fmt.Println("received job", j)
			} else {
				fmt.Println("received all jobs")
				done <- true
				return
			}
		}
	}()
	for j := 1; j <= 3; j++ {
		jobs <- j
		fmt.Println("sent job", j)
	}

	close(jobs)
	fmt.Println("sent all jobs")

	<-done
}
```
    
### select机制

监控一系列的文件句柄,一旦其中一个文件句柄发生了IO动作,该select()调用就会
被返回。后来该机制也被用于实现高并发的Socket服务器程序。

Go语言直接在语言级别支持select关键字，用于处理异步IO问题。

常见用法：超时处理。

```go
package main

import (
	"time"
	"fmt"
)

func main() {
	DoSomething()
	fmt.Println("task end")
}

func DoSomething() error {
	done := make(chan error)
	go func() {
		done <- DoThing()
	}()

	select {
	case <-time.After(time.Second * 10):
		Clear()
		return fmt.Errorf("Timeout")
	case err := <-done:
		if err != nil {
			err = fmt.Errorf("call failed, err %v", err)
		}
		return err
	}
}

func DoThing() (error) {
	fmt.Println("do some thing")
	time.Sleep(time.Second * 14)
	return nil
}

func Clear() {
	fmt.Println("clear job")
	time.Sleep(time.Second * 5)
}
```

### 多核并行

```go
import (
	"fmt"
	"runtime"
	"time"
)

var n int64 = 10000000000
var h float64 = 1.0 / float64(n)

func f(a float64) float64 {
	return 4.0 / (1.0 + a * a)
}

func chunk(start, end int64, c chan float64) {
	var sum float64 = 0.0
	for i := start; i < end; i++ {
		x := h * (float64(i) + 0.5)
		sum += f(x)
	}
	c <- sum * h
}

func main() {

	start := time.Now()

	var pi float64
	np := runtime.NumCPU()
	runtime.GOMAXPROCS(np)
	c := make(chan float64, np)

	for i := 0; i < np; i++ {
		go chunk(int64(i) * n / int64(np), (int64(i) + 1) * n / int64(np), c)
	}

	for i := 0; i < np; i++ {
		pi += <-c
	}

	fmt.Println("Pi: ", pi)

	end := time.Now()

	fmt.Printf("spend time: %vs\n", end.Sub(start).Seconds())
}
```

    
# Docker

# Namespace

--------------------

    如果要自己实现一个资源隔离的容器，应该从哪些方面下手呢？也许你第一反应可
    能就是chroot命令，这条命令给用户最直观的感觉就是使用后根目录/的挂载点切换
    了，即文件系统被隔离了。内部的文件系统无法访问外部的内容。

**一共六种：**

--------------------

Namespace | 系统调用参数 | 隔离内容
---|---|---
UTS | CLONE_NEWUTS | 主机名与域名
IPC | CLONE_NEWIPC | 信号量、消息队列和共享内存
PID | CLONE_NEWPID | 进程编号
Mount | CLONE_NEWNS | 挂载点（文件系统）
User | CLONE_NEWUSER | 用户和用户组
Network | CLONE_NEWNET | 网络设备、网络栈、端口等等

--------------------

    用户就可以在/proc/[pid]/ns文件下看到指向不同namespace号的文件,[xxxx]即为namespace号。

    ls -l /proc/$$/ns
    
- clone() – 实现线程的系统调用，创建一个新的进程，通过设计上述参数达到隔离。
- unshare() – 使某进程脱离某个namespace
- setns() – 把某进程加入到某个namespace

## 系统调用clone

```cpp    
#define _GNU_SOURCE

#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {
    printf("Container - inside the container!\n");
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    /*
     * 在一个进程终止或者停止时，将SIGCHLD信号发送给其父进程。按系统默认将忽略此信号。如果父进程希望被告知其子系统的这种状态，
     * 则应捕捉此信号。信号的捕捉函数中通常调用wait函数以取得进程ID和其终止状态。
     */
    int container_pid = clone(container_main, container_stack + STACK_SIZE, SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
    
## UTS

```cpp
#define _GNU_SOURCE

#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {
    printf("Container - inside the container!\n");

    /*
     * int gethostname(char *name, size_t len);
       int sethostname(const char *name, size_t len);
     */
    sethostname("container", 10); /* 设置hostname */
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack + STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}

```
    
## IPC

```cpp
#define _GNU_SOURCE

#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {
    printf("Container - inside the container!\n");

    /*
     * int gethostname(char *name, size_t len);
       int sethostname(const char *name, size_t len);
     */
    sethostname("container", 10); /* 设置hostname */
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {
    printf("Parent - start a container!\n");
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack + STACK_SIZE, CLONE_NEWUTS | CLONE_NEWIPC | SIGCHLD,
                              NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
    
## PID

```cpp
#define _GNU_SOURCE

#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {

    printf("Container [%5d] - inside the container!\n", getpid());
    /*
     * int gethostname(char *name, size_t len);
       int sethostname(const char *name, size_t len);
     */
    sethostname("container", 10); /* 设置hostname */
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {

    printf("Parent [%5d] - start a container!\n", getpid());
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack + STACK_SIZE,
                              CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | SIGCHLD,
                              NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
    
## Mount

```cpp
#define _GNU_SOURCE

#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

/* 定义一个给 clone 用的栈，栈大小1M */
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];

char *const container_args[] = {
        "/bin/bash",
        NULL
};

int container_main(void *arg) {

    printf("Container [%5d] - inside the container!\n", getpid());
    /*
     * int gethostname(char *name, size_t len);
       int sethostname(const char *name, size_t len);
     */
    sethostname("container", 10); /* 设置hostname */
    system("mount -t proc proc  /proc");
    /* 直接执行一个shell，以便我们观察这个进程空间里的资源是否被隔离了 */
    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {

    printf("Parent [%5d] - start a container!\n", getpid());
    /* 调用clone函数，其中传出一个函数，还有一个栈空间的（为什么传尾指针，因为栈是反着的） */
    int container_pid = clone(container_main, container_stack + STACK_SIZE,
                              CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNS | SIGCHLD,
                              NULL);
    /* 等待子进程结束 */
    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
    
## User

GID为GroupId，即组ID，用来标识用户组的唯一标识符
UID为UserId，即用户ID，用来标识每个用户的唯一标示符

```cpp
#define _GNU_SOURCE

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/mount.h>
#include <sys/capability.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024 * 1024)

static char container_stack[STACK_SIZE];
char *const container_args[] = {
        "/bin/bash",
        NULL
};

int pipefd[2];

void set_map(char *file, int inside_id, int outside_id, int len) {
    FILE *mapfd = fopen(file, "w");
    if (NULL == mapfd) {
        perror("open file error");
        return;
    }
    fprintf(mapfd, "%d %d %d", inside_id, outside_id, len);
    fclose(mapfd);
}

void set_uid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/uid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

void set_gid_map(pid_t pid, int inside_id, int outside_id, int len) {
    char file[256];
    sprintf(file, "/proc/%d/gid_map", pid);
    set_map(file, inside_id, outside_id, len);
}

int container_main(void *arg) {

    printf("Container [%5d] - inside the container!\n", getpid());

    printf("Container: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
           (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());

    /* 等待父进程通知后再往下执行（进程间的同步） */
    char ch;
    close(pipefd[1]);
    read(pipefd[0], &ch, 1);

    printf("Container [%5d] - setup hostname!\n", getpid());
    //set hostname
    sethostname("container", 10);

    //remount "/proc" to make sure the "top" and "ps" show container's information
    mount("proc", "/proc", "proc", 0, NULL);

    execv(container_args[0], container_args);
    printf("Something's wrong!\n");
    return 1;
}

int main() {
    const int gid = getgid(), uid = getuid();

    printf("Parent: eUID = %ld;  eGID = %ld, UID=%ld, GID=%ld\n",
           (long) geteuid(), (long) getegid(), (long) getuid(), (long) getgid());

    pipe(pipefd);

    printf("Parent [%5d] - start a container!\n", getpid());

    int container_pid = clone(container_main, container_stack + STACK_SIZE,
                              CLONE_NEWUTS | CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUSER | SIGCHLD, NULL);


    printf("Parent [%5d] - Container [%5d]!\n", getpid(), container_pid);

    set_uid_map(container_pid, 0, uid, 1);
    set_gid_map(container_pid, 0, gid, 1);

    printf("Parent [%5d] - user/group mapping done!\n", getpid());

    /* 通知子进程 */
    close(pipefd[1]);

    waitpid(container_pid, NULL, 0);
    printf("Parent - container stopped!\n");
    return 0;
}
```
    
## Network
    
## 点评内部网络模式

- docker原有网络

![docker网络](http://wangyapu.github.io/img/go_docker_share/nat_net.png)

---------------------

- 点评的网络（docker、kvm通用）

![点评内部网络](http://wangyapu.github.io/img/go_docker_share/host_net.png)


## 模拟network namespace

```bash
brctl addbr lxcbr0
brctl stp lxcbr0 off
ifconfig lxcbr0 192.168.10.1/24 up
ip netns add ns1 
ip netns exec ns1 ip link set dev lo up 
ip link add veth-ns1 type veth peer name lxcbr0.1
ip link set veth-ns1 netns ns1
ip netns exec ns1 ip link set dev veth-ns1 name eth0 
ip netns exec ns1 ifconfig eth0 192.168.10.11/24 up
brctl addif lxcbr0 lxcbr0.1
ifconfig lxcbr0 up
ifconfig lxcbr0.1 up
ip netns exec ns1 ip route add default via 192.168.10.1
mkdir -p /etc/netns/ns1
echo "nameserver 8.8.8.8" > /etc/netns/ns1/resolv.conf
```

# Cgroup

## Cpu限制

```bash
cd /sys/fs/cgroup/cpu
sudo mkdir wyp
cat /sys/fs/cgroup/cpu/wyp/cpu.cfs_quota_us
echo 40000 > /sys/fs/cgroup/cpu/wyp/cpu.cfs_quota_us
top
echo pid >> /sys/fs/cgroup/cpu/wyp/tasks
top
```

## 内存限制

```bash
mkdir /sys/fs/cgroup/memory/wyp
echo 128k > /sys/fs/cgroup/memory/wyp/memory.limit_in_bytes
echo [pid] > /sys/fs/cgroup/memory/wyp/tasks
```

## 磁盘IO限制

```bash
mkdir /sys/fs/cgroup/blkio/wyp
sudo dd if=/dev/sda1 of=/dev/null
echo '8:0 1048576' >  /sys/fs/cgroup/blkio/wyp/blkio.throttle.read_bps_device
echo [tid] > /sys/fs/cgroup/blkio/wyp/tasks
```



