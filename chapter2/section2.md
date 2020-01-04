### prometheus

### prometheus

它本身就是一个监控系统，它类似于Zabbix，它也需要在Node上安装Agent,而prometheus将自己的Agent称为node\_exporter, 但这个node\_exporter它仅是用于给Prometheus提供Node的系统级监控指标数据的，因此，若你想采集MySQL的监控数据，你还需要自己部署一个MySQL\_exporter，才能采集mySQL的监控数据，而且Prometheus还有很多其它重量级的应用exporter，可在用到时自行学习。

我们需要知道，若你能获取一个node上的监控指标数据，那么去获取该node上运行的Pod的指标数据就非常容易了。因此Prometheus，就是通过metrics URL来获取node上的监控指标数据的，当然我们还可以通过在Pod上定义一些监控指标数据，然后,定义annotations中定义允许 Prometheus来抓取监控指标数据，它就可以直接获取Pod上的监控指标数据了。

**PromQL:**  
　　这是Prometheus提供的一个RESTful风格的，强大的查询接口，这也是它对外提供的访问自己采集数据的接口。  
　　但是Prometheus采集的数据接口与k8s API Server资源指标数据格式不兼容，因此API Server是不能直接使用Prometheus采集的数据的，需要借助一个第三方开发的k8s-prometheus-adapter来解析prometheus采集到的数据, 这个第三方插件就是通过PromQL接口，获取Prometheus采集的监控数据，然后，将其转化为API Server能识别的监控指标数据格式，但是我们要想通过kubectl来查看这些转化后的Prometheus监控数据，还需要将k8s-prometheus-adpater聚合到API Server中，才能实现直接通过kubectl获取数据 。

\#接下来部署Prometheus的步骤大致为：  
　　1. 部署Prometheus  
　　2. 配置Prometheus能够获取Pod的监控指标数据  
　　3. 在K8s上部署一个k8s-prometheus-adpater Pod  
　　4. 此Pod部署成功后，还需要将其聚合到APIServer中

