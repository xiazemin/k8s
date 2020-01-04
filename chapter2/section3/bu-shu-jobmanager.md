首先，我们通过创建 Kubernetes Job 对象来部署 Flink JobManager。Job 和 Deployment 是 K8s 中两种不同的管理方式，他们都可以通过启动和维护多个 Pod 来执行任务。不同的是，Job 会在 Pod 执行完成后自动退出，而 Deployment 则会不断重启 Pod，直到手工删除。Pod 成功与否是通过命令行返回状态判断的，如果异常退出，Job 也会负责重启它。因此，Job 更适合用来部署 Flink 应用，当我们手工关闭一个 Flink 脚本时，K8s 就不会错误地重新启动它。



以下是 jobmanager.yml 配置文件：



1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

apiVersion: batch/v1

kind: Job

metadata:

  name: ${JOB}-jobmanager

spec:

  template:

    metadata:

      labels:

        app: flink

        instance: ${JOB}-jobmanager

    spec:

      restartPolicy: OnFailure

      containers:

      - name: jobmanager

        image: flink-on-kubernetes:0.0.1

        command: \["/opt/flink/bin/standalone-job.sh"\]

        args: \["start-foreground",

               "-Djobmanager.rpc.address=${JOB}-jobmanager",

               "-Dparallelism.default=1",

               "-Dblob.server.port=6124",

               "-Dqueryable-state.server.ports=6125"\]

        ports:

        - containerPort: 6123

          name: rpc

        - containerPort: 6124

          name: blob

        - containerPort: 6125

          name: query

        - containerPort: 8081

          name: ui

${JOB} 变量可以使用 envsubst 命令来替换，这样同一份配置文件就能够为多个脚本使用了；

容器的入口修改为了 standalone-job.sh，这是 Flink 的官方脚本，会以前台模式启动 JobManager，扫描类加载路径中的 Main-Class 作为脚本入口，我们也可以使用 -j 参数来指定完整的类名。之后，这个脚本会被自动提交到集群中。

JobManager 的 RPC 地址修改为了 Kubernetes Service 的名称，我们将在下文创建。集群中的其他组件将通过这个名称来访问 JobManager。

Flink Blob Server & Queryable State Server 的端口号默认是随机的，为了方便将其开放到集群中，我们修改为了固定端口。

使用 kubectl 命令创建对象，并查看状态：



1

2

3

4

5

$ export JOB=flink-on-kubernetes

$ envsubst &lt;jobmanager.yml \| kubectl create -f -

$ kubectl get pod

NAME                                   READY   STATUS    RESTARTS   AGE

flink-on-kubernetes-jobmanager-kc4kq   1/1     Running   0          2m26s

随后，我们创建一个 K8s Service 来将 JobManager 的端口开放出来，以便 TaskManager 前来注册：



service.yml



1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

apiVersion: v1

kind: Service

metadata:

  name: ${JOB}-jobmanager

spec:

  selector:

    app: flink

    instance: ${JOB}-jobmanager

  type: NodePort

  ports:

  - name: rpc

    port: 6123

  - name: blob

    port: 6124

  - name: query

    port: 6125

  - name: ui

    port: 8081

这里 type: NodePort 是必要的，因为通过这项配置，我们可以在 K8s 集群之外访问 JobManager UI 和 RESTful API。



1

2

3

4

$ envsubst &lt;service.yml \| kubectl create -f -

$ kubectl get service

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT\(S\)                                                      AGE

flink-on-kubernetes-jobmanager   NodePort    10.109.78.143   &lt;none&gt;        6123:31476/TCP,6124:32268/TCP,6125:31602/TCP,8081:31254/TCP  15m

我们可以看到，Flink Dashboard 开放在了虚拟机的 31254 端口上。Minikube 提供了一个命令，可以获取到 K8s 服务的访问地址：



1

2

3

4

5

$ minikube service $JOB-jobmanager --url

http://192.168.99.108:31476

http://192.168.99.108:32268

http://192.168.99.108:31602

http://192.168.99.108:31254

部署 TaskManager

taskmanager.yml



1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

apiVersion: apps/v1

kind: Deployment

metadata:

  name: ${JOB}-taskmanager

spec:

  selector:

    matchLabels:

      app: flink

      instance: ${JOB}-taskmanager

  replicas: 1

  template:

    metadata:

      labels:

        app: flink

        instance: ${JOB}-taskmanager

    spec:

      containers:

      - name: taskmanager

        image: flink-on-kubernetes:0.0.1

        command: \["/opt/flink/bin/taskmanager.sh"\]

        args: \["start-foreground", "-Djobmanager.rpc.address=${JOB}-jobmanager"\]

