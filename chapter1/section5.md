# kubeadm init

主节点部署

当上述步骤完成后，我们依照以下步骤来完成主节点的安装：

1.Kubeadm以及相关工具包的安装

安装脚本如下所示：

复制代码

\#配置源

echo '\#k8s

\[kubernetes\]

name=Kubernetes

baseurl=[https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86\_64](https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64)

enabled=1

gpgcheck=0

'&gt;/etc/yum.repos.d/kubernetes.repo

\#kubeadm和相关工具包

yum -y install kubelet kubeadm kubectl kubernetes-cni

复制代码

注意，以上脚本使用阿里云镜像进行安装。

安装完成之后，需要重启kubelet：

systemctl daemon-reload

systemctl enable kubelet

2.批量拉取k8s相关镜像

如果使用代理、国际网络或者指定镜像库地址，此步骤可以忽略。在国内，由于国际网络问题，k8s相关镜像在国内可能无法下载，因此我们需要手动准备。

首先，我们先使用“kubeadm config”命令来查看kubeadm相关镜像的列表：

kubeadm config images list

接下来我们可以从其他仓库批量下载镜像并且修改镜像标签：

复制代码

\#批量下载镜像

kubeadm config images list \|sed -e 's/^/docker pull /g' -e 's\#k8s.gcr.io\#docker.io/mirrorgooglecontainers\#g' \|sh -x

\#批量命名镜像

docker images \|grep mirrorgooglecontainers \|awk '{print "docker tag ",$1":"$2,$1":"$2}' \|sed -e 's\# mirrorgooglecontainers\# k8s.gcr.io\#2' \|sh -x

\#批量删除mirrorgooglecontainers镜像

docker images \|grep mirrorgooglecontainers \|awk '{print "docker rmi ", $1":"$2}' \|sh -x

\# coredns没包含在docker.io/mirrorgooglecontainers中

docker pull coredns/coredns:1.3.1

docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

docker rmi coredns/coredns:1.3.1

复制代码

注：coredns没包含在docker.io/mirrorgooglecontainers中，需要手工从coredns官方镜像转换下。

### 3.使用“kubeadm init”启动k8s主节点

在前面，我们讲解过了“kubeadm init”命令可以用于启动一个Kubernetes主节点，语法如下所示：

kubeadm init \[flags\]

其中主要的参数如下所示：

| 可选参数 | 说明 |
| :--- | :--- |
| --apiserver-advertise-address | 指定API Server地址 |
| --apiserver-bind-port | 指定绑定的API Server端口，默认值为6443 |
| --apiserver-cert-extra-sans | 指定API Server的服务器证书 |
| --cert-dir | 指定证书的路径 |
| --dry-run | 输出将要执行的操作，不做任何改变 |
| --feature-gates | 指定功能配置键值对，可控制是否启用各种功能 |
| -h, --help | 输出init命令的帮助信息 |
| --ignore-preflight-errors | 忽视检查项错误列表，例如“IsPrivilegedUser,Swap”，如填写为 'all' 则将忽视所有的检查项错误 |
| --kubernetes-version | 指定Kubernetes版本 |
| --node-name | 指定节点名称 |
| --pod-network-cidr | 指定pod网络IP地址段 |
| --service-cidr | 指定service的IP地址段 |
| --service-dns-domain | 指定Service的域名，默认为“cluster.local” |
| --skip-token-print | 不打印Token |
| --token | 指定token |
| --token-ttl | 指定token有效时间，如果设置为“0”，则永不过期 |
| --image-repository | 指定镜像仓库地址，默认为"k8s.gcr.io" |

值得注意的是，如上所述，如果我们不想每次都手动批量拉取镜像，我们可以使用参数“--image-repository”来指定第三方镜像仓库，如下述命令所示：

