**Kubernetes的架构图**



  
![](https://www.hi-linux.com/img/linux/k8s_0330_01.png)

**核心组件**

* Master

Master节点运行着集群管理相关的一组进程：`etcd`、`kube-apiserver`、`kube-controller-manager`、`scheduler`。这些进程实现了整个集群的资源管理、Pod调度、弹性伸缩、安全控制、系统监控、纠错等管理功能。

* Node\(节点\)

节点是Kubernates系统中的一台工作机器\(之前的版本叫做Minion\)，既从属主机。它可以是物理机，也可以是虚拟机。每一个节点都包含了Pod运行所需的必要服务，例如Docker、kubelet和网络代理\(proxy\)，节点受Kubernates系统中的主节点控制。

和Pod、Service不一样，节点本身并不属于Kubernates的概念，它是云平台中的虚拟机或实体机。所以当一个节点加入到Kubernates系统中时，它将会创建一个数据结构来记录该节点的信息。另外，不是所有节点都能加入到Kubernates系统中，只有那些通过验证的节点才能成为Kubernates节点。

你可以通过下面的命令获取节点运行情况列表：

| 1 2 3 4  | $ kubectl get nodes  NAME          LABELS                               STATUS 172.19.17.99  kubernetes.io/hostname=172.19.17.99  Ready  |
| :--- | :--- |


* Pod\(容器组\)

Pod是一组并置的应用容器，是Kubernetes中可以被创建，调度的最小单元。

一个Pod可以被一个容器化的环境看做是应用层的逻辑宿主机\(Logical Host\)，通常一个Node中可以运行几百个Pod，每个Pod中有多个容器应用，同一个Pod中的多个容器应用通常是紧密耦合的\(相当于多个业务容器组成的一个逻辑虚拟机\)。

一个Pod中的多个容器应用通常是紧耦合的。Pod在Node上被创建、启动或者销毁。

每个Pod中有一个特殊的Pause容器，其他的成为业务容器，这些业务容器共享Pause容器的网络栈以及Volume挂载卷，因而他们之间的通信及数据交互更为高效。

同一个pod中的业务容器共享如下资源：

1. PID命名空间\(不同应用程序可以看到其他应用程序的PID\)
2. 网络命名空间\(pod中多个容器可以访问同一个IP和端口范围\)
3. IPC命名空间\(能够使用SystemV IPC或者POSIX消息队列进行通信\)
4. UTS命名空间\(共享同一个主机名\)
5. Volumes\(访问定义在pod级别的存储卷\)

Pod可以单独创建。由于Pods没有可控的生命周期，如果他们进程死掉了，他们将不会重新创建。出于这个原因，建议您使用复制控制器。

* Replication Controller\(复制控制器\)

Replication Controller是Kubernetes系统中的核心概念，用于管理Pod的生命周期。在Master内，Controller Manager进程通过RC的定义来完成Pod的创建、监控、启停等操作。

Kubernetes通过RC中定义的Label筛选出对应的Pod实例并实时监控其状态和数量，如果实例数量少于定义的副本数量，则会根据RC中定义的Pod模板来创建一个新的Pod，然后Scheduler将此Pod调度到合适的Node上启动运行，直到Pod实例数量达到预定目标。这个过程完全是自动化的。

* Replica Set

Replica Sets能够确保在某个时间点上，一定数量的Pod在运行。RelicaSet是Replication Controller的升级版本，两者的区别主要在选择器selector,Replica支持集合级别的选择器，而前期的Replication Controller支持在等号描述的选择器。目前Replica Sets主要用于Deployment中。

* Deployment

Deployment是Kubernetes 1.2起一个新引入的概念，Deployment是Replica Sets更高一层的抽象。Kubernetes Deployment提供了官方的用于更新Pod和Replica Set的方法，您可以在Deployment对象中只描述您所期望的理想状态\(预期的运行状态\)，Deployment控制器为您将现在的实际状态转换成您期望的状态。

Deployment集成了上线部署、滚动升级、创建副本、暂停上线任务，恢复上线任务，回滚到以前某一版本\(成功/稳定\)的Deployment等功能，在某种程度上，Deployment可以帮我们实现无人值守的上线，大大降低我们的上线过程的复杂沟通、操作风险。

Deployment的使用场景

1. 使用Deployment来启动（上线/部署）一个Pod或者ReplicaSet
2. 检查一个Deployment是否成功执行
3. 更新Deployment来重新创建相应的Pods（例如，需要使用一个新的Image）
4. 如果现有的Deployment不稳定，那么回滚到一个早期的稳定的Deployment版本
5. 暂停或者恢复一个Deployment

* Service\(服务\)

服务为一组Pod提供单一稳定的名称和地址。他们作为基本负载均衡器而存在。是一系列Pod以及这些Pod的访问策略的抽象。

