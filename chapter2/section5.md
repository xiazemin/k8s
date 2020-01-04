# Sandeep Dinesh\(谷歌\)的5大Kubernetes最佳实践



[来自Sandeep Dinesh\(谷歌\)的5大Kubernetes最佳实践](https://blog.csdn.net/KINGSUO2016/article/details/86625394#Sandeep_Dinesh5Kubernetes_1)

* * [Kubernetes的最佳实践](https://blog.csdn.net/KINGSUO2016/article/details/86625394#Kubernetes_5)
  * [一、构建容器](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_15)
  * * [1.不要相信任意基础镜像](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1_16)
    * [2.保持基础镜像尽可能小](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2_24)
    * [3.使用构建器模式](https://blog.csdn.net/KINGSUO2016/article/details/86625394#3_37)
  * [二、容器内部](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_45)
  * * [1.容器内使用非root用户](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1root_46)
    * [2.文件系统只读](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2_58)
    * [3.一个容器一个进程](https://blog.csdn.net/KINGSUO2016/article/details/86625394#3_61)
    * [4.不要使用Restart on Failure， 而应当Crash Cleanly](https://blog.csdn.net/KINGSUO2016/article/details/86625394#4Restart_on_Failure_Crash_Cleanly_66)
    * [5.将所有日志打到标准输出和标准错误输出\(stdout&stderr\)](https://blog.csdn.net/KINGSUO2016/article/details/86625394#5stdoutstderr_69)
  * [三、部署](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_72)
  * * [1.使用“记录”选项以便轻松回滚\(record\)](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1record_73)
    * [2.多使用描述性的标签\(label\)](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2label_80)
    * [3.用sidecar来做代理、监视器等](https://blog.csdn.net/KINGSUO2016/article/details/86625394#3sidecar_83)
    * [4.不要使用sidecar来做启动引导](https://blog.csdn.net/KINGSUO2016/article/details/86625394#4sidecar_88)
    * [5.不要使用：latest或者无标签](https://blog.csdn.net/KINGSUO2016/article/details/86625394#5latest_95)
    * [6.善用readiness、liveness探针](https://blog.csdn.net/KINGSUO2016/article/details/86625394#6readinessliveness_98)
  * [四、服务](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_102)
  * * [1.不要使用type: Loadbalancer](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1type_Loadbalancer_103)
    * [2.Type: Nodeport可能已经够用了](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2Type_Nodeport_109)
    * [3.使用静态IP， 它们免费！](https://blog.csdn.net/KINGSUO2016/article/details/86625394#3IP__112)
    * [4.将外部服务映射到内部](https://blog.csdn.net/KINGSUO2016/article/details/86625394#4_115)
  * [五、应用架构](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_118)
  * * [1.使用Helm Charts](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1Helm_Charts_119)
    * [2.所有下游的依赖是不可靠的](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2_123)
    * [3.使用Weave Cloud](https://blog.csdn.net/KINGSUO2016/article/details/86625394#3Weave_Cloud_126)
    * [4.确保你的微服务不要太“微小”](https://blog.csdn.net/KINGSUO2016/article/details/86625394#4_129)
    * [5.使用命名空间来分离集群](https://blog.csdn.net/KINGSUO2016/article/details/86625394#5_132)
    * [6.基于角色的访问控制\(RBAC\)](https://blog.csdn.net/KINGSUO2016/article/details/86625394#6RBAC_135)
  * [从运行Weave Cloud生产环境学到的教训](https://blog.csdn.net/KINGSUO2016/article/details/86625394#Weave_Cloud_138)
  * [挑战1.基础设施的版本控制](https://blog.csdn.net/KINGSUO2016/article/details/86625394#1_147)
  * [问题：当Prod与版本控制不匹配时，您会怎么做？](https://blog.csdn.net/KINGSUO2016/article/details/86625394#Prod_158)
  * [挑战2.自动化持续交付](https://blog.csdn.net/KINGSUO2016/article/details/86625394#2_163)
  * [综上所述](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_172)



# 来自Sandeep Dinesh\(谷歌\)的5大Kubernetes最佳实践

原文链接：[https://www.weave.works/blog/kubernetes-best-practices](https://www.weave.works/blog/kubernetes-best-practices)  
在最近的[Weave Online User Group \(WOUG\)](https://www.meetup.com/pro/weave/)上，两位发言者介绍了Kubernetes的主题。Sandeep Dinesh[（@SandeepDinesh）](https://twitter.com/SandeepDinesh)，Google Cloud的开发者倡导者提供了在Kubernetes上运行应用程序的最佳实践列表。Weaveworks工程师Jordan Pellizzari[（@jpellizzari）](https://twitter.com/jpellizzari)接着就Kubernetes开发和运行SaaS Weave Cloud两年后的经验教训进行了跟进。

## Kubernetes的最佳实践

此演示文稿中的最佳实践源于Sandeep及其团队关于您可以在Kubernetes中执行相同任务的许多不同方式的讨论。他们编制了一份这些任务的清单，并从中衍生出一套最佳实践。

最佳实践分为：

1. [构建容器](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_15)
2. [容器内部](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_45)
3. [部署](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_72)
4. [服务](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_102)
5. [应用架构](https://blog.csdn.net/KINGSUO2016/article/details/86625394#_118)

## 一、构建容器

### 1.不要相信任意基础镜像

不幸的是，我们看到这种情况一直在发生，Pradeep说。人们将从DockerHub中获取某人创建的基本图像 - 因为乍一看它有他们需要的包 - 但随后将任意选择的容器推向生产。

这有很多错误：您可能使用了具有漏洞的错误代码版本，其中存在错误，或者更糟糕的是它可能会故意捆绑恶意软件 - 您只是不知道。

为了缓解这种情况，您可以运行CoreOS’Clair或Banyon Collector之类的静态分析，您可以使用它来扫描容器中的漏洞。

### 2.保持基础镜像尽可能小

从最精简最可行的基本图像开始，然后在顶部构建您的包，以便您知道内部的内容。

较小的基础图像也减少了开销。您的应用程序可能只有大约5 MB，但如果您盲目地使用现成的图像，例如Node.js，它包含一个额外的600MB库，您不需要。

较小图像的其他优点：

1. 更快的构建
2. 存储量减少
3. 图像拉动速度更快
4. 可能减少攻击面
 
   ![](https://img-blog.csdnimg.cn/20190124135819174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

### 3.使用构建器模式

此模式对于编译为Go和C ++或Typescript for Node.js的静态语言更有用。

在这种模式中，您将拥有一个包含编译器，依赖项和单元测试的构建容器。然后代码运行第一步并输出构建工件。它们与任何静态文件，包等组合在一起，并通过运行时容器，该容器也可能包含一些监视或调试工具。

最后，您的Docker文件应该只引用您的基本映像和运行时环境容器。  
![](https://img-blog.csdnimg.cn/20190124140016669.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

## 二、容器内部

### 1.容器内使用非root用户

当以root身份在容器内更新包时，您需要将用户更改为非root用户。

原因是，如果有人攻击您的容器并且您没有从root更改用户，那么简单的容器转义可以让他们访问您将成为root用户的主机。当您将用户更改为非root用户时，黑客需要额外的黑客尝试才能获得root访问权限。

作为最佳实践，您希望尽可能多地在您的基础架构周围安装shell。

在Kubernetes中，您可以通过设置[Security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)**runAsNonRoot：true**来强制执行此操作，这将使其成为整个群集的策略范围设置。

### 2.文件系统只读

这是另一个可以通过设置选项**readOnlyFileSystem：true**来强制执行的最佳实践。

### 3.一个容器一个进程

您可以在容器中运行多个进程，但建议只运行一个进程。这是因为orchestrator的工作方式。Kubernetes根据流程是否健康来管理容器。如果你在容器内运行了20个进程 - 它将如何知道它是否健康？

要运行所有谈话并相互依赖的多个进程，您需要在Pod中运行它们。

### 4.不要使用Restart on Failure， 而应当Crash Cleanly

Kubernetes为您重新启动失败的容器，因此您应该使用错误代码彻底崩溃，以便他们可以在没有您干预的情况下成功重新启动。

### 5.将所有日志打到标准输出和标准错误输出\(stdout&stderr\)

默认情况下，Kubernetes会监听这些管道并将输出发送到您的日志记录服务。例如，在Google Cloud上，他们会自动转到StackDriver。

## 三、部署

### 1.使用“记录”选项以便轻松回滚\(record\)

应用yaml时，请使用–record标志：

`kubectl apply -f deployment.yaml --record`

使用此选项，每次有更新时，它都会保存到这些部署的历史记录中，并使您能够回滚更改。  
![](https://img-blog.csdnimg.cn/20190124140941108.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

### 2.多使用描述性的标签\(label\)

由于标签是任意键值对，因此它们非常强大。例如，考虑下面的图表，名为’Nifty’的应用程序分布在四个容器中。使用标签，您可以通过选择后端（BE）标签仅选择后端容器。  
![](https://img-blog.csdnimg.cn/20190124141142472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

### 3.用sidecar来做代理、监视器等

有时您需要一组进程来相互通信。但是你不希望所有这些都在一个容器中运行（参见上面“每个容器一个进程”），而是在Pod中运行相关进程。

同样，当您运行代理或您的流程所依赖的观察程序时。例如，您的进程所依赖的数据库。您不会将凭据硬编码到每个容器中。相反，您可以将凭证作为代理部署到安全处理连接的sidecar中：  
![](https://img-blog.csdnimg.cn/20190124141327830.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

### 4.不要使用sidecar来做启动引导

尽管sidecar在处理集群内外的请求时非常有用，Sandeep不推荐使用它做启动。再过去，引导启动（bootstraping）是唯一选项，但是现在Kubernetes有了“init 容器”。

当容器里面的一个进程依赖于其它的一个微服务时， 你可以使用init容器来等到进程启动以后再启动你的容器。这可以避免当进程和微服务不同步时产生的很多错误。

基本原则就是： 使用sidecar来处理总是发生的事件，而用init容器来处理一次性的事件。

### 5.不要使用：latest或者无标签

这个原则是很明显的而且大家基本都这么在用。如果你不给你的容器加标签，那么它会总是拉最新的，这个“最新的”并不能保证包括你认为它应该有的那些更新。

### 6.善用readiness、liveness探针

使用探针可以让Kubernetes知道节点是否正常，以此决定是否把流量发给它。缺省情况下Kubernetes检查进程是否在运行。但是通过使用探针， 你可以在缺省行为下加上你自己的逻辑。  
![](https://img-blog.csdnimg.cn/20190124141539860.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

## 四、服务

### 1.不要使用type: Loadbalancer

每次你在部署文件里面加一个公有云提供商的loadbalancer（负载均衡器）的时候，它都会创建一个。 它确实是高可用，高速度，但是它也有经济成本。  
使用Ingress来代替，同样可以实现通过一个end point来负载均衡多个服务。这种方式不但更简单，而且更经济。  
![](https://img-blog.csdnimg.cn/20190124141655612.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")  
当然这个策略只有你提供http和web服务时有用，对于普通的TCP/UDP应用就没用了。

### 2.Type: Nodeport可能已经够用了

这个更多是个人喜好，并不是所有人都推荐。NodePort把你的应用通过一个VM的特定端口暴露到外网上。 问题就是它没有像负载均衡器那样有高可用。比如极端情况，VM挂了你的服务也挂了。

### 3.使用静态IP， 它们免费！

在谷歌云上很简单，只需要为你的ingress来创建全局IP。类似的对你的负载均衡器可以使用Regional IP。这样当你的服务down了之后你不必担心IP会变。

### 4.将外部服务映射到内部

Kubernetes提供的这个功能不是所有人都知道。如果您需要群集外部的服务，您可以做的是使用**ExternalName**类型的服务。这样你就可以通过名字来调用这个服务，Kubernetes manager会把请求传递给它，就好像它在集群之中一样。Kubernetes对待这个服务就好像它在同一个内网里面，即使实际上它不在。

## 五、应用架构

### 1.使用Helm Charts

Helm基本上就是打包Kubernetes应用配置的仓库。如果你要部署一个MongoDB， 存在一个预先配置好的Helm chart，包括了它所有的依赖，你可以十分容易的把它部署到集群中。  
很多流行的软件/组件都有写好了的Helm charts， 你可以直接用，省掉大量的时间和精力。

### 2.所有下游的依赖是不可靠的

你的应用应该有逻辑和错误信息负责审计你不能控制的所有依赖。Sandeep建议说你可以使用[Istio](https://istio.io/)或者[Linkerd](https://linkerd.io/)这样的服务网格来做下游管理。

### 3.使用Weave Cloud

集群是很难可视化管理的。 使用[Weave Cloud](https://www.weave.works/features/troubleshooting-dashboard/)可以帮你监视集群内的情况和跟踪依赖。

### 4.确保你的微服务不要太“微小”

你需要的是逻辑组件，而不是每个单独的功能/函数都变成一个微服务。

### 5.使用命名空间来分离集群

例如， 你可以在同一个集群里面创建prod、dev、test这样不同的命名空间，同时可以对不同的命名空间分配资源， 这样万一某个进程有问题也不会用尽所有的集群资源。

### 6.基于角色的访问控制\(RBAC\)

实施时当的访问控制来限制访问量， 这也是最佳的安全实践。

## 从运行Weave Cloud生产环境学到的教训

接下来，Jordan Pellizzari讲述了我们在过去两年中在Kubernetes上运行和开发Weave Cloud所学到的知识。

我们目前在AWS EC2上运行，在13个主机和大约150个容器上运行大约72个Kubernetes Deployments。

我们所有的持久存储都保存在S3，DynamoDB或RDS中，我们不会将状态保存在容器中。

有关我们如何设置基础架构的更多详细信息，请参阅[Weaveworks & AWS: How we manage Kubernetes clusters](https://www.weave.works/technologies/weaveworks-on-aws/)。

## 挑战1.基础设施的版本控制

在Weaveworks，我们所有的基础设施都保存在Git中，当我们进行基础设施更改（如代码）时，它也可以通过拉取请求完成。我们一直在调用这个GitOps，我们有很多关于它的博客。您可以从第一个开始：[GitOps - Operations by Pull Request](https://www.weave.works/blog/gitops-operations-by-pull-request)。

在Weave，Terraform脚本，Ansible以及当然Kubernetes YAML文件都在Git下进行版本控制。

将您的基础设施保留在Git中的最佳做法有很多原因：

* 发布很容易回滚
* 关于谁做了什么创造的可审计的踪迹
* 灾难恢复要简单得多

## 问题：当Prod与版本控制不匹配时，您会怎么做？

除了将所有内容保存在Git中之外，我们还运行一个过程来检查prod群集中运行的内容与检入版本控制中的内容之间的差异。当它检测到差异时，会向我们的松弛通道发送警报。

我们使用名为[Kube-Diff](https://github.com/weaveworks/kubediff)的开源工具检查差异。

## 挑战2.自动化持续交付

自动化CI / CD管道并避免手动部署Kubernetes。由于我们每天要多次部署，因此这种方法可以节省团队宝贵的时间，因为它可以消除手动容易出错的步骤。在Weaveworks，开发人员只需执行Git推送，Weave Cloud负责其余工作：

* 标记代码通过CircleCI测试运行并构建新的容器映像并将新映像推送到注册表。
* Weave Cloud’Deploy Automator’通知图像，从存储库中提取新图像，然后在配置仓库中更新其YAML。
* 部署同步器检测到群集已过期，它从配置存储库中提取已更改的清单，并将新映像部署到群集。
* 
这是一篇较长的文章[\(The GitOps Pipeline\)](https://www.weave.works/blog/the-gitops-pipeline)，我们认为这是构建自动化CICD管道时的最佳实践。  
![](https://img-blog.csdnimg.cn/20190124142625532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0tJTkdTVU8yMDE2,size_16,color_FFFFFF,t_70 "在这里插入图片描述")

## 综上所述

Sandeep Dinesh深入介绍了在Kubernetes上创建，部署和运行应用程序的5个最佳实践。接下来是Jordan Pellizzari关于Weave如何在Kubernetes中管理其SaaS产品Weave Cloud以及所吸取教训的演讲。

