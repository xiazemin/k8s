# Kubernetes的几个概念和部署

话说在数据中心虚拟化的大潮中，除了Google以外，每个大玩家都有一个自己的云，例如aws之于亚马逊，阿里云，百度云，腾讯云之流，而Google明显是点开了别的技能树，他的app engine以及后续的Cloud Platform则是基于container技术的实现。对于虚拟机的云，我们完全可以采用一个OpenStack来涵盖他们的功能，而对应Google的，Google则自己推出了他们云屏他的开源实现Kubernetes。

Kubernetes\(以下将简称为K8S\)是一个基于docker container技术的资源管理调度系统。举个不太恰当例子的话，如果docker中位于底层的runc相当于libvirt；docker shell相当于qume的话，那K8S就相当于Openstack了。

这里我们先讲讲几个概念：

Node\(节点\)：在Kubernetes中，节点是实际工作的点，较早版本称为Minion。还是沿用上面不恰当的比喻，这个’node’相当于compute node和controller node。除了docker服务之外，每个节点上还会有Kubelet和 Kube-Proxy服务用语节点间的控制和通讯。

Pod\(容器组\)：这是Kubernetes的基本操作单元，有依赖关系的docker instance要放置在同一个Pod中，也就是因为这重依赖关系，Pod包含的容器都会运行在同一个节点上，共享相同的volumes和network namespace/IP和Port空间。K8S通过Replication Controller（RC）来管理pod，鉴于用户的不同配置，RC可以支持同时启动多个相同的pod用于failover或者load balance。并且给pod定义了4种状态：Pending（等待）, Running（运行中）, Succeeded（成功）, Failed（失败）。

引入pod这个概念主要是考虑到container设计哲学中的“低耦合”和一个复杂系统无可避免的“强依赖”之前的冲突。

Service\(服务\)和Volume\(存储卷\)：对应的就是一个独立的port和一个pod之间可以共享访问的volume。如果你之前用过docker，那么这两个概念事实上就对应了docker run命令中的“-p”和“-v”入参。真要说的话，service相当于Openstack的IP，只不过Openstack提供的是一个IP地址而K8S提供的是一个port；volume则是多个VM之间的“共享存储”。

Label\(标签\)：不同于pod那种强依赖关系，label是一个由用户指定的，可用于区分Pod、Service、Replication Controller的标识符，同一个Pod、Service、 Replication Controller可以有多个label，但是每个label的key只能对应一个value。比如有多个pod中都启动了相同的service用于负载均衡，那将这多个pod定义相同的label之后，可以简单的将所有service的请求简单的通过label匹配就可以转发给对应的pod。

Proxy\(代理\)：反向代理，Proxy会根据Load Balancer规则将外网请求分发到后端正确的容器处理。

Namespace\(命名空间\)：通过将系统内部的对象划分到不同的Namespace中，形成逻辑上的不同分组，便于在共享使用整个集群的资源同时形成不同的权限规则。

Annotation\(注解\)：与Label类似，是用户为了方便管理，任意定义的信息。

![](/assets/import.png)

于container本身相较于VM是一个轻量级的实现，尽管从逻辑上我们有namespace、label、pod、container几层的隔离，但事实上包括container本身都是一个基于逻辑意义上的隔离，并没有一个基于软件调用层面的stack划分。更薄的层级关系让性能损耗降到最低。当然，由于没有严格的stack划分，资源隔离直接受制于cgroup，用户态隔离则直接受制于kernel中的namespace，不具有严格意义上的逐级隔离。