通过修改 replicas 配置，我们可以开启多个 TaskManager。镜像中的 taskmanager.numberOfTaskSlots 参数默认为 1，这也是我们推荐的配置，因为扩容缩容方面的工作应该交由 K8s 来完成，而非直接使用 TaskManager 的槽位机制。



至此，Flink 脚本集群已经在运行中了。我们在之前已经打开的 nc 命令窗口中输入一些文本：



1

2

3

$ nc -lk 9999

hello world

hello flink

打开另一个终端，查看 TaskManager 的标准输出日志：



1

2

3

4

$ kubectl logs -f -l instance=$JOB-taskmanager

\(hello,2\)

\(flink,1\)

\(world,1\)

开启高可用模式

可用性方面，上述配置中的 TaskManager 如果发生故障退出，K8s 会自动进行重启，Flink 会从上一个 Checkpoint 中恢复工作。但是，JobManager 仍然存在单点问题，因此需要开启 HA 模式，配合 ZooKeeper 和分布式文件系统（如 HDFS）来实现 JobManager 的高可用。在独立集群中，我们需要运行多个 JobManager，作为主备服务器。然而在 K8s 模式下，我们只需开启一个 JobManager，当其异常退出后，K8s 会负责重启，新的 JobManager 将从 ZooKeeper 和 HDFS 中读取最近的工作状态，自动恢复运行。



开启 HA 模式需要修改 JobManager 和 TaskManager 的启动命令：



jobmanager-ha.yml



1

2

3

4

5

6

7

8

9

10

11

12

13

command: \["/opt/flink/bin/standalone-job.sh"\]

args: \["start-foreground",

       "-Djobmanager.rpc.address=${JOB}-jobmanager",

       "-Dparallelism.default=1",

       "-Dblob.server.port=6124",

       "-Dqueryable-state.server.ports=6125",

       "-Dhigh-availability=zookeeper",

       "-Dhigh-availability.zookeeper.quorum=192.168.99.1:2181",

       "-Dhigh-availability.zookeeper.path.root=/flink",

       "-Dhigh-availability.cluster-id=/${JOB}",

       "-Dhigh-availability.storageDir=hdfs://192.168.99.1:9000/flink/recovery",

       "-Dhigh-availability.jobmanager.port=6123",

       \]

taskmanager-ha.yml



1

2

3

4

5

6

7

8

command: \["/opt/flink/bin/taskmanager.sh"\]

args: \["start-foreground",

       "-Dhigh-availability=zookeeper",

       "-Dhigh-availability.zookeeper.quorum=192.168.99.1:2181",

       "-Dhigh-availability.zookeeper.path.root=/flink",

       "-Dhigh-availability.cluster-id=/${JOB}",

       "-Dhigh-availability.storageDir=hdfs://192.168.99.1:9000/flink/recovery",

       \]

准备好 ZooKeeper 和 HDFS 测试环境，该配置中使用的是宿主机上的 2181 和 9000 端口；

Flink 集群基本信息会存储在 ZooKeeper 的 /flink/${JOB} 目录下；

Checkpoint 数据会存储在 HDFS 的 /flink/recovery 目录下。使用前，请先确保 Flink 有权限访问 HDFS 的 /flink 目录；

jobmanager.rpc.address 选项从 TaskManager 的启动命令中去除了，是因为在 HA 模式下，TaskManager 会通过访问 ZooKeeper 来获取到当前 JobManager 的连接信息。需要注意的是，HA 模式下的 JobManager RPC 端口默认是随机的，我们需要使用 high-availability.jobmanager.port 配置项将其固定下来，方便在 K8s Service 中开放。

管理 Flink 脚本

我们可以通过 RESTful API 来与 Flink 集群交互，其端口号默认与 Dashboard UI 一致。在宿主机上安装 Flink 命令行工具，传入 -m 参数来指定目标集群：



1

2

3

4

$ bin/flink list -m 192.168.99.108:30206

------------------ Running/Restarting Jobs -------------------

24.08.2019 12:50:28 : 00000000000000000000000000000000 : Window WordCount \(RUNNING\)

--------------------------------------------------------------

在 HA 模式下，Flink 脚本 ID 默认为 00000000000000000000000000000000，我们可以使用这个 ID 来手工停止脚本，并生成一个 SavePoint 快照：



1

2

$ bin/flink cancel -m 192.168.99.108:30206 -s hdfs://192.168.99.1:9000/flink/savepoints/ 00000000000000000000000000000000

Cancelled job 00000000000000000000000000000000. Savepoint stored in hdfs://192.168.99.1:9000/flink/savepoints/savepoint-000000-f776c8e50a0c.

