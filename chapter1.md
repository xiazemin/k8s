# 安装

对于Centos来说，默认官方无法直接通过yum来安装K8S，所以我们第一步要做的是把对应的repo加入到yum中。







在两台主机上分别编辑 /etc/yum.repos.d/virt7-docker-common-release.repo



\[virt7-docker-common-release\]

 name=virt7-docker-common-release

 baseurl=http://cbs.centos.org/repos/virt7-docker-common-release/x86\_64/os/

gpgcheck=0





通过yum安装 Kubernetes和etcd







yum -y install --enablerepo=virt7-docker-common-release kubernetes etcd





配置基础环境



在两台主机的hosts文件中加入配置，确保在两个节点中可以通过主机名互访，如果你的网络支持DDNS则可以忽略这一步。







echo  “192.168.0.1 centos-master" &gt;&gt; /etc/hosts

echo  “192.168.0.2 centos-minion" &gt;&gt; /etc/hosts

作为一个经常用Ubuntu的人上手有点不太习惯的地方是Centos默认启用了防火墙，我们需要先关掉它：



sudo systemctl stop firewalld.service

sudo systemctl disable firewalld.service





至于Selinux，个人建议还是保留着吧，至少大部分情况下它还是很有效的工具，如果执意要关掉它的话：



sudo setenforce 0





修改每台主机的 /etc/kubernetes/config配置



\# etcd api地址

KUBE\_ETCD\_SERVERS="--etcd-servers=http://centos-master:2379"



\# 报错日志输出方式

KUBE\_LOGTOSTDERR="--logtostderr=true"



\# 报错日志等级，0表示debug

KUBE\_LOG\_LEVEL="--v=0"



\# 是否支持特权container

KUBE\_ALLOW\_PRIV="--allow-privileged=false"



\# master API

KUBE\_MASTER="--master=http://centos-master:8080"







为master节点配置etcd



vi /etc/etcd/etcd.conf







\# \[member\]

 ETCD\_NAME=default

 ETCD\_DATA\_DIR="/var/lib/etcd/default.etcd"

 ETCD\_LISTEN\_CLIENT\_URLS="http://0.0.0.0:2379"

\#\[cluster\]

ETCD\_ADVERTISE\_CLIENT\_URLS="http://0.0.0.0:2379"Edit /etc/kubernetes/apiserver to appear as such:



\# KUBE监听地址0.0.0.0表示在所有IP上监听

KUBE\_API\_ADDRESS="--address=0.0.0.0"



\# 监听端口

KUBE\_API\_PORT="--port=8080"



\# kubelets监听端口

KUBELET\_PORT="--kubelet-port=10250"



\# 所用的container的IP地址池

KUBE\_SERVICE\_ADDRESSES="--service-cluster-ip-range=192.168.0.0/16"



\# 自定义入参

KUBE\_API\_ARGS=""





启动master节点，按照官方的做法，用了一个shell脚本中的for in遍历。不得不说，我开始喜欢起systemd的管理方式了！







for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler;

do

    systemctl restart $SERVICES

    systemctl enable $SERVICES

    systemctl status $SERVICES

done





这个时候，你的master 节点应该已经成功的运行起来了，结下来的任务就是把minion节点加入进集群中去。







编辑minion节点上的/etc/kubernetes/kubelet：



\# 监听地址，0.0.0.0表示监听所有可用地址

 KUBELET\_ADDRESS="--address=0.0.0.0"

\# 监听端口

KUBELET\_PORT="--port=10250"



\# 设置一个K8s自己的主机标示覆盖真实主机名

KUBELET\_HOSTNAME="--hostname-override=centos-minion"



\# 本地API地址

KUBELET\_API\_SERVER="--api-servers=http://centos-master:8080"



\# 自定义入参

KUBELET\_ARGS="”





同样的for in遍历启动服务



for SERVICES in kube-proxy kubelet docker;

do

    systemctl restart $SERVICES

    systemctl enable $SERVICES

    systemctl status $SERVICES

done





在master节点上执



