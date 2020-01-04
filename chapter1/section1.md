## Kubernetes是什么 {#item-1}



Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。可以在物理或虚拟机的Kubernetes集群上运行容器化应用，Kubernetes能提供一个以“容器为中心的基础架构”。如果你曾经用过Docker容器技术部署容器，那么可以将Docker看成Kubernetes内部使用的低级别组件。Kubernetes不仅仅支持Docker，还支持Rocket，这是另一种容器技术。



通过Kubernetes你可以:



自动化容器的部署和复制

快速扩展应用

将容器组织成组，并且提供容器间的负载均衡

无缝对接新的应用功能

想理解Kubernetes集群，需要先搞明白其中的几个重要概念。



Deployment

Deployment负责创建和更新应用，当创建Deployment后，Kubernetes master 会将Deployment创建好的应用实例调度到集群中的各个节点。

应用实例创建完成后，Kubernetes Deployment Controller会持续监视这些实例。如果管理实例的节点被关闭或删除，那么 Deployment Controller将会替换它们，实现自我修复能力。



Pod

创建Deployment时，Kubernetes会创建了一个Pod来托管应用。Pod是Kubernetes中一个抽象化概念，由一个或多个容器组合在一起得共享资源。Pod是独立运行的基本单位，包含一组容器和卷。同一个Pod里的容器共享同一个网络命名空间，可以使用localhost互相通信。Pod是短暂的，不是持续性实体。

当在Kubernetes上创建Deployment时，该Deployment将会创建具有容器的Pods（而不会直接创建容器），每个Pod将被绑定调度到Node节点上，并一直保持在那里直到被终止（根据配置策略）或删除。在节点出现故障的情况下，群集中的其他可用节点上将会调度之前相同的Pod。



Node

一个Pod总是在一个（Node）节点上运行，Node是Kubernetes中的工作节点，可以是虚拟机或物理机。每个Node由 Master管理，Node上可以有多个pod，Kubernetes Master会自动处理群集中Node的pod调度，同时Master的自动调度会考虑每个Node上的可用资源。



Replication Controller

Replication Controller确保任意时间都有指定数量的Pod“副本”在运行。如果为某个Pod创建了Replication Controller并且指定3个副本，它会创建3个Pod，并且持续监控它们。如果某个Pod不响应，那么Replication Controller会替换它。如果在运行中将副本总数改为5，Replication Controller会立刻启动2个新Pod，保证总数为5。



Service

事实上，Pod是有生命周期的。当一个工作节点\(Node\)销毁时，节点上运行的Pod也会销毁，然后通过ReplicationController动态创建新的Pods来保持应用的运行。

举个例子，考虑一个图片处理 backend，它运行了3个副本，这些副本是可互换的 —— 前端不需要关心它们调用了哪个 backend 副本。也就是说，Kubernetes集群中的每个Pod都有一个独立的IP地址，因此需要有一种方式来自动协调各个Pod之间的变化，以便应用能够持续运行。

Kubernetes中的Service 是一个抽象的概念，它定义了Pod的逻辑分组和一种可以访问它们的策略，让你的这组Pods能被Service访问。借助Service，可以方便的实现服务发现与负载均衡。

Service可以被指定四种类型：



ClusterIP - 在集群中内部IP上暴露服务。此类型使Service只能从群集中访问。

NodePort - 通过每个 Node 上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求 &lt;NodeIP&gt;:&lt;NodePort&gt;，可以从集群的外部访问一个 NodePort 服务。

LoadBalancer - 使用云提供商的负载均衡器（如果支持），可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务。\(一般常用此类型向外暴露端口，并做负载均衡\)

ExternalName - 通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容，没有任何类型代理被创建。

Label

你可以赋予标签（键值对）来标识你的Pod、Deployment、Service。之后就可以通过选择标签来做具体的指令。

