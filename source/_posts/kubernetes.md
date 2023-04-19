---
title: Kubernetes 基本介绍
tags:
  - Kubernetes
  - 云原生
description: 本篇文章主要介绍Kubernetes的基本概念，每个组件的在系统中的位置和功能，以及Kubernetes的基本使用。
cover: 'http://images.ashery.cn/img/DSC01653%20(2).JPG'
abbrlink: 52b98882
date: 2023-02-04 19:34:20
categories:
---




# Kubernetes 基本架构

## 什么是Kubernetes
我们知道`Docker`是一个很好的打包和运行应用程序的方式，但是在生产环境中，你需要管理数以千计运行着应用程序的容器，并确保服务不会下线。 例如，如果一个容器发生故障，则你需要启动另一个容器，这对运维人员来说是不能忍受也不现实的。 如果此行为交由给系统处理，是不是会更容易一些？
这就是 Kubernetes 做的事情！Kubernetes 提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移你的应用、提供部署模式等。 例如，Kubernetes 可以轻松管理系统的 Canary (金丝雀) 部署。

所以Kubernetes是一个管理容器（此处不仅限于Docker）的系统，即容器编排系统。

实际上不应该仅仅把Kubernetes当做容器编排系统来看待。实际上，有了Kubernetes之后，并不需要人为的编排。编排是指定一组工作流workflow，规定每个条件下触发什么操作。但是Kubernetes并不是这样的，**Kubernetes只需要我们提供一份理想状态下系统应该达到的状态**（`Spec`），Kubernetes会自动进行内部调整（可以看做是一系列内部的workflow），以不断维持系统理想状态。不仅便于用户使用，也使得系统更易于使用且功能更强大、 系统更健壮，更为弹性和可扩展。

