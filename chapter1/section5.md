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

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86\_64

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



