---
layout:     post
title:      "天池中间件Golang版Service Mesh思路分享"
subtitle:   "\"Service Mesh for Dubbo (Golang)\""
date:       2018-06-12 12:00:00
author:     "wangyapu"
header-img: "img/read_notes/pexels-photo-715842.jpeg"
tags:
    - 架构
---

# 天池中间件大赛Golang版Service Mesh思路分享

   这次天池中间件性能大赛初赛和复赛的成绩都正好是`第五名`，出乎意料的是作为Golang是这次比赛的“稀缺物种”，这次在前十名中我也是侥幸存活在C大佬和Java大佬的中间。
   
   关于这次初赛《Serviwangyapu.iocoder.cnce Mesh for Dubbo》难度相对复赛《单机百万消息队列的存储设计》简单一些，最终成绩是`6983分`，因为一些Golang的小伙伴在正式赛512并发压测的时候大多都卡在6000分大关，这里主要跟大家分享下我在这次Golang版本的一些心得和踩过的坑。
   
   由于工作原因实在太忙，比赛只有周末的时间可以突击，下一篇我会抽空整理下复赛《单机百万消息队列的存储设计》的思路方案分享给大家，个人感觉实现方案上也是决赛队伍中比较特别的。
   
   
##  What's Service Mesh？

   Service Mesh另辟蹊径，实现服务治理的过程不需要改变服务本身。通过以proxy或sidecar形式部署的 Agent，所有进出服务的流量都会被Agent拦截并加以处理，这样一来微服务场景下的各种服务治理能力都可以通过Agent来完成，这大大降低了服务化改造的难度和成本。而且Agent作为两个服务之间的媒介，还可以起到协议转换的作用，这能够使得基于不同技术框架和通讯协议建设的服务也可以实现互联互通，这一点在传统微服务框架下是很难实现的。
   
   下图是一个官方提供的一个评测框架，整个场景由5个Docker 实例组成（蓝色的方框），分别运行了 etcd、Consumer、Provider服务和Agent代理。Provider是服务提供者，Consumer是服务消费者，Consumer消费Provider提供的服务。Agent是Consumer和Provider服务的代理，每个Consumer或 Provider都会伴随一个Agent。etcd是注册表服务，用来记录服务注册信息。从图中可以看出，Consumer 与Provider 之间的通讯并不是直接进行的，而是经过了Agent代理。这看似多余的一环，却在微服务的架构演进中带来了重要的变革。
   
   ![](http://wangyapu.iocoder.cn/15324462480023.jpg)

有关Service Mesh的更多内容，请参考下列文章：

- [What Is a Service Mesh?](https://www.nginx.com/blog/what-is-a-service-mesh/)
- [What’s a service mesh? And why do I need one? (中文翻译)](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)
- [聊一聊新一代微服务技术 Service Mesh](https://www.sdnlab.com/20363.html)

## 赛题要求

- 服务注册和发现
- 协议转换（这也是实现不同语言、不同框架互联互通的关键）
- 负载均衡
- 限流、降级、熔断、安全认证（不作要求）

当然Agent Proxy最重要的就是通用性、可扩展性强，通过增加不同的协议转换可以支持更多的应用服务。`最后Agent Proxy的资源占用率一定要小，因为Agent与服务是共生的，服务一旦失去响应，Agent即使拥有再好的性能也是没有意义的。`


## Why Golang？

   个人认为关于Service Mesh的选型一定会在Cpp和Golang之间，这个要参考公司的技术栈。如果追求极致的性能还是首选Cpp，这样可以避免Gc问题。因为Service Mesh链路相比传统Rpc要长，Agent Proxy需要保证轻量、稳定、性能出色。
   
   关于技术选型为什么是Golang？这里不仅仅是为了当做一次锻炼自己Golang的机会，当然还出于以下一些原因：
   - 一些大厂的经验沉淀，比如蚂蚁Sofa Mesh，新浪Motan Mesh等。
   - K8s、docker在微服务领域很火，而且以后Agent的部署一定依托于k8s，所以Go是个不错的选择，亲和度高。
   - Go有协程，有高质量的网络库，高性能方面应该占优势。
   
   
## 优化点剖析

   官方提供了一个基于Netty实现的Java Demo，由于是阻塞版本，所以性能并不高，当然这也是对Java选手的一个福音了，可以快速上手。其他语言相对起步较慢，全部都要自己重新实现。
   
   不管什么语言，大家的优化思路大部分都是一样的。这里分享一下Kirito徐靖峰非常细致的思路总结(Java版本)：[天池中间件大赛dubboMesh优化总结（qps从1000到6850）](https://mp.weixin.qq.com/s/M9QY-Oe9xdLW2plFNlraFw)，大家可以作为参考。
   
   下面这张图基本涵盖了在整个agent所有优化的工作，图中绿色的箭头都是用户可以自己实现的。

   ![](http://wangyapu.iocoder.cn/15324483540592.jpg)


- 全部过程变成`异步非阻塞、无锁`，所有请求均采用异步回调的形式。这也是提升最大的一点。
- 自己实现Http服务解析。
- Agent之间通信采用最简单的自定义协议。
- 网络传输中`ByteBuffer复用`。
- Agent之间通信`批量打包`发送。


    ```golang
ForBlock:
	for {
		httpReqList[reqCount] = req
		agentReqList[reqCount] = &AgentRequest{
			Interf:    req.interf,
			Method:    req.callMethod,
			ParamType: ParamType_String,
			Param:     []byte(req.parameter),
		}
		reqCount++
		if reqCount == *config.HttpMergeCountMax {
			break
		}
		select {
		case req = <-workerQueue:
		default:
			break ForBlock
		}
	}
    ```

- Provider负载均衡：加权轮询、`最小响应时间`（效果并不是非常明显）
- Tcp连接负载均衡：支持按最小请求数选择Tcp连接。
- Dubbo请求`批量encode`。
- Tcp参数的优化：开启TCP_NODELAY(disable Nagle algorithm),调整Tcp发送和读写的缓冲区大小。

    ```golang
	if err = syscall.SetsockoptInt(fd, syscall.IPPROTO_TCP, syscall.TCP_NODELAY, *config.Nodelay); err != nil {
		logger.Error("cannot disable Nagle's algorithm", err)
	}
    
	if err := syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_SNDBUF, *config.TCPSendBuffer); err != nil {
		logger.Error("set sendbuf fail", err)
	}
	if err := syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_RCVBUF, *config.TCPRecvBuffer); err != nil {
		logger.Error("set recvbuf fail", err)
	}
    ```

## 网络辛酸史 —— （预热赛256并发压测4400~4500）

   Go因为有协程以及高质量的网络库，协程切换代价较小，所以大部分场景下Go推荐的网络玩法是每个连接都使用对应的协程来进行读写。
   
   ![](http://wangyapu.iocoder.cn/15324504091219.jpg)
   
 
   这个版本的网络模型也取得了比较客观的成绩，QPS最高大约在4400~4500。对这个网络选型简单做下总结：
 
   - Go因为有goroutine，可以采用多协程来解决并发问题。
   - 在linux上Go的网络库也是采用的epoll作为最底层的数据收发驱动。
   - Go网络底层实现中同样存在“上下文切换”的工作，只是切换工作由runtime调度器完成。


##  网络辛酸史 —— （正式赛512并发压测） 

   然而在正式赛512并发压测的时候我们的程序并没有取得一个稳定提升的成绩，大约5500 ~ 5600左右，`cpu的资源占用率也是比较高的，高达约100%`。
   
   获得高分的秘诀分析：
   
   - Consumer Agent压力繁重，给Consumer Agent减压。
   - 由于Consumer的性能很差，Consumer以及Consumer Agent共生于一个Docker实例(4C 8G)中，只有避免资源争抢，才能达到极致性能。
   - Consumer在压测过程中Cpu占用高达约350%。
  - 为了避免与Consumer争抢资源，需要把Consumer Agent的资源利用率降到极致。
   
   通过上述分析，我们确定了优化的核心目标：`尽可能降低Consumer Agent的资源开销`。
   
#### a. 优化方案1：协程池 + 任务队列（废弃）
   
   这是一个比较简单、常用的优化思路，类似线程池。虽然有所突破，但是并没有达到理想的效果，cpu还是高达约70~80%。Goroutine虽然开销很小，毕竟高并发情况下还是有一定上下文切换的代价，只能想办法再去寻找一些性能的突破。
   
   `经过慎重思考，我最终还是决定尝试采用类似netty的reactor网络模型`。关于Netty的架构学习在这就不再赘述，推荐同事的一些分享总结[闪电侠的博客](https://www.jianshu.com/nb/7981390)。
   
#### b. 优化方案2：Reactor网络模型

   选型之前咨询了几位好朋友，都是遭到一顿吐槽。当然他们没法理解我只有不到50%的Cpu资源可以利用的困境，最终还是毅然决然地走向这条另类的路。
   
   经过一番简单的调研，我找到了一个看上去还挺靠谱（Github Star2000, 没有一个PR）的开源第三方库[evio](https://github.com/tidwall/evio)，但是真正实践下来遇到太多坑，而且功能非常简易。不禁感慨Java拥有Netty真的是太幸福了！Java取得成功的原因在于它的生态如此成熟，Go语言这方面还需要时间的磨炼，高质量的资源太少了。
   
   当然不能全盘否定evio，它可以作为一个学习网络方面很好的资源。先看Github上一个简单的功能介绍：

    evio is an event loop networking framework that is fast and small. It makes direct epoll and kqueue syscalls rather than using the standard Go net package, and works in a similar manner as libuv and libevent.
    
`说明：关于kqueue是FreeBSD上的一种的多路复用机制，推荐学习。`

为了能够达到极致的性能，我对evio进行了大量改造：

- 支持主动连接（默认只支持被动连接）
- 支持多种协议
- 减少无效的唤醒次数
- 支持异步写，提高吞吐率
- 修复Linux下诸多bug造成的性能问题

改造之后的网络模型也是取得了很好的效果，可以达到`6700+`的分数，但这还远远不够，还需要再去寻找一些突破。


#### c. 复用EventLoop

对优化之后的网络模式再进行一次梳理（见下图）：

![](http://wangyapu.iocoder.cn/15324541151412.jpg)

可以把eventLoop理解为io线程，在此之前每个网络通信c->ca，ca->pa，pa->p都单独使用的一个eventLoop。`如果入站的io协程和出站的io协程使用相同的协程，可以进一步降低Cpu切换的开销`。于是做了最后一个关于网络模型的优化：`复用EventLoop`，通过判断连接类型分别处理不同的逻辑请求。

```golang
func CreateAgentEvent(loops int, workerQueues []chan *AgentRequest, processorsNum uint64) *Events {
	events := &Events{}
	events.NumLoops = loops

	events.Serving = func(srv Server) (action Action) {
		logger.Info("agent server started (loops: %d)", srv.NumLoops)
		return
	}

	events.Opened = func(c Conn) (out []byte, opts Options, action Action) {
		if c.GetConnType() != config.ConnTypeAgent {
			return GlobalLocalDubboAgent.events.Opened(c)
		}
		lastCtx := c.Context()
		if lastCtx == nil {
			c.SetContext(&AgentContext{})
		}

		opts.ReuseInputBuffer = true

		logger.Info("agent opened: laddr: %v: raddr: %v", c.LocalAddr(), c.RemoteAddr())
		return
	}

	events.Closed = func(c Conn, err error) (action Action) {
		if c.GetConnType() != config.ConnTypeAgent {
			return GlobalLocalDubboAgent.events.Closed(c, err)
		}
		logger.Info("agent closed: %s: %s", c.LocalAddr(), c.RemoteAddr())
		return
	}

	events.Data = func(c Conn, in []byte) (out []byte, action Action) {
		if c.GetConnType() != config.ConnTypeAgent {
			return GlobalLocalDubboAgent.events.Data(c, in)
		}

		if in == nil {
			return
		}
		agentContext := c.Context().(*AgentContext)

		data := agentContext.is.Begin(in)

		for {
			if len(data) > 0 {
				if agentContext.req == nil {
					agentContext.req = &AgentRequest{}
					agentContext.req.conn = c
				}
			} else {
				break
			}

			leftover, err, ready := parseAgentReq(data, agentContext.req)

			if err != nil {
				action = Close
				break
			} else if !ready {
				data = leftover
				break
			}

			index := agentContext.req.RequestID % processorsNum
			workerQueues[index] <- agentContext.req
			agentContext.req = nil
			data = leftover
		}
		agentContext.is.End(data)
		return
	}
	return events
}
```

复用eventloop得到了一个比较稳健的成绩提升，每个阶段的eventloop的资源数都设置为1个，最终512并发压测下cpu资源占用率约50%。


## Go语言层面的一些优化尝试

最后阶段只能丧心病狂地寻找一些细节点，所以也对语言层面做了一些尝试：

- Ringbuffer来替代Go channel实现任务分发

RingBuffer在高并发任务分发的场景中比Channel性能有小幅度提升，但是站在工程的角度，个人还是推荐Go channel这种更加优雅的做法。

- Go自带的encoding/json包是基于反射实现的，性能是个诟病

使用字符串自己拼装Json数据，这样压测的数据越多，节省的时间越多。

- Goroutine线程绑定

    ```golang
	runtime.LockOSThread()
	defer runtime.UnlockOSThread()
    ```

- 修改调度器默认时间片大小，自己编译Go语言（没啥效果）


## 总结

- 剑走偏锋，花费了大量时间去改造网络，功夫不负有心人，结果是令人欣慰的。
- Golang在高性能方面是足够出色的，值得深入研究学习。
- `性能优化离不开的一些套路：异步、去锁、复用、零拷贝、批量等`。
    
最后抛出几个想继续探讨的Go网络问题，和大家一起讨论，有经验的朋友还希望能指点一二：  
  
1. 在资源稀少的情况下，处理高并发请求的网络模型你会怎么选型？（假设并发为1w长连接或者短连接）
2. 百万连接的规模下又将如何选型？
   

   
   
   
   

   
   