![container_evolution](http://images.ashery.cn/img/63e63c994757feff3390edde.jpg)

## Kubernetes 提供的能力
Kubernetes 提供的最主要的能力有
1. **自动部署和回滚**

   你可以使用 Kubernetes 描述已部署容器的所需状态（`Spec`）， 它可以根据当前状态（`Status`）以受控的速率将实际状态更改为期望状态。 例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器。

2. **自我修复**

   Kubernetes 将重新启动失败的容器、替换容器、杀死不响应用户定义的运行状况检查的容器， 并且在准备好服务之前不将其通告给客户端。

3. **自动进行资源调度**

   你为Kubernetes提供一个由多个节点组成的集群，以用于运行你的容器进化成呢个。然后告诉 Kubernetes 每个容器需要多少 CPU 和内存 (RAM)。 Kubernetes 可以将这些容器按实际情况调度到你的节点上，以最佳方式利用你的资源。

3. **服务发现和负载均衡**

   Kubernetes 可以使用 DNS 名称或自己的 IP 地址来暴露容器。 如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。



# 组件介绍

和Docker一样，Kubernetes为了实现这些能力，抽象出一系列的组件，每种组件提供特定的能力，所有的这些组件相互协作共同组成了强大的Kubernetes。

下面是Kubernetes的架构图，展示了一些组件和运行流程。

![High level Kubernetes architecture diagram showing a cluster with a master and two worker nodes ](http://images.ashery.cn/img/Kubernetes-101-Architecture-Diagram.jpg)

可见，Kubernetes 首先是一套分布式系统，由多个节点组成，节点分为两类：一类是属于控制平面的主节点/控制节点（Master Node）；一类是属于运行平面的工作节点（Worker Node）。

控制节点更为复杂，所需资源比较少，工作节点相反。

## 控制面

### **API Server**

是整个系统的对外接口，提供一套 RESTful 的 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)，供客户端和其它组件调用。可通过复制支持水平扩展与负载均衡。

### Control Manager：

负责管理**控制器**，是Kubernetes的一个守护进程。 在 Kubernetes 中，每个`Controller`是一个`for loop`，通过 API 服务器监视集群的共享状态， 并尝试进行更改以将当前状态转为期望状态（`Spec`）。

理论上来说， 每个[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在同一个进程中运行。

这些控制器包括：

- Node Controller：负责在节点出现故障时进行通知和响应。
- Job Controller：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成。
- EndpointSlice controller：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
- ServiceAccount controller：为新的命名空间创建默认的服务账号（ServiceAccount）。

### Scheduler

负责对资源进行调度，分配某个 `pod` 到某个`node`上。内部实现机制会考虑很多因素，比如资源需求和限制，亲和性等，当然也可以实现自己的策略。

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：**过滤**和**打分**。

**过滤阶段**会将所有满足 Pod 调度需求的节点选出来。 例如，PodFitsResources 过滤函数会检查候选节点的可用资源能否满足 Pod 的资源请求。 在过滤之后，得出一个节点列表，里面包含了所有可调度节点；通常情况下，这个节点列表包含不止一个节点。如果这个列表是空的，代表这个 Pod 不可调度。

**打分阶段**，调度器会为 Pod 从所有可调度节点中选取一个最合适的节点。 根据当前启用的打分规则，调度器会给每一个可调度节点进行打分。

最后，kube-scheduler 会将 Pod 调度到**得分最高**的节点上。 如果存在多个得分最高的节点，kube-scheduler 会从中随机选取一个。

### etcd

etcd是分布式一致性高可用的键值存储数据库，用作 Kubernetes 所有集群数据的后台数据库。包括所有节点的状态、配置文件等都通过etcd同步。组件可以自动的去侦测 Etcd 中的数值变化来获得通知，并且获得更新后的数据来执行相应的操作。

## 运行面

### Node & Pod

**Node**对应K8S集群中的一个节点，可以认为是一个虚拟机或者物理机器，取决于所在的集群配置。 这些节点由控制面负责管理。通常集群中会有若干个节点。每个节点包含运行 Pod 所需的服务： kubelet、 container runtime (一般来说是Docker) 以及 kube-proxy。

**Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元，**这意味着Pod内的Containers是一同被调度的。**Pod**是一组（一个或多个）容器，这些容器共享存储、网络、命名空间等，部署在统一Pod内的Containers理论上应该是紧密相关的。

### Kubelet & Proxy

**Kubelet 作为集群中的 node agent**，一方面，kubelet 扮演的是集群控制器的角色，它定期从 API Server 获取 Pod 等相关资源的信息，并依照这些信息，控制运行在节点上 Pod 的执行；另外一方面， kubelet 作为节点状况的监视器，获取节点信息，并以集群客户端的角色，把这些状态信息同步到 API Server。

**kube-proxy 是集群中每个节点Node上所运行的网络代理**， 是实现 Kubernetes 服务（Service） 概念的一部分。kube-proxy 维护节点上的一些网络规则， 这些网络规则会允许从集群内部或外部的网络会话与 Pod 进行网络通信。

在运行面的这些组件中，只有这两个概念是具体的代码组件，意味着你能在源码中找到对应的代码，并且查看其工作流程。其他的概念都是抽象的概念，是这些实际组件的操作对象和抽象功能。

### Service & Ingress

**Service将一组Pods使用相对固定的IP公开为网络服务**，以供集群内的其他组件访问。 在集群的运行和更新中，Pod 是非永久性资源，会被随时创建和销毁的。但是每个 Pod 都有自己的 IP 地址，这导致了一个问题： 如果一组两组Pod具有依赖关系，那么每次调用/每次Pod更新都需要重新获取对方的IP地址。通过**将一组Pod附加一个固定IP地址（Service）**，解耦了Pod和Service（IP地址）生命周期，

**Ingress 是对集群中服务的外部访问进行管理的 API 对象**，典型的访问方式是 HTTP。Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管等功能。

### Persist Data With Volume

**Volumes是Pod数据持久化的地方。**和Docker一样， Kubernetes也有Volume的概念。但是Kubernetes上的Volumes类型更多，提供的功能也更强大。Volume的核心是一个目录，其中可能存有数据，Pod 中的容器可以访问该目录中的数据。 所采用的特定的Volume类型将决定该目录如何形成的、使用何种介质保存数据以及目录中存放的内容。

Kubernetes支持的Volume类型有：

1. **持久卷**（PersistentVolume，PV）：**持久卷** 是集群中的一块存储，持久卷是集群资源，就像节点Node也是集群资源一样。持久卷的生命周期与Pod无关，Pod重启持久卷的数据也不会消失。
2. **临时卷**（Ephemeral Volume）：有些应用程序需要额外的存储，但并不关心数据在重启后是否仍然可用。 例如，缓存服务经常受限于内存大小，可以将不常用的数据转移到比内存慢的存储中，对总体性能的影响并不大。另外有些应用程序需要以文件形式注入的只读数据，比如配置数据或密钥。**临时卷** 就是为此类用例设计的。**临时卷会遵从 Pod 的生命周期**，与 Pod 一起创建和删除， 所以停止和重新启动 Pod 时，不会受持久卷在何处可用的限制。
3. **投射卷**（Projected Volumes）：投射卷将其他存储映射到统一目录上，比如`Secret`、`ConfigMap`、`Token`

### Configuration With ConfigMap & Secret

应用程序启动时可能需要从外部读取一些配置参数，比如数据库用户名和密码，在Docker中可以通过docker-compose的.yaml文件实现。同样，在Kubernetes中，也提供了配置这些参数的方法。

**ConfigMap**是Kubernetes提供的一种 API 对象，用来将**非机密性**的数据保存到键值对中。使用时， [Pods](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)可以将其用作环境变量、命令行参数或者存储卷中的配置文件。ConfigMap 在设计上不是用来保存大量数据的。在 ConfigMap 中保存的数据不可超过 1 MiB。如果你需要保存超出此尺寸限制的数据，你可能希望考虑挂载存储卷 或者使用独立的数据库或者文件服务。

ConfigMap 并不提供保密或者加密功能，如果你想存储的数据是机密的，比如账号密码，可使用 **Secret**。

**Secret** 是一种包含少量敏感信息例如密码、令牌或密钥的对象。使用 Secret 意味着不需要在应用程序代码中包含机密数据。由于创建 Secret 可以独立于使用它们的 Pod， 因此在创建、查看和编辑 Pod 的工作流程中暴露 Secret（及其数据）的风险较小。 Kubernetes 和在集群中运行的应用程序也可以对 Secret 采取额外的预防措施， 例如避免将机密数据写入非易失性存储。

### Replica With Deployment & StatefulSet

**Deployment** 为 Pod 和 ReplicaSet 提供**声明式**的更新能力。

你负责描述 Deployment 中的目标状态，而 Deployment 控制器（Controller） 以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 收养其资源。

**ReplicaSet** 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

**StatefulSet** 是用来管理**有状态应用**的工作负载 API 对象。

和 Deployment 类似， StatefulSet 管理基于相同容器规约的一组 Pod。但和 Deployment 不同的是， StatefulSet 为它们的每个 Pod 维护了一个有粘性的 ID。这些 Pod 是基于相同的规约来创建的，但是不能相互替换：无论怎么调度，**每个 Pod 都有一个永久不变的 ID**。

但是在Kubernetes中，使用StatefulSet非常麻烦，一般不建议用。

## 组件的设计

1. 在 Kubernetes 中几乎所有的组件都是**无状态**的，状态都保存在统一的 etcd 里面，这使得扩展性非常好，组件之间**异步**完成自己的任务，将结果放在 etcd 里面，互相不耦合。

2. 组件通过**事件**需要与apiserver 交互，核心功能组件不直接与api-server 通信，而是抽象了一个Informer 负责apiserver 数据的本地cache及监听。Informer 还会比对 资源是否变更（依靠内部的Delta FIFO Queue），只有变更的资源 才会触发handler。

# 配置文件

前面介绍了这些抽象组件后，下面我们开始构建这些组件，创造一个完整Kubernetes集群。

API Server是Kubernetes的入口，我们可以通过openAPI，CLI和UI界面向API Server，发起请求，更新集群状态。请求的格式必须是yaml文件（推荐）或者json文件。

下面是一个`yaml`示例文件，一个配置文件由三部分组成：metadata、specification（目标状态）、status（实际状态）。其中status由Kubernetes维护，从etcd中读取，并不会出现在请求中。

```yaml
apiVersion: apps/v1 # 指定api版本
kind: Deployment # Deployment, ConfigMap, Servicem, Secret ....
metadata: 
  name: mongo-deployment
  labels: # labels 的作用是与选择器selector 匹配
    app: mongo
spec: # replicaSet 的期望状态
  replicas: 1 # Pod副本个数
  selector: # 指定创建的 ReplicaSet 如何查找要管理的 Pod（根据label匹配）
    matchLabels:
      app: mongo
  template: # Pod 的模版
    metadata: 
      labels: 
        app: mongo # 所有的Pod被打上`app:mongo`的标签
    spec: # Pod 的期望状态
      containers: # Pod内的容器配置
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-user
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password  
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector: # 指定Service需要附加在哪个Pod上
    app: mongo # forward pod name
  ports:
    - protocol: TCP
      port: 27017 # service port 
      targetPort: 27017 # port in Container, should in the .spec.template.spec.containers.ports
```

# 本地构建K8S集群

要想在自己的开发机器上创建一个Kubernetes集群是非常麻烦的，你需要部署很多个Node。使用**[Minikube](https://minikube.sigs.k8s.io/docs/start/)**可以帮助我们简化这个流程，只需要一台机器，一个命令` minikube start`（with Docker Running），启动一个容器，就可以模拟整个K8S集群。

[**kubectl**](https://kubernetes.io/docs/reference/kubectl/)是K8S集群的命令行工具CLI，向API Server发送minikube集群的操作命令。

```bash
$ minikube start

$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

$ kubectl get po -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-787d4945fb-pngqb           1/1     Running   2 (3m ago)      4m14s
kube-system   etcd-minikube                      1/1     Running   2 (3m5s ago)    4m27s
kube-system   kube-apiserver-minikube            1/1     Running   1 (3m5s ago)    4m26s
kube-system   kube-controller-manager-minikube   1/1     Running   2 (3m5s ago)    4m26s
kube-system   kube-proxy-nvknt                   1/1     Running   1 (3m15s ago)   4m14s
kube-system   kube-scheduler-minikube            1/1     Running   2 (3m5s ago)    4m27s
kube-system   storage-provisioner                1/1     Running   2 (3m15s ago)   4m25s
```



# 小结

本文主要介绍了Kubernetes的基本概念，Docker+K8s的概念太多了，官方文档也比较乱，刚开始的时候其实不需要按照官方文档的排版，只把他当做一个Referece比较好。

后面打算参考[Kubernetes 学习路径](https://www.infoq.cn/article/9DTX*1i1Z8hsxkdrPmhk)里面提到的路径来系统的学习Kubernetes的内容。
