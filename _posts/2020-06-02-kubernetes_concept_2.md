---
layout:     post
title:      "Kubernetes关键知识点（二）"
subtitle:   "\"摆摊也能说我了解K8S了\""
date:       2020-05-31 12:00:00
author:     "wangyapu"
header-img: "img/post-kubernetes-concept-2.jpg"
tags:
    - 云原生
---

# Kubernetes关键知识点（二）

## 前言

接着上文 [Kubernetes关键知识点（一）](https://mp.weixin.qq.com/s/zK-9RX8jiBID54WPRapj6A)继续整理 Kubernetes 的基础知识。

## 核心概念

### Job & CronJob

Job 确保执行任务的 Pod 只运行一次，执行完结束退出。其中 RestartPolicy 仅支持 Never 或 OnFailure，不支持 Always。很好理解，如果支持 Always，执行任务会造成无限循环。

同名的 Job 不得重复执行，关于 Job 的清理有以下几种方法：

- 直接删除 Job 对象。
- TTL 机制：设置 .spec.ttlSecondsAfterFinished，如果设置为 60，则 60s 之后删除 Job。若设置为 0，则在 Job 结束后立即删除。
- 如果是 HelmChart 部署，可以采用 hook 机制删除。

Job Spec 几个重要的参数：

- .spec.completions: Job 需要成功运行 Pods 的个数，默认值 1。
- .spec.parallelism: Job 在任一时刻应该并发运行 Pods 的个数，默认值 1。
- .spec.activeDeadlineSeconds: Job 中标记失败的 Pod 的最大运行时间，超过时间未结束将会终止。
- .spec.backoffLimit: Job 失败后的进行重试次数。每次重试都会有延迟时间，延迟时间是指数级增长。

Job 的几种模式和使用场景：


| 用法 | 场景 |
| --- | --- |
| 一次性 Job | 项目初始化，如数据库、环境配置 |
| 固定结束次数 Job | 串行执行任务 |
| 带有工作队列的并行 Job | 并行处理工作队列中任务，如离线任务 |

CronJob 指定时运行任务，它在 Job 上加上了时间调度。schedule 调度配置也和 Linux 的 crontab 格式是一致的。

### Volume

Volume是对各种存储资源的抽象，为管理存储资源提供统一的接口。

Volume 的生命周期与 Pod 的保持一致，当 Pod 被删除，对应的 Volume 也会被删除，这里并不是指数据也会被删除。

Volume 常见的几种类型：

- emptyDir

    emptyDir 是 Host 上创建的临时目录，它不具备持久性。emptyDir 是节点上的一个空目录，任何 Pod 中的容器都挂载使用。常用于临时目录或者缓存数据。

    - Pod 中容器重启或者意外删除，emptyDir 不会删除。
    - Pod 调度到其他节点，emptyDir 删除且数据永久丢失。
    - Pod 删除或者缩容，emptyDir 删除且数据永久丢失。

- HostPath

    HostPath 是将宿主机的目录 mount 到 Pod 的容器中，同一个节点上的 Pod 可用于共享数据。Pod 即使被删除，HostPath 的数据还会保留。但是 HostPath 也存在一个很大的缺陷，当 Pod 调度到另外一个节点后，就无法使用之前节点存储的数据文件了。

- gitRepo

    数据将首先会 git clone 下来，但是后续里面内容更新不会与 git 同步。

- 网络存储卷

    Kubernetes 支持挂载网络存储，例如 NFS、glusterfs等。

### Persistent Volume (PV) & Persistent Volume Claim (PVC)

PV & PVC 可以说是用于定义网络存储，单独在这里写说明它的重要性。PV 是用于描述存储的模型，PVC 表示用户对存储资源的申请，用于关联 Pod 和 PV 的模型。`PV 和 PVC 是一一对应的关系，它们的关系类似于 Pod 与节点的关系，Pod 消耗节点资源，PVC 消耗 PV 资源。`

临时卷无法解决数据持久化的问题，而 PV 将 Pod 和 Volume 的生命周期分离，所以 PV 是 Kubernetes 集群资源，并不属于某个 namespace。既然 PV 可以实现存储和应用的分离，为什么还需要 PVC 对象呢？

PV 和 PVC 的关注点不一样，PV 描述了存储类型以及复杂的存储信息，而 PVC 是对存储的二次抽象，让用户不需要关心存储底层的细节，用户只需要通过 PVC 定义自己关心的需求，例如存储大小、访问权限等，降低用户的使用心智。

![](media/15927398447233.jpg)

PV 的生命周期：

- Provisioning：PV 创建
- Binding：PV 分配给 PVC
- Using：Pod 引用 PVC 使用 Volume
- Releasing：Pod 释放 Volume，删除 PVC
- Reclaiming：回收 PV


### StorageClass

PV 是支持静态和动态的，上述我们提到的都是静态的，如果需要使用一个 PVC，必须手动创建一个 PV，在实际的生产环境中如果无法实现自动化是非常痛苦的。那么如何实现动态的 PV？那就是 StorageClass 存储类。

PV 定义了具体 Volume 的属性，例如 Volume 类型、挂载目录等，而 StorageClass 相当于 PV 的模板，可以动态创建 PV。根据 StorageClass 的描述能够清楚各类存储资源的特性，用户可以根据需求申请不同的存储资源。

StorageClass 重要参数：

- Provisioner 存储分配器：用来决定使用哪个卷插件分配 PV。该字段必须指定。
- Parameters 参数：根据分配器可以接收不同的参数。
- mountOptions 挂载选项：StorageClass 动态创建的 PV 将用该字段指定挂载选项。
- reclaimPolicy 回收策略：Delete 或者 Retain，默认 Delete。

### ConfigMap & Secrets

ConfigMap 将应用的配置信息与 Pod 分离，用于存储非机密性的键值对数据。

Pod 使用 ConfigMap 的方式：

- 环境变量
- 命令行参数
- 将 ConfigMap 作为文件以 Volume 的形式挂载

对于机密性的数据还是需要使用 Secrets 来进行管理。Secret 与 ConfigMap 的使用方式基本类似，区别在于 Secret 是以 Base64 保存。

> 1. ConfigMap 必须在 Pod 之前创建
> 2. Pod 只能使用同一个namespace 的 ConfigMap

### Downward API

Downward API 可以将 Pod 的信息传递到容器内。

- 环境变量

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pod
    spec:
      containers:
      - name: test-container
        image: busybox
        command: ["/bin/sh","-c","sleep 3600"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
    ```

- Volume 挂载

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pod
      labels:
        app: test-app
        cluster: test-cluster
      annotations:
        region: shanghai
    spec:
      containers:
      - name: test-container
        image: busybox
        imagePullPolicy: Never
        command: ["sh","-c"]
        args:
        - while true; do
            if [[ -e /etc/labels ]];then
              echo -en '\n\n';cat /etc/labels;
            fi
            if [[ -e /etc/annotations ]];then
              echo -en '\n\n';cat /etc/annotations;
            fi
            sleep 3600;
          done
        volumeMounts:
        - name: podinfo
          mountPath: /etc
          readOnly: false
      volumes:
      - name: podinfo
        downwardAPI:
          items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
    ```

### QoS

QoS ( Quality of Service ) 服务质量，在 Kubernetes 中用于描述资源需求。

QoS 由 Pod 的 request 和 limits 决定，按照优先级从高到低分为三类：

- Guaranteed
    - 只设置 limit
    - request == limit
    
- Burstable： 保障资源的最小需求，但不会超过limits。
    - Pod 的一个或多个容器只设置了 requests。
    - Pod 的一个或多个容器同时设置 requests 和 limit，request != limit。
    - Pod 所有容器均设置了 limit，但资源类型不一样。
    - Pod 中有容器满足 Burstable 条件，Pod 则也是 Burstable 类型。

- Best-Effort：没设置 requests 和 limits，可能会消耗完 Node 所有资源，资源紧张时会优先被杀死。

### Taint & Toleration

Taint（污点）和 Toleration（容忍）用于优化 Pod 在集群中的调度策略。

- Taint 

    每个 Node 可以设置一个或者多个 Taint，与亲和性相反，允许 Node 排斥 Pod 的调度，具有 Taint 的 Node 和 Pod 是互斥关系。
    
    - 至少 1 个 effect == NoSchedule 的 Taint 没有被 Toleration，Pod 不会调度到该 Node。
    - 所有 effect == NoSchedule 都被 Toleration，但至少 1 个 effect == PreferNoSchedule 没有被 Toleration，将尽可能不调度到该 Node。
    - 至少 1 个 effect == NoExecute，不仅不会调度到该  Pod，而且该节点没有设置对应 Toleration 的 Pod 都将被驱逐。
    
    
    Kubernetes 内置的一些 Taint：
    
    - node.kubernetes.io/not-ready：节点未就绪
    - node.kubernetes.io/unreachable：节点无法访问
    - node.kubernetes.io/out-of-disk：节点磁盘资源不足
    - node.kubernetes.io/memory-pressure：节点内存资源有压力
    - node.kubernetes.io/disk-pressure：节点磁盘资源有压力。
    - node.kubernetes.io/network-unavailable：节点网络不可用
    - node.kubernetes.io/unschedulable: 节点不可调度
   
- Toleration

    ```yaml
    tolerations: 
    - key: "node.kubernetes.io/test-key" 
      operator: "Equal" 
      value: "test-value"
      effect: "NoExecute"
      tolerationSeconds: 3600 # 当 pod 被驱逐时，可以继续在 node 上运行的时间
    ```
    
    operator 取值:

    - Equal: 默认值，需要配置 value 值。
    - Exists: 不需要指定 value。
    
    > 1. key 为空且 operator: "Exists"：容忍所有 Taint。
    > 2. effect 为空：匹配所有 effects。
    
    



