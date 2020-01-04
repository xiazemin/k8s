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

