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

