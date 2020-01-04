# Label控制Pod位置



Label是Kubernetes系列中一个核心概念。是一组绑定到K8s资源对象上的key/value对。同一个对象的labels属性的key必须唯一。label可以附加到各种资源对象上，如Node,Pod,Service,RC等。



通过给指定的资源对象捆绑一个或多个不用的label来实现多维度的资源分组管理功能，以便于灵活，方便地进行资源分配，调度，配置，部署等管理工作。



默认配置下，Scheduler 会将 Pod 调度到所有可用的 Node。不过有些实际情况我们希望将 Pod 部署到指定的 Node，比如将有大量磁盘 I/O 的 Pod 部署到配置了 SSD 的 Node；或者 Pod 需要 GPU，需要运行在配置了 GPU 的节点上。



下面我们来实际的操作下，比如执行如下命令标注 k8s-node1 是配置了 SSD的节点。



kubectl label node k8snode1 disktype=ssd

然后通过 kubectl get node --show-labels 查看节点的 label。



博客01.png



可以看到disktype=ssd 已经成功添加到 k8snode1，除了 disktype，Node 还有几个 Kubernetes 自己维护的 label。有了 disktype 这个自定义 label，接下来就可以指定将 Pod 部署到 k8snod1。比如我编辑nginx.yml，增加nodeSelector标签，指定将此Pod部署到具有ssd属性的Node上去。



博客02.png



最后通过kubectl get pod -o wide。







如果要删除 label disktype，就执行如下命令删除即可：



     



kubectl label node k8s-node1 disktype-





但是要注意已经部署的 Pod 并不会重新部署，依然在 k8snode1 上运行。可能会有人说了，那怎么让Pod变回原样呢也就是分配到多个node上，那就需要一个笨方法了（至少在目前我学习的方法里面只会这样操作），就是在刚才编辑的那个nginx.yml文件里面删除nodeSelector标签，然后在利用kubectl apply重新部署，Kubernetes 会删除之前的 Pod 并调度和运行新的 Pod。