![](https://img2018.cnblogs.com/blog/922925/201908/922925-20190802202838002-1426529093.png)

说明：  
　　Prometheus它本身就是一个时序数据库，因为它内建了一个存储 所有eporter 或 主动上报监控指标数据给Prometheus的Push Gateway的数据 存储到自己的内建时序数据库中，因此它不需要想InfluxDB这种外部数据库来存数据。  
　　Prometheus在K8s中通过Service Discovery来找到需要监控的目标主机，然后通过想Grafana来展示自己收集到的所有监控指标数据，另外它还可以通过Web UI 或 APIClients\(PromQL\)来获取其中的数据。  
　　Prometheus自身没有提供报警功能，它会将自己的报警需求专给另一个组件Alertmanger来实现报警功能。

\#在K8s上部署Prometheus需要注意，因为Prometheus本身是一个有状态数据集，因此建议使用statefulSet来部署并控制它，但是若你只打算部署一个副本，那么使用deployment和statefulSet就不重要了。但是你若需要后期进行纵向或横向扩展它，那你就只能使用StatefulSet来部署了。

部署K8s Prometheus  
需要注意，这是马哥自己做的简版Prometheus，他没有使用PVC，若需要部署使用PVC的Prometheus，可使用kubernetes官方的Addons中的清单来创建。  
官方地址：[https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus)  
马哥版的地址：[https://github.com/iKubernetes/k8s-prom](https://github.com/iKubernetes/k8s-prom)

下面以马哥版本来做说明：

1. 先部署名称空间：

  kubectl  apply  -f   namespace.yaml

    ---

    apiVersion: v1

    kind: Namespace

    metadata:

      name: prom       \#这里需要注意： 他是先创建了一个prom的名称空间，然后，将所有Prometheus的应用都放到这个名称空间了。

              

2. 先创建node\_exporter:

cd  node\_exporter

  \#需要注意：

   apiVersion: apps/v1

    kind: DaemonSet

    metadata:

      name: prometheus-node-exporter

      namespace: prom

   .......

       spec:

          tolerations:   \#这里需要注意：你要保障此Pod运行起来后，它能容忍Master上的污点.这里仅容忍了默认Master的污点.这里需要根据实际情况做确认。

          - effect: NoSchedule

            key: node-role.kubernetes.io/master 

         containers:

              - image: prom/node-exporter:v0.15.2  \#这里使用的node-exporter的镜像版本。

                name: prometheus-node-exporter



\#应用这些清单

 kubectl  apply  -f   ./

复制代码

  \#应用完成后，查看Pod



　　



复制代码

\#接着进入Prometheus的清单目录

cd  prometheus



\#prometheus-deploy.yaml 它需要的镜像文件可从hub.docker.com中下载，若网速慢的话。



\#prometheus-rbac.yaml

    apiVersion: rbac.authorization.k8s.io/v1beta1

    kind: ClusterRole

    metadata:

      name: prometheus

    rules:

    - apiGroups: \[""\]

      resources:

      - nodes

      - nodes/proxy

      - services

      - endpoints

      - pods

      verbs: \["get", "list", "watch"\]

    - apiGroups:

      - extensions

      resources:

      - ingresses

      verbs: \["get", "list", "watch"\]

    - nonResourceURLs: \["/metrics"\]

      verbs: \["get"\]





\#prometheus-deploy.yaml

    resources:         

    \#这里做了一个资源使用限制，需要确认你每个节点上要能满足2G的可用内存的需求，若你的Node上不能满足这个limits，就将这部分删除，然后做测试。   

      limits:

        memory: 2Gi



\#接着开始应用这些清单：

   kubectl  apply  -f   ./

复制代码

   \#应用完成后查看：



　　kubectl  get  all  -n  prom



复制代码

\#现在来部署，让K8s能获取Prometheus的监控数据

 cd  kube-state-metrics

 \#此清单中的镜像若不能从google仓库中获取，可到hub.docker.com中搜索镜像名，下载其他人做的测试

 

 \#K8s需要通过kube-state-metrics这个组件来进行格式转化，实现将Prometheus的监控数据转换为K8s API Server能识别的格式。

 \#但是kube-state-metrics转化后，还是不能直接被K8s所使用，它还需要借助k8s-prometheus-adpater来将kube-state-metrics聚合到k8s的API Server里面，这样才能通过K8s API Server来访问这些资源数据。

 

\#应用kube-state-metrics的清单文件

 　　kubectl   apply  -f  ./

复制代码

\#应用完成后，再次验证

　　



 



 



复制代码

\#以上创建好以后，可先到k8s-prometheus-adapter的开发者github上下载最新的k8s-prometheus-adapter的清单文件

　　https://github.com/DirectXMan12/k8s-prometheus-adapter/tree/master/deploy/manifests



\#注意：

\# ！！！！！！！！！

\#   使用上面新版本的k8s-prometheus-adapter的话，并且是和马哥版的metrics-server结合使用，需要修改清单文件中的名称空间为prom

\# !!!!!!!!!!!!!!!

\#     但下面这个文件要特别注意：

    custom-metrics-apiserver-auth-reader-role-binding.yaml

        piVersion: rbac.authorization.k8s.io/v1

        kind: RoleBinding

        metadata:

          name: custom-metrics-auth-reader

          namespace: kube-system     \#这个custom-metrics-auth-reader必须创建在kube-system名称空间中，因为它要绑到这个名称空间中的Role上

        roleRef:

          apiGroup: rbac.authorization.k8s.io

          kind: Role                 \#此角色是用于外部APIServer认证读的角色

          name: extension-apiserver-authentication-reader

        subjects:

        - kind: ServiceAccount

          name: custom-metrics-apiserver  \#这是我们自己创建的SA账号

          namespace: prom     \#这些需要注意：要修改为prom



\#以上清单文件下载完成后，需要先修改这些清单文件中的namespace为prom，因为我们要部署的Prometheus都在prom这个名称空间中.

之后就可以正常直接应用了

　　kubectl  apply  -f  ./

复制代码

   \# 应用完成后，需要检查

　　kubectl get all -n prom \#查看所有Pod都已经正常运行后。。



  \# 查看api-versions中是否已经包含了 custom.metrics.k8s.io/v1beta1, 若包含，则说明部署成功

　　kubectl api-versions



  \# 测试获取custom.metrics.k8s.io/v1beta1的监控数据

　　curl   http://localhost:8080/custom.metrics.k8s.io/v1beta1/



复制代码

下面测试将Grafana部署起来，并且让Grafana从Prometheus中获取数据

    

\#部署Grafana,这里部署方法和上面部署HeapSter一样，只是这里仅部署Grafana

wget  https://raw.githubusercontent.com/kubernetes-retired/heapster/master/deploy/kube-config/influxdb/grafana.yaml



\#此清单若需要修改apiVersion，也要向上面修改一样，配置其seletor。

 

\#另外还有需要注意:

   volumeMounts:

    - mountPath: /etc/ssl/certs  \#它会自动挂载Node上的/etc/ssl/certs目录到容器中，并自动生成证书，因为它使用的HTTPS.

      name: ca-certificates

      readOnly: true

              

\#Grafana启动时，会去连接数据源，默认是InfluxDB，这个配置是通过环境变量来传入的

   env:

    \#- name: INFLUXDB\_HOST

    \# value: monitoring-influxdb              

        \#这里可看到,它将InfluxDB的Service名传递给Grafana了。

        \#需要特别注意：因为这里要将Grafana的数据源指定为Prometheus，所以这里需要将InfluxDB做为数据源给关闭，若你知道如何定义prometheus的配置，

        \#也可直接修改，不修改也可以，那就直接注释掉，然后部署完成后，登录Grafana后，在修改它的数据源获取地址。

    - name: GF\_SERVER\_HTTP\_PORT

      value: "3000" 

      \#默认Grafara在Pod内启动时,监听的端口也是通过环境变量传入的,默认是3000端口.

    

 \#在一个是，我们需要配置Grafana可以被外部直接访问

  ports:

      - port: 80

        targetPort: 3000

      selector:

        k8s-app: grafana

      type:  NodePort

                



\#配置完成后，进行apply

  kubectl  apply  -f   grafana.yaml

  

\#然后查看service对集群外暴露的访问端口

  kubectl   get   svc   -n  prom

复制代码

  \#随后打开浏览器，做以下修改



　　



  \#接着，你可以查找一个，如何导入第三方做好的模板，然后，从grafana官网下载一个模板，导入就可以获取一个漂亮的监控界面了。

  \#获取Prometheus的模板文件，可从这个网站获取

　　https://grafana.com/grafana/dashboards?search=kubernetes



　　







 



 



HPA功能：

　　正如前面所说，它可根据我们所设定的规则，监控当前Pod整体使用率是否超过我们设置的规则，若超过则设置的根据比例动态增加Pod数量。

　　举个简单的例子：

　　假如有3个Pod，我们规定其最大使用率不能高于60%，但现在三个Pod每个CPU使用率都到达90%了，那该增加几个Pod的？

　　HPA的计算方式是：

　　　　90% × 3 = 270% , 那在除以60，就是需要增加的Pod数量， 270 / 60 = 4.5 ，也就是5个Pod



\#HPA示例：

　　kubectl run myapp --image=harbor.zcf.com/k8s/myapp:v1 --replicas=1 \

　　　　--requests='cpu=50m,memory=256Mi' --limits='cpu=50m,memory=256Mi' \

　　　　--labels='app=myapp' --expose --port=50



\#修改myapp的svcPort类型为NodePort，让K8s集群外部可以访问myapp，这样方便压力测试，让Pod的CPU使用率上升，然后，查看HPA自动创建Pod.

　　kubectl patch svc myapp -p '{"spec":{"type":"NodePort"}}'



　　\# kubectl get pods



　　



　　\#这里目前只有一个Pod！！



　　kubectl describe pod myapp-xxxx 　　 \#可查看到它当前的QoS类别为： Guranteed



\#创建HPA，根据CPU利用率来自动伸缩Pod

　　kubectl autoscale deployment myapp --min=1 --max=8 --cpu-percent=40



　　



复制代码

\#查看当前Pod是Service：

\# kubectl get svc

    NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT\(S\)        AGE

    ..................

    myapp        NodePort    172.30.162.48   &lt;none&gt;        80:50113/TCP   103m



\#创建成功后，就可以通过ab等压测工具来测试自动伸缩

\#apt-get install  apache2-utils

\#

\# ab  -c 1000  -n 3000   http://192.168.111.84:50113



\#查看HPA自动伸缩情况

   \# kubectl describe hpa

    Name:                                                  myapp

    Namespace:                                             default

   ............................

      resource cpu on pods  \(as a percentage of request\):  76% \(38m\) / 40%

    Min replicas:                                                                        1

    Max replicas:                                                                        8

    Deployment pods:                                       6 current / 8 desired   \#这里可看到现在已经启动6个Pod



        

   

\#创建一个HPA v2版本的自动伸缩其

\#完整配置清单：

vim  hpa-pod-demo-v2.yaml 

    apiVersion: v1

    kind: Service

    metadata:

      labels:

        app: myapp

      name: myapp-v2

    spec:

      clusterIP: 172.30.10.98

      ports:

      - port: 80

        protocol: TCP

        targetPort: 80

      type: NodePort

      selector:

        app: myapp

    ---

    apiVersion: apps/v1

    kind: Deployment

    metadata:

      labels:

        app: myapp

      name: myapp-v2

    spec:

      replicas: 1

      selector:

        matchLabels:

          app: myapp

      strategy: {}

      template:

        metadata:

          labels:

            app: myapp

        spec:

          containers:

          - image: harbor.zcf.com/k8s/myapp:v1

            name: myapp-v2

            ports:

            - containerPort: 80

            resources:

              limits:

                cpu: 50m

                memory: 256Mi

              requests:

                cpu: 50m

                memory: 256Mi



    ---

    apiVersion: autoscaling/v2beta1

    kind: HorizontalPodAutoscaler

    metadata:

      name: myapp-v2

    spec:

      maxReplicas: 8

      minReplicas: 1

      scaleTargetRef:

        apiVersion: extensions/v1beta1

        kind: Deployment

        name: myapp

      metrics:

       - type: Resource

         resource:

           name: cpu

           targetAverageUtilization: 55

       - type: Resource

         resource:

           name: memory

           targetAverageValue: 50Mi

复制代码

  \#压测方便和上面一样。这个配置清单中定义了CPU和内存的资源监控指标，V2是支持内存监控指标的，但V1是不支持的。



　　



  \#若以后自己程序员开发的Pod，能通过Prometheus导出Pod的资源指标，比如：HTTP的访问量，连接数，我们就可以根据HTTP的访问量或者连接数来做自动伸缩。

　在那个Pod上的那些指标可用，是取决于你的Prometheus能够从你的Pod的应用程序中获取到什么样的指标的，但是Prometheus能获取的指标是由一定语法要求的，开发要依据  Prometheus支持的RESTful风格的接口，去输出一些指标数据，这指标记录当前系统上Web应用程序所承载的最大访问数等一些指标数据，那我们就可基于这些输出的指标数据，来完成HPA自动伸缩的扩展。



\#自定义资源指标来创建HPA，实现根据Pod中输出的最大连接数来自动扩缩容Pod

\#下面是一个HPA的定义，你还需要创建一个能输出http\_requests这个自定义资源指标的Pod，然后才能使用下面的HPA的清单。



下面清单是使用自定义资源监控指标 http\_requests 来实现自动扩缩容:

　　docker pull ikubernetes/metrics-app　　 \#可从这里获取metrics-app镜像



复制代码

vim  hpa-http-requests.yaml

apiVersion: v1

kind: Service

metadata:

  labels:

    app: myapp

  name: myapp-hpa-http-requests

spec:

  clusterIP: 172.30.10.99    \#要根据实际情况修改为其集群IP

  ports:

  - port: 80

    protocol: TCP

    targetPort: 80

  type: NodePort             \#若需要集群外访问，可添加

  selector:

    app: myapp

---

apiVersion: apps/v1

kind: Deployment

metadata:

  labels:

    app: myapp

  name: myapp-hpa-http-requests

spec:

  replicas: 1      \#这里指定Pod副本数量为1

  selector:

    matchLabels:

      app: myapp

  strategy: {}

  template:

    metadata:

      labels:

        app: myapp

      annotations:         \#annotations一定要，并且要定义在容器中！！

        prometheus.io/scrape: "true"     \#这是允许Prometheus到容器中抓取监控指标数据

        prometheus.io/port: "80"                 

        prometheus.io/path: "/metrics"   \#这是指定从那个URL路径中获取监控指标数据

    spec:

      containers:

      - image: harbor.zcf.com/k8s/metrics-app  \#此镜像中包含了做好的，能输出符合Prometheus监控指标格式的数据定义。

        name: myapp-metrics

        ports:

        - containerPort: 80

        resources:

          limits:

            cpu: 50m

            memory: 256Mi

          requests:

            cpu: 50m

            memory: 256Mi



---

apiVersion: autoscaling/v2beta1

kind: HorizontalPodAutoscaler

metadata:

  name: myapp-hpa-http-requests

spec:

  maxReplicas: 8

  minReplicas: 1

  scaleTargetRef:       \#这指定要伸缩那些类型的Pod。

    apiVersion: extensions/v1beta1

    kind: Deployment    \#这里指定对名为 myapp-hpa-http-requests这个 Deployment控制器 下的所有Pod做自动伸缩.

    name: myapp-hpa-http-requests

  metrics:

   - type: Pods    \#设置监控指标是从那种类型的资源上获取：它支持Resource，Object，Pods ；

                   \#resource:核心指标如:cpu,内存可指定此类型，若监控的资源指标是从Pod中获取，那类型就是Pods

     pods:

       metricName: http\_requests   \#http\_requests就是自定义的监控指标,它是Prometheus中Pod中获取的。

       targetAverageValue: 800m    \#800m：是800个并发请求，因为一个并发请求，就需要一个CPU核心来处理，所以是800个豪核，就是800个并发请求。

     

     

\#  curl http://192.168.111.84:55066/metrics

   \#注意：Prometheus抓取数据时，它要求获取资源指标的数据格式如下：

    \# HELP   http\_requests\_total The amount of requests in total   \#HELP:告诉Prometheus这个数据的描述信息

    \# TYPE   http\_requests\_total   counter     \#TYPE: 告诉Prometheus这个数据的类型

    http\_requests\_total   1078                 \#告诉Prometheus这个数据的值是多少。



    \# HELP http\_requests\_per\_second The amount of requests per second the latest ten seconds

    \# TYPE http\_requests\_per\_second gauge

    http\_requests\_per\_second   0.1



                      

\# 测试方法：

1. 先在一个终端上执行：

   for  i  in  \`seq  10000\`;  do   curl  http://K8S\_CLUSTER\_NODE\_IP:SERVER\_NODE\_PORT/ ;  done



2. 查看hpa的状态

  \# kubectl   describe   hpa 

    Name:                       myapp-hpa-http-requests

    Namespace:                  default

    .........

    Reference:                  Deployment/myapp-hpa-http-requests

    Metrics:                    \( current / target \)

      "http\_requests" on pods:  4366m / 800m

    Min replicas:                                1

    Max replicas:                               8

    Deployment pods:            8 current / 8 desired

    ........................

    Events:

      Type     Reason                        Age   From                       Message

      ----     ------                        ----  ----                       -------

      .....................

      Normal   SuccessfulRescale             51s   horizontal-pod-autoscaler  New size: 4; reason: pods metric http\_requests above target

      Normal   SuccessfulRescale             36s   horizontal-pod-autoscaler  New size: 8; reason: pods metric http\_requests above target



3. 查看Pod

   \# kubectl   get   pod

    NAME                                       READY   STATUS    RESTARTS   AGE

    myapp-hpa-http-requests-69c9968cdf-844lb   1/1     Running   0          24s

    myapp-hpa-http-requests-69c9968cdf-8hcjl   1/1     Running   0          24s

    myapp-hpa-http-requests-69c9968cdf-8lx9t   1/1     Running   0          39s

    myapp-hpa-http-requests-69c9968cdf-d4xdr   1/1     Running   0          24s

    myapp-hpa-http-requests-69c9968cdf-k4v6h   1/1     Running   0          114s

    myapp-hpa-http-requests-69c9968cdf-px2rl   1/1     Running   0          39s

    myapp-hpa-http-requests-69c9968cdf-t52xr   1/1     Running   0          39s

    myapp-hpa-http-requests-69c9968cdf-whjl6   1/1     Running   0          24s

复制代码