```
kubeadm init --kubernetes-version=v1.15.0  --apiserver-advertise-address=172.16.2.201  --pod-network-cidr=10.0.0.0/16 --service-cidr 11.0.0.0/12 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

“kubeadm init”命令会执行系列步骤来保障启动一个k8s主节点，我们可以通过命令“kubeadm init --dry-run”来查看其将进行的一些步骤，了解了其动作，我们才能保障在安装的过程中处理起来游刃有余

其主体常规步骤（部分步骤根据参数会有变动）如下：

确定Kubernetes版本。

预检。出现错误则退出安装，比如虚拟内存（swap）没关闭，端口被占用。出现错误时，请用心按照按照提示进行处理，不推荐使用“--ignore-preflight-errors”来忽略。

写入kubelet配置。

生成自签名的CA证书（可指定已有证书）。

将 kubeconfig 文件写入 /etc/kubernetes/ 目录以便 kubelet、controller-manager 和 scheduler 用来连接到 API server，它们每一个都有自己的身份标识，同时生成一个名为 admin.conf 的独立的kubeconfig文件，用于管理操作（我们下面会用到）。

为 kube-apiserver、kube-controller-manager和kube-scheduler生成静态Pod的定义文件。如果没有提供外部的etcd服务的话，也会为etcd生成一份额外的静态Pod定义文件。这些静态Pod的定义文件会写入到“/etc/kubernetes/manifests”目录（如下图所示），kubelet会监视这个目录以便在系统启动的时候创建这些Pod。

注意：静态Pod是由kubelet进行管理，仅存在于特定节点上的Pod。它们不能通过API Server进行管理，无法与ReplicationController、Deployment或DaemonSet进行关联，并且kubelet也无法对其健康检查。静态 Pod 始终绑定在某一个kubelet，并且始终运行在同一个节点上。

对master节点应用labels和taints以便不会在它上面运行其它的工作负载，也就是说master节点只做管理不干活。

生成令牌以便其它节点注册。

执行必要配置（比如集群ConfigMap，RBAC等）。

安装“CoreDNS”组件（在 1.11 版本以及更新版本的Kubernetes中，CoreDNS是默认的DNS服务器）和“kube-proxy”组件。

4.启动k8s主节点

根据前面的规划，以及刚才讲述的“kubeadm init”命令语法和执行步骤，我们使用如下命令来启动k8s集群主节点：

kubeadm init --kubernetes-version=v1.15.0  --apiserver-advertise-address=172.16.2.201  --pod-network-cidr=10.0.0.0/16 --service-cidr 11.0.0.0/16

其中，kubernetes version为v1.15.0，apiserver地址为172.16.2.201，pod IP段为10.0.0.0/16。

集群创建成功后，注意这一条命令需要保存好，以便后续将节点添加到集群时使用：



kubeadm join 172.16.2.201:6443 --token jx82lw.8ephcufcot5j06v7 \

    --discovery-token-ca-cert-hash sha256:180a8dfb45398cc6c3addd84a61c1

令牌是用于主节点和新添加的节点之间进行相互身份验证的，因此需要确保其安全，因为任何人一旦知道了这些令牌，就可以随便给集群添加节点。如果令牌过期了，我们可以使用 “kubeadm token”命令来列出、创建和删除这类令牌，具体操作见后续的《集群异常解决方案》。



 



5.kubectl认证

 集群主节点启动之后，我们需要使用kubectl来管理集群，在开始前，我们需要设置其配置文件进行认证。



这里我们使用root账户，命令如下所示：



\#kubectl认证

export KUBECONFIG=/etc/kubernetes/admin.conf

如果是非root账户，则需要使用以下命令：



\# 如果是非root用户

$ mkdir -p $HOME/.kube

$ cp -i /etc/kubernetes/admin.conf

$HOME/.kube/config$ chown $\(id -u\):$\(id -g\) $HOME/.kube/config

 

6.安装flannel网络插件

 这里我们使用默认的网络组件flannel，相关安装命令如下如下所示：



kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

命令“kubectl apply”可以用于创建和更新资源，以上命令使用了网络路径的yaml来进行创建flanner：







 



7.检查集群状态

安装完成之后，我们可以使用以下命令来检查集群组件是否运行正常：



kubectl get cs







同时，我们需要确认相关pod已经正常运行，如下所示：



kubectl get pods -n kube-system -o wide







如果coredns崩溃或者其他pod崩溃，可参考后续章节的常见问题进行解决，请注意确保这些pod正常运行（Running状态）后再添加工作节点。



如果命名空间“kube-system”下的pod均正常运行，那么我们的主节点已经成功的启动了，接下来我们来完成工作节点的部署。



 



工作节点部署

这里我们以Node1节点为例进行安装。开始安装之前，请确认已经完成之前的步骤（设置主机、IP、系统、Docker和防火墙等）。注意主机名、IP等配置不要出现重复和错误。



1.安装 kubelet和kubeadm

kubelet是节点代理，而kubeadm则用于将当前节点加入集群。下面我们就开始进行安装，



安装命令如下所示：



复制代码

\#配置源

echo '\#k8s

\[kubernetes\]

name=Kubernetes

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86\_64

enabled=1

gpgcheck=0

'&gt;/etc/yum.repos.d/kubernetes.repo

\#kubeadm和相关工具包

yum -y install kubelet kubeadm

复制代码

重启kubelet：



systemctl daemon-reload

systemctl enable kubelet

 



2.拉取相关镜像

请参考上面小节中的《批量拉取k8s相关镜像》，此处略过。



 



3.使用“kubeadm join”将当前节点加入集群

“kubeadm join”命令可以启动一个Kubernetes工作节点并且将其加入到集群，语法如下所示：



kubeadm join \[api-server-endpoint\] \[flags\]

使用“kubeadm join”就相对简单多了，这里，我们回到前面，找到使用“kubeadm init”启动主节点时打印出来的“kubeadm join”脚本进行执行：



kubeadm join 172.16.2.201:6443 --token jx82lw.8ephcufcot5j06v7 \

    --discovery-token-ca-cert-hash sha256:180a8dfb45398cc6c3addd84a61c1bd4364297da1e91611c8c46a976dc12ff17

如未保存该命令或者token已过期，请参考后续章节的常见问题。这里，正常情况下加入成功后如下所示：







加入集成成功之后，k8s就会自动调度Pod，这时我们仅需耐心等待即可。



 



4.复制admin.conf并且设置配置

为了在工作节点上也能使用kubectl，而kubectl命令需要使用kubernetes-admin来运行，因此我们需要将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到工作节点相同目录下，这里推荐使用scp进行复制，语法如下所示：



\#复制admin.conf，请在主节点服务器上执行此命令

scp /etc/kubernetes/admin.conf {当前工作节点IP}:/etc/kubernetes/admin.conf

具体执行内容如下：



scp /etc/kubernetes/admin.conf 172.16.2.202:/etc/kubernetes/admin.conf

scp /etc/kubernetes/admin.conf 172.16.2.203:/etc/kubernetes/admin.conf

复制时需要输入相关节点的root账户的密码：







复制完成之后，我们就可以设置kubectl的配置文件了，以便我们在工作节点上也可以使用kubectl来管理k8s集群：



\#设置kubeconfig文件

export KUBECONFIG=/etc/kubernetes/admin.conf

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" &gt;&gt; ~/.bash\_profile

至此，k8s工作节点的部署初步完成。接下来，我们需要以同样的方式将其他工作节点加入到集群之中。



 



5.查看集群节点状态

集群创建完成之后，我们可以输入以下命令来查看当前节点状态：



kubectl get nodes







接下来，我们可以开始按需安装仪表盘以及部署应用了。



 



安装仪表盘

命令如下所示：



\#如果使用代理或者使用国际网络，则可跳过此步骤

docker pull mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1

docker tag mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

\#安装仪表盘

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml





复制代码

创建admin权限：

echo '

---

apiVersion: v1

kind: ServiceAccount

metadata:

  labels:

    k8s-app: kubernetes-dashboard

  name: kubernetes-dashboard-admin

  namespace: kube-system

---

apiVersion: rbac.authorization.k8s.io/v1beta1

kind: ClusterRoleBinding

metadata:

  name: kubernetes-dashboard-admin

  labels:

    k8s-app: kubernetes-dashboard

roleRef:

  apiGroup: rbac.authorization.k8s.io

  kind: ClusterRole

  name: cluster-admin

subjects:

- kind: ServiceAccount

  name: kubernetes-dashboard-admin

  namespace: kube-system' &gt;kubernetes-dashboard-admin.rbac.yaml



kubectl create -f kubernetes-dashboard-admin.rbac.yaml

复制代码





使用命令得到token：



\#获取token名称

kubectl -n kube-system get secret \| grep kubernetes-dashboard-admin

\#根据名称拿到token

kubectl describe -n kube-system secret/kubernetes-dashboard-admin-token-lphq4





接下来可以使用以下命令来访问面板：



kubectl proxy







访问地址如下所示：



http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

