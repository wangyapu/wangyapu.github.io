---
layout:     post
title:      "Helm V3使用指北"
subtitle:   "\"云原生技术体系中被广泛使用的应用管理工具\""
date:       2020-04-10 12:00:00
author:     "wangyapu"
header-img: "img/post-helm-user-guide.png"
tags:
    - 架构
---

# Helm V3使用指北

## 前言

什么是 Helm？Helm 是云原生领域最火热的应用管理工具。众所周知 Kubernetes 是自动化的容器管理平台，然而 Kubernetes 并没有抽象出应用的概念，通常应用的描述是非常复杂的，一个应用可能是由多种资源组成。例如一个典型的前后端分离的应用包含以下资源：

1. web-application-deployment.yaml，前端 Web 服务。
2. web-service.yaml，前端服务访问入口。
3. backend-server-deployment.yaml，后端服务。
4. backend-server-configmap.yaml，后端服务配置。
5. backend-mysql.yaml，后端服务依赖的 Mysql 实例。
6. backend-mysql-service.yaml，Mysql 实例访问入口。

我们通过多次 kubectl apply -f 上述资源，但是后续无法有效管理应用所包含的资源。这也正是 Helm 要解决的难题，更好地帮助用户定义、部署以及管理应用。

## Helm核心概念

### Chart

Helm 采用 Chart 的格式来标准化描述一个应用（K8S 资源文件集合），Chart 有自身标准的目录结构，可以将目录打包成版本化的压缩包进行部署。就像我们下载一个软件包之后，就可以在电脑上直接安装一样，同理 Chart 包可以通过 Helm 部署到任意的 K8S 集群中。

### Config

Config 指应用配置参数，在 Chart 中由 values.yaml 和命令行参数组成。Chart 采用 Go Template 的特性 + values.yaml 对部署的模板文件进行参数渲染，也可以通过 Helm Client 的命令
--set key=value 的方式进行参数赋值。

### Repository

类似于 Docker Hub，Helm 官方、阿里云等社区都提供了 Helm Repository，我们可以通过 helm repo add 导入仓库地址，便可以检索仓库并选择别人已经制作好的 Chart 包，开箱即用。

### Release

Release 代表 Chart 在集群中的运行实例，同一个集群的同一个 Namespace 下 Release 名称是唯一的。Helm 围绕 Release 对应用提供了强大的生命周期管理能力，包括 Release 的查询、安装、更新、删除、回滚等。

## Helm V2 & V3 架构设计

Helm V2 到 V3 经历了较大的变革，其中最大的改动就是移除了 Tiller 组件，所有功能都通过 Helm CLI 与 ApiServer 直接交互。Tiller 在 V2 的架构中扮演这重要的角色，但是它与 K8S 的设计理念是冲突的。

1. 围绕 Tiller 管理应用的生命周期不利于扩展。
2. 增加用户的使用壁垒，服务端需要部署 Tiller 组件才可以使用，侵入性强。
3. K8S 的 RBAC 变得毫无用处，Tiller 拥有过大的 RBAC 权限存在安全风险。
4. 造成多租户场景下架构设计与实现复杂。

