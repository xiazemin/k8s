## 使用Minikube {#item-2}

Minikube 是一个使我们很容易在本地运行 kubernetes 的工具，由Kubernetes社区开发。



准备工作

Node.js

Docker

VM - VirtualBox

Minikube

Kubectl

对于Node.js、Docker、VirtualBox的安装，在这里不做详细介绍。可以直接到官网下载安装最新稳定版本。



创建Minikube集群

使用curl下载并安装最新版本Minikube：



$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 && \

  chmod +x minikube && \

  sudo mv minikube /usr/local/bin/

使用Homebrew下载kubectl命令管理工具：



$ brew install kubectl

默认的VM驱动程序VirtualBox，因此可直接启动Minikube：



$ minikube start

接下来会打印出类似信息...



Starting local Kubernetes v1.9.4 cluster...

Starting VM...

Getting VM IP address...

Moving files into cluster...

Setting up certs...

Connecting to cluster...

Setting up kubeconfig...

Starting cluster components...

Kubectl is now configured to use the cluster.

Loading cached images from config file.

验证kubectl是否安装成功：



$ kubectl cluster-info

会打印出类似的信息：



Kubernetes master is running at https://192.168.99.100:8443

创建Node.js应用

编写应用程序。将这段代码保存在一个名为hellonode的文件夹中，文件名server.js:



var http = require\('http'\);



var handleRequest = function\(request, response\) {

  console.log\('Received request for URL: ' + request.url\);

  response.writeHead\(200\);

  response.end\('Hello World!'\);

};

var www = http.createServer\(handleRequest\);

www.listen\(3000\);

启动应用：



$ node server.js

现在可以在 http://localhost:3000 中查看到“Hello World！”消息。

Ctrl-C停止正在运行的Node.js服务器。



创建Docker容器镜像

在hellonode文件夹中创建一个Dockerfile命名的文件。



FROM node:8.10.0

EXPOSE 3000

COPY server.js .

CMD node server.js

我们使用Minikube，而不是将Docker镜像push到registry，可以使用与Minikube VM相同的Docker主机构建镜像，以使镜像自动存在。为此，请确保使用Minikube Docker守护进程：



$ eval $\(minikube docker-env\)

注意：如果不在使用Minikube主机时，可以通过运行eval $\(minikube docker-env -u\)来撤消此更改。



使用Minikube Docker守护进程build Docker镜像：



$ docker build -t hello-node:v1 .

创建Deployment

Kubernetes Deployment 是检查Pod的健康状况，如果它终止，则重新启动一个Pod的容器，Deployment管理Pod的创建和扩展。



使用kubectl run命令创建Deployment来管理Pod。



$ kubectl run hello-node --image=hello-node:v1 --port=3000

查看Deployment：



$ kubectl get deployments

输出：



NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE

hello-node   1         1         1            1           5s

查看Pod：



$ kubectl get pods

输出：



NAME                          READY     STATUS    RESTARTS   AGE

hello-node-6ddb5576c9-644xn   1/1       Running   0          1m

查看deployment的详细信息：



$ kubectl describe deployment

创建Service

Pod只能通过Kubernetes群集内部IP访问。要使hello-node容器能从Kubernetes虚拟网络外部访问，须要使用Kubernetes Service暴露Pod。



我们可以使用kubectl expose命令将Pod暴露到外部环境：



$ kubectl expose deployment hello-node --type=LoadBalancer

查看Service：



$ kubectl get services

输出：



NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT\(S\)          AGE

hello-node   LoadBalancer   10.110.94.17   &lt;pending&gt;     8080:32081/TCP   2m

kubernetes   ClusterIP      10.96.0.1      &lt;none&gt;        443/TCP          6d

查看详细信息：



$ kubectl describe service hello-node

输出：



Name:                     hello-node

Namespace:                default

Labels:                   run=hello-node

Annotations:              &lt;none&gt;

Selector:                 run=hello-node

Type:                     LoadBalancer

IP:                       10.110.94.17

Port:                     &lt;unset&gt;  8080/TCP

TargetPort:               8080/TCP

NodePort:                 &lt;unset&gt;  32081/TCP

Endpoints:                172.17.0.5:8080

Session Affinity:         None

External Traffic Policy:  Cluster

Events:                   &lt;none&gt;

通过--type=LoadBalancer 来在群集外暴露Service，在支持负载均衡的云提供商上，将配置外部IP\(EXTERNAL-IP，在Minikube显示为：&lt;pending&gt;\)地址来访问Service。在Minikube上，该LoadBalancer type使服务可以通过minikube Service 命令访问。



$ minikube service hello-node

将打开浏览器，在本地IP地址为应用提供服务，显示“Hello World”的消息。



可以查看到日志：



$ kubectl logs &lt;pod-name&gt;  //exp: kubectl logs hello-node-6ddb5576c9-644xn

// 可通过 kubectl get pods 查看pod-name

扩展实例

根据线上需求，扩容和缩容是常会遇到的问题。Scaling 是通过更改 Deployment 中的副本数量实现的。一旦有多个实例，就可以滚动更新，而不会停止服务。通过kubectl scale指令来扩容和缩容。



$ kubectl scale deployments/hello-node --replicas=4

在查看pods：



$ kubectl get pods

输出：



NAME                         READY     STATUS    RESTARTS   AGE

hello-node-9f5f775d6-6qdmn   1/1       Running   0          3s

hello-node-9f5f775d6-9mrm6   1/1       Running   0          3s

hello-node-9f5f775d6-jxb8z   1/1       Running   0          3s

hello-node-9f5f775d6-tx8kg   1/1       Running   0          11m

总共有4个实例，那么就可通过Service 的 --type=LoadBalancer 进行负载均衡。



更新应用程序

编辑server.js文件以返回新消息：



response.end\('Hello World Again!'\);

docker build新版本镜像：



$ docker build -t hello-node:v2 .

Deployment更新镜像：



$ kubectl set image deployment/hello-node hello-node=hello-node:v2

再次在浏览器查看消息：



$ minikube service hello-node

清理删除

删除在群集中创建的资源：



$ kubectl delete service hello-node

$ kubectl delete deployment hello-node

查看pods:



$ kubectl get pods

输出：



NAME                         READY     STATUS        RESTARTS   AGE

hello-node-9f5f775d6-6qdmn   1/1       Terminating   0          6m

hello-node-9f5f775d6-9mrm6   1/1       Terminating   0          6m

hello-node-9f5f775d6-jxb8z   1/1       Terminating   0          6m

hello-node-9f5f775d6-tx8kg   1/1       Terminating   0          18m

全部在停止中... 稍等一分钟，再查看，输出 No resources found.



停止

$ minikube stop

输出：



Stopping local Kubernetes cluster...

Machine stopped.

完毕