执行完毕后，可以看到 K8s Job 对象的状态变为了已完成：



1

2

3

$ kubectl get job

NAME                             COMPLETIONS   DURATION   AGE

flink-on-kubernetes-jobmanager   1/1           4m40s      7m14s

重新启动脚本前，我们需要先将配置从 K8s 中删除：



1

2

$ kubectl delete job $JOB-jobmanager

$ kubectl delete deployment $JOB-taskmanager

然后在 JobManager 的启动命令中加入 --fromSavepoint 参数：



1

2

3

4

5

command: \["/opt/flink/bin/standalone-job.sh"\]

args: \["start-foreground",

       ...

       "--fromSavepoint", "${SAVEPOINT}",

       \]

使用刚才得到的 SavePoint 路径替换该变量，并启动 JobManager：



1

2

$ export SAVEPOINT=hdfs://192.168.99.1:9000/flink/savepoints/savepoint-000000-f776c8e50a0c

$ envsubst &lt;jobmanager-savepoint.yml \| kubectl create -f -

需要注意的是，SavePoint 必须和 HA 模式配合使用，因为当 JobManager 异常退出、K8s 重启它时，都会传入 --fromSavepoint，使脚本进入一个异常的状态。而在开启 HA 模式时，JobManager 会优先读取最近的 CheckPoint 并从中恢复，忽略命令行中传入的 SavePoint。



扩容

有两种方式可以对 Flink 脚本进行扩容。第一种方式是用上文提到的 SavePoint 机制：手动关闭脚本，并使用新的 replicas 和 parallelism.default 参数进行重启；另一种方式则是使用 flink modify 命令行工具，该工具的工作机理和人工操作类似，也是先用 SavePoint 停止脚本，然后以新的并发度启动。在使用第二种方式前，我们需要在启动命令中指定默认的 SavePoint 路径：



1

2

3

4

5

command: \["/opt/flink/bin/standalone-job.sh"\]

args: \["start-foreground",

       ...

       "-Dstate.savepoints.dir=hdfs://192.168.99.1:9000/flink/savepoints/",

       \]

然后，使用 kubectl scale 命令调整 TaskManager 的个数；



1

2

$ kubectl scale --replicas=2 deployment/$JOB-taskmanager

deployment.extensions/flink-on-kubernetes-taskmanager scaled

最后，使用 flink modify 调整脚本并发度：



1

2

3

$ bin/flink modify 755877434b676ce9dae5cfb533ed7f33 -m 192.168.99.108:30206 -p 2

Modify job 755877434b676ce9dae5cfb533ed7f33.

Rescaled job 755877434b676ce9dae5cfb533ed7f33. Its new parallelism is 2.

但是，因为存在一个尚未解决的 Issue，我们无法使用 flink modify 命令来对 HA 模式下的 Flink 集群进行扩容，因此还请使用人工的方式操作。



Flink 将原生支持 Kubernetes

Flink 有着非常活跃的开源社区，他们不断改进自身设计（FLIP-6），以适应现今的云原生环境。他们也注意到了 Kubernetes 的蓬勃发展，对 K8s 集群的原生支持也在开发中。我们知道，Flink 可以直接运行在 YARN 或 Mesos 资源管理框架上。以 YARN 为例，Flink 首先启动一个 ApplicationMaster，作为 JobManager，分析提交的脚本需要多少资源，并主动向 YARN ResourceManager 申请，开启对应的 TaskManager。当脚本的并行度改变后，Flink 会自动新增或释放 TaskManager 容器，达到扩容缩容的目的。这种主动管理资源的模式，社区正在开发针对 Kubernetes 的版本（FLINK-9953），今后我们便可以使用简单的命令来将 Flink 部署到 K8s 上了。



此外，另一种资源管理模式也在开发中，社区称为响应式容器管理（FLINK-10407 Reactive container mode）。简单来说，当 JobManager 发现手中有多余的 TaskManager 时，会自动将运行中的脚本扩容到相应的并发度。以上文中的操作为例，我们只需使用 kubectl scale 命令修改 TaskManager Deployment 的 replicas 个数，就能够达到扩容和缩容的目的，无需再执行 flink modify。相信不久的将来我们就可以享受到这些便利的功能。



参考资料

https://ci.apache.org/projects/flink/flink-docs-release-1.8/ops/deployment/kubernetes.html

https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/

https://jobs.zalando.com/tech/blog/running-apache-flink-on-kubernetes/

https://www.slideshare.net/tillrohrmann/redesigning-apache-flinks-distributed-architecture-flink-forward-2017

https://www.slideshare.net/tillrohrmann/future-of-apache-flink-deployments-containers-kubernetes-and-more-flink-forward-2019-sf