![](http://wangyapu.iocoder.cn/15878084179986.jpg)


## 基本使用

### Chart目录结构

![](http://wangyapu.iocoder.cn/15878187708502.jpg)

每个 Chart 固定由 charts 目录、templates 目录、Chart.yaml 等必要的内容组成。Chart 的描述形式是非常灵活的，是可以嵌套多层 Chart 的，但是并不推荐这么写。

### 模板管理

- 创建 Chart 骨架

  ```bash
  helm create ./chart-demo
  ```
  
- Chart 打包
  
  ```bash
  helm package ./chart-demo
  ```
  
- 获取 Chart 包元数据信息
  
  ```bash
  helm inspect chart ./chart-demo
  ```
  
- 本地渲染模板文件

  ```bash
  helm template ${chart-demo-release-name} --namespace ${namespace} ./chart-demo
  ```
  
- 查询 Chart 依赖信息 

  ```bash
  helm dependency list ./chart-demo
  ```  

### 模板部署

- 查询 Release 列表

  ```bash
  helm list --namespace ${namespace}
  ```  

- Chart 安装

  ```bash
  helm install ${chart-demo-release-name} ./chart-demo --namespace ${namespace}
  ``` 
  
- Chart 版本升级

  ```bash
  helm upgrade ${chart-demo-release-name} ./chart-demo-new-version --namespace ${namespace}
  ``` 
  
- Chart 版本回滚

  ```bash
  helm rollback ${chart-demo-release-name} ${revision} --namespace ${namespace}
  ```  

- 查看 Release 历史版本

  ```bash
  helm history ${chart-demo-release-name} --namespace ${namespace}
  ```  
  
### 第三方仓库

- 添加第三方仓库

  ```bash
  helm repo add stable http://kubernetes-charts.storage.googleapis.com/
  ```

- 检索第三方仓库

  ```bash
  helm search repo ${keyword}
  ``` 

## 一些最佳实践

### Job 无法重新执行

Job 更新后想重新升级版本，结果发现 Job 资源冲突。有以下解决办法：

1. 发布产品层解析 Chart，控制 Job 资源的清理。
2. 采用 Helm 的 hook 机制，来干预 Release 生命周期的执行点。比较推荐这种做法。

**Helm 定义的 hooks**

| 名称 | 说明 |
| --- | --- |
| pre-install | 在模板渲染后，K8S 创建资源之前执行 |
| post-install | 资源加载至 K8S 后执行 |
| pre-delete | K8S 删除资源之前执行 |
| post-delete | 删除 Release 资源后执行 |
| pre-upgrade | 在模板渲染后，资源加载至 K8S 之前执行 |
| post-upgrade | 在所有资源升级后执行升级 |
| pre-rollback | 模板渲染后，资源已回滚之前，在回滚请求上执行 |
| post-rollback | 在修改所有资源后执行回滚请求 |

**hook 删除策略**

1. hook-succeeded：hook 成功执行后删除。
2. hook-failed：hook 执行期间失败删除。
3. before-hook-creation：创建新 hook 之前删除以前的 hook。

**hooks 的常用场景**

1. 安装 Chart 之前执行数据库初始化工作。
2. 安装 Chart 之前加在 ConfigMap 或者 Secret。
3. 删除 Release 之前执行清理工作，实现优雅下线。

**hooks 实现 Job 资源管理**

- Chart 目录结构

  ```
    test-job
    ├── Chart.yaml
    ├── charts
    └── templates
        ├── hooks
        │   └── pi-job.yaml
        └── deployment.yaml
  ```

- pi-job.yaml

  ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: pi
      annotations:
        "helm.sh/hook": "pre-install,pre-upgrade,pre-rollback"
        "helm.sh/hook-delete-policy": "before-hook-creation"
        "helm.sh/hook-weight": "-5" # 字符串类型，可以为正数或者负数，在每个执行周期的时间点会对 hook 进行排序
    spec:
      template:
        spec:
          containers:
            - name: pi
              image: perl
              command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          restartPolicy: Never
      backoffLimit: 4
  ```

### ConfigMap/Secret 更新不生效

部署好的应用，再次变更 ConfigMap/Secret 后，Workload 并无法感知。有以下解决办法：

1. 应用程序实现定时读取 ConfigMap/Secret 的功能，更新配置参数。
2. 在 Workload 的模板文件中加入以下内容（根据实际情况调整），只要有ConfigMap/Secret的变更，都会触发 Pod 的重建。

    ```
    annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    ```

### 应用发布顺序依赖

虽然 Chart 可以通过 requirements.yaml 来管理依赖关系，并按照顺序下发模板资源，但是并无法控制子 Chart 之间的发布顺序。例如服务 B 部署必须依赖服务 A 的资源全部 Ready。可以通过自定义子 Chart 之间的依赖顺序，在产品层控制每个子 Chart 的发布过程。

## 总结与展望

Helm 在云原生技术体系中提供了强大应用管理能力，Helm + K8S 使得复杂应用的部署成本大幅度降低。V3 版本社区也在快速地迭代当中，完全拥抱 K8S 的设计理念，你完全可以像 Kubectl 一样使用 Helm。

Helm 在企业级的落地需要考虑诸多因素，如技术风险能力、无损发布、资源依赖等等，这些虽然看上去并不是 Helm 所管辖的范畴，但都是生产环境稳定落地需要解决的问题，这就需要与企业的技术生态做深度的打通。

