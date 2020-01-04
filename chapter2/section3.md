# Kubernetes 部署 Flink



Kubernetes 是目前非常流行的容器编排系统，在其之上可以运行 Web 服务、大数据处理等各类应用。这些应用被打包在一个个非常轻量的容器中，我们通过声明的方式来告知 Kubernetes 要如何部署和扩容这些程序，并对外提供服务。Flink 同样是非常流行的分布式处理框架，它也可以运行在 Kubernetes 之上。将两者相结合，我们就可以得到一个健壮和高可扩的数据处理应用，并且能够更安全地和其它服务共享一个 Kubernetes 集群。



Flink on Kubernetes

Flink on Kubernetes



在 Kubernetes 上部署 Flink 有两种方式：会话集群（Session Cluster）和脚本集群（Job Cluster）。会话集群和独立部署一个 Flink 集群类似，只是底层资源换成了 K8s 容器，而非直接运行在操作系统上。该集群可以提交多个脚本，因此适合运行那些短时脚本和即席查询。脚本集群则是为单个脚本部署一整套服务，包括 JobManager 和 TaskManager，运行结束后这些资源也随即释放。我们需要为每个脚本构建专门的容器镜像，分配独立的资源，因而这种方式可以更好地和其他脚本隔离开，同时便于扩容或缩容。文本将以脚本集群为例，演示如何在 K8s 上运行 Flink 实时处理程序，主要步骤如下：



编译并打包 Flink 脚本 Jar 文件；

构建 Docker 容器镜像，添加 Flink 运行时库和上述 Jar 包；

使用 Kubernetes Job 部署 Flink JobManager 组件；

使用 Kubernetes Service 将 JobManager 服务端口开放到集群中；

使用 Kubernetes Deployment 部署 Flink TaskManager；

配置 Flink JobManager 高可用，需使用 ZooKeeper 和 HDFS；

借助 Flink SavePoint 机制来停止和恢复脚本。

Kubernetes 实验环境

如果手边没有 K8s 实验环境，我们可以用 Minikube 快速搭建一个，以 MacOS 系统为例：



安装 VirtualBox，Minikube 将在虚拟机中启动 K8s 集群；

下载 Minikube 程序，权限修改为可运行，并加入到 PATH 环境变量中；

执行 minikube start，该命令会下载虚拟机镜像，安装 kubelet 和 kubeadm 程序，并构建一个完整的 K8s 集群。如果你在访问网络时遇到问题，可以配置一个代理，并告知 Minikube 使用它；

下载并安装 kubectl 程序，Minikube 已经将该命令指向虚拟机中的 K8s 集群了，所以可以直接运行 kubectl get pods -A 来显示当前正在运行的 K8s Pods：

1

2

3

4

NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE

kube-system   kube-apiserver-minikube            1/1     Running   0          16m

kube-system   etcd-minikube                      1/1     Running   0          15m

kube-system   coredns-5c98db65d4-d4t2h           1/1     Running   0          17m

Flink 实时处理脚本示例

我们可以编写一个简单的实时处理脚本，该脚本会从某个端口中读取文本，分割为单词，并且每 5 秒钟打印一次每个单词出现的次数。以下代码是从 Flink 官方文档 上获取来的，完整的示例项目可以到 GitHub 上查看。



1

2

3

4

5

6

7

8

DataStream&lt;Tuple2&lt;String, Integer&gt;&gt; dataStream = env

    .socketTextStream\("192.168.99.1", 9999\)

    .flatMap\(new Splitter\(\)\)

    .keyBy\(0\)

    .timeWindow\(Time.seconds\(5\)\)

    .sum\(1\);



dataStream.print\(\);

K8s 容器中的程序可以通过 IP 192.168.99.1 来访问 Minikube 宿主机上的服务。因此在运行上述代码之前，需要先在宿主机上执行 nc -lk 9999 命令打开一个端口。



接下来执行 mvn clean package 命令，打包好的 Jar 文件路径为 target/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar。



构建 Docker 容器镜像

Flink 提供了一个官方的容器镜像，可以从 DockerHub 上下载。我们将以这个镜像为基础，构建独立的脚本镜像，将打包好的 Jar 文件放置进去。此外，新版 Flink 已将 Hadoop 依赖从官方发行版中剥离，因此我们在打镜像时也需要包含进去。



简单看一下官方镜像的 Dockerfile，它做了以下几件事情：



将 OpenJDK 1.8 作为基础镜像；

下载并安装 Flink 至 /opt/flink 目录中；

添加 flink 用户和组；

指定入口文件，不过我们会在 K8s 配置中覆盖此项。

1

2

3

4

5

6

7

FROM openjdk:8-jre

ENV FLINK\_HOME=/opt/flink

WORKDIR $FLINK\_HOME

RUN useradd flink && \

  wget -O flink.tgz "$FLINK\_TGZ\_URL" && \

  tar -xf flink.tgz

ENTRYPOINT \["/docker-entrypoint.sh"\]

在此基础上，我们编写新的 Dockerfile：



1

2

3

4

5

FROM flink:1.8.1-scala\_2.12

ARG hadoop\_jar

ARG job\_jar

COPY --chown=flink:flink $hadoop\_jar $job\_jar $FLINK\_HOME/lib/

USER flink

在构建镜像之前，我们需要安装 Docker 命令行工具，并将其指向 Minikube 中的 Docker 服务，这样打出来的镜像才能被 K8s 使用：



1

2

$ brew install docker

$ eval $\(minikube docker-env\)

下载 Hadoop Jar 包，执行以下命令：



$ cd /path/to/Dockerfile

$ cp /path/to/flink-shaded-hadoop-2-uber-2.8.3-7.0.jar hadoop.jar

$ cp /path/to/flink-on-kubernetes-0.0.1-SNAPSHOT-jar-with-dependencies.jar job.jar

$ docker build --build-arg hadoop\_jar=hadoop.jar --build-arg job\_jar=job.jar --tag flink-on-kubernetes:0.0.1 .

脚本镜像打包完毕，可用于部署：



$ docker image ls

REPOSITORY           TAG    IMAGE ID      CREATED         SIZE

flink-on-kubernetes  0.0.1  505d2f11cc57  10 seconds ago  618MB



