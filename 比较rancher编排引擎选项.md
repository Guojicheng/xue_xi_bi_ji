# **比较Rancher编排引擎选项（**Kubernetes，Docker Native OrchestrationMarathon）

Rancher的最新版本增加了对几个常见编排引擎的支持。 除了Cattle之外新加支持三个Docker社区中使用最广泛的编排引擎，包括Swarm（Docker Native Orchestration）、Kubernetes和Mesos，它们将为用户提供了更多的功能及可用的编排选项。虽然Docker现在已经发展成为容器化领域的事实标准，但在细分的docker编排引擎市场中目前并没有明确的赢家。在本文中，我们将讨论三个编排引擎系统的功能和特性，并给出它们可能适用的场景用例的建议。

Docker Native Orchestration目前是相对来说项目较新，但它成长速度很快并不断的加入新功能。由于它是Docker官方本身推出的，因此有良好的工具和社区支持，是许多开发人员的默认选择。

Kubernetes是当今最广泛使用的容器编排系统之一，并且得到了Google的支持。

最后Mesos与Mesosphere（或Marathon，其开放源代码版本）支持分层、分类的服务管理方法，许多管理功能可基于独立的插件和应用程序。 因为架构的灵活性这使得它更容易深度定制化部署。当然，这也意味着需要更多的整合工作量。而Kubernetes的常见案例则更偏向于如何构建集群和交付完整统一的综合系统 。

## Docker Native Orchestration

![](/assets/dockernative.png)

### 基本结构

Docker Engine 1.12 集成了原生的编排引擎，用以替换了之前独立的Docker Swarm项目。Docker原生集群（Swarm）同时包括了（Docker Engine \/ Daemons），这使原生docker可以任意充当集群的管理\(manager\)或工（worker\)节点角色。工作节点将负责执行任务，运行你启动的容器，而管理节点从用户那里获得任务说明，负责整合现有的集群，维护集群状态。​并且您可以有多个管理节点以支持高可用，但推荐的数量不超过七个。管理节点会保障集群内部环境状态的强一致性，以及基于 [Raft](https://raft.github.io) 协议实现状态备份 ，与所有一制性算法一样，拥有更多manager节点意味着更多的性能开销，同时在manager节点内部保持一致性表明Docker原生编排没有外部依赖性，集群管理更容易。

### 可用性

Docker将使用单节点Docker的概念扩展到Swarm集群。如果你熟悉docker那你学习swarm相当容易。你想要使docker节点环境到迁移到运行swarm集群群也相当简单，（译者按：如果现有系统使用Docker Engine，则可以平滑将Docker Engine切到Swarm上，无需改动现有系统。）你只需要在其中的一个docker节点运行使用 docker swarm init命令创建一个集群，在您要添加任何其他节点，使用，通过docker swarm join命令加入到刚才创建的集群中。之后您可以象在一个独立的docker环境下一样使用同样的Docker Compose的模板及相同的docker命令行工具。

### 功能集

Docker Native Orchestration 使用与Docker Engine和Docker Compose来支持。您仍然可以使用象原先在单个节点一样的链接服务（link\)，创建卷\(volume\)和定义公开端口\(expose）功能。 除了这些，还有两个新的概念，服务（ services ）和网络（ networks ）。

docker服务是在您的节点上启动的一定数量的始终保持运行的一组容器，并且如果其中一个容器死了，它支持自动更换。有两种类型的服务，复制（replicated）或全局（global）。 复制（replicated）服务在集群中维护指定数量的容器（意味着这个服务可以随意 scale），全局（global）服务在每个群集节点上运行容器的一个实例（译者按：不多, 也不少）。 要创建复制（replicated）服务，请使用以下命令。

```
docker service create          \
   –name frontend              \
   –replicas 5                 \
   -network my-network         \
   -p 80:80/tcp nginx:latest.
```

您可以使用_docker network create–driver overlay NETWORK\_NAME 创建命名的overylay网络。_使用指定的overylay网络，您可以在你的容器内创建孤立（isolated）的，平面化的（flat），加密（encrypted）的跨节点的主机虚拟网络。

你可以使用constraints加labels 来做一些非常基本的容器调度。 使用constraints 参数您可以向服务添加关联到具有指定标签的节点上启动容器。

```
docker service create                        \
   –name frontend                            \
   –replicas 5                               \
   -network my-network                       \
   --constraint engine.labels.cloud==aws     \
   --constraint node.role==manager           \
   -p 80:80/tcp nginx:latest.
```

此外，您可以使用 reserve CPU 和reserve memory标记来定义服务中的每个容器使用的资源量，以便在群集上启动多个服务时，以最小化资源争用放置容器（译者按：容器会被部署到最适合它们的位置，最少的容器运行，最多的CPU或者内存可用，等等\)。

```
 --limit-cpu value Limit CPUs (default 0.000)             \ 
 --limit-memory value Limit Memory (default 0 B)          \
 --reserve-cpu value Reserve CPUs (default 0.000)         \
 --reserve-memory value Reserve Memory (default 0 B) 
```

您可以使用以下命令进行基本的滚动部署。这将更新服务的容器镜像，但这样做，每次2个容器，每两组之间有10s的间隔。但是不支持运行状况检查和自动回滚。

```
docker service update        \
   –name frontend            \
   –replicas 5               \
   -network my-network       \
   --update-delay 10s        \
   --update-parallelism 2    \
   -p 80:80/tcp nginx:other-version.

```

Docke 支持使用卷驱动\( volume drivers \)程序支持持久性外部卷，并且使用Native orchestration 为其service create命令扩展支持了使用mount选项。 将以下代码段添加到上面的命令将会将NFS挂载到您的容器中。请注意这需要在Docker外部的主机上已经设置好NFS，一些其他驱动程序添加了对Amazon EBS卷驱动程序或Google容器引擎卷驱动程序的支持能够在不需要主机支持的情况下工作。 此外，这个功能还没有很好的文档，可能需要一些测试并参考github issue以在docker项目得到运行。

```
 --mount type=volume,src=/path/on/host,volume-driver=local,\
    dst=/path/in/container,volume-opt=type=nfs,\
    volume-opt=device=192.168.1.1:/your/nfs/path
```

## Kubernetes

![](/assets/K8S-300x300.png)

### 基本结构

从概念上讲，Kubernetes类似于Swarm的唯一点就是它也使用一个RAFT的管理器Master节点来保证强一制性。同时为达成目的Kubernetes采用[ETCD](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=https://github.com/coreos/etcd&usg=ALkJrhh427xzIQKlO1SvzIFPjHbislIwOw)。此外它将采用一个外部的扩展的网络层，是像overlay ，weave网络等。使用这些外部工具，您可以启动Kubernetes主要的组件; 象API服务器（API Server），控制器管理器（Controller Manager）和调度程序（Scheduler），这些通常作为Kubernetes pod在主（master）节点上运行。除了这些，你还需要在每个节点\(node\)上运行kubelet和kubeproxy。工作节点（Worker nodes）只运行Kubelet和Kubeproxy以及一个网络层提供者象是flanneld，

![](/assets/clipboard.png)

在这个设置中，kubelet的主要功能就是获取节点上 pod\/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），kubelet将使用主（master\)节点上的controller manager操纵指定的节点上pod\/containers。调度器\(scheduler\)负责资源分配和平衡资源，将具有最多可用资源的工作节点\(worker node\)上放置容器。 API控制器负责您的本地kubectl CLI将向集群发出命令。 最后，kubeproxy组件用于为Kubernetes中定义的服务提供负载平衡和高可用性。

### 可用性

从头开始设置Kubernetes是一个困难的过程，因为它需要设置etcd，网络插件，DNS服务器和证书颁发机构。 从基础开始建立Kubernetes的详情，可浏览[这里](http://kubernetes.io/docs/getting-started-guides/)，但幸运的是Rancher已经做好这一切的设置。以前的文章中我们已经介绍了如何在Rancher设置[kubernetes](http://rancher.com/getting-micro-services-production-kubernetes/)。

除了初始设置，Kubernetes仍然有一些陡峭的学习曲线，因为它使用自己的术语和概念。Kubernetes使用资源类型，如Pods，Deployments,Replication Controllers，Services，Daemon sets等来定义部署。 这些概念不是Docker术语词典的一部分，因此您需要在开始创建第一个部署之前熟悉它们。 此外，一些概念与Docker冲突， 例如Kubernetes Services 概念上并不等同于Docker Services ，（Docker Services更贴近地映射到Kubernetes世界中的Deployments）。 此外，您使用kubectl而不是docker CLI与来用于群集交互，您必须使用Kubernetes配置文件，而不是docker compose文件。

Kubernetes有这样一套独立于核心Docker的概念本身并不是一件坏事。 Kubernetes提供比核心Docker更丰富的功能集。 然而，Docker将添加更多的功能来与Kubernetes竞争，它们将具有不同的实现方式或冲突的概念。这将肯定重现象CoreOS与rkt之间这种类似但竞争的解决方案情况， 今天，Docker Swarm和Kubernetes的目标是非常不同的用例（Kubernetes更适合于使用于有专门的集群管理团队的面向服务的架构的大型生产部署），然而随着Docker Native Orchestration的成熟，它也舞台同台竞争。

### 功能集

Kubernetes的完整功能集太大了，不能涵盖在本文中，但我们将讨论一些基本概念和一些有趣的区分。 首先，Kubernetes使用Pods的概念作为其缩放的基本单位，而不是单个容器。 每个pod是一组容器（设置可以是1），它们总是在同一节点上启动，共享相同的卷并分配一个虚拟IP（VIP），以便它们可以在集群中寻址。 单个pod的Kubernetes规范文件如下所示。

```
kind: Pod
metadata:
  name: mywebservice
spec:
  containers:
  - name: web-1-10
    image: nginx:1.10
    ports:
    - containerPort: 80
```

接下来展开 Deployment 设置，这些松散映射服务到Docker原生编排。您可以像Docker Native中的服务一样扩展部署运行请求数量的容器。需要的是注意，Deployment仅类似于Docker本地中的复制服务，如Kubernetes使用守护程序集概念（ Daemon Set） 来支持其等效的全局调度（globally scheduled）服务。部署还支持使用HTTP或TCP可达性或自定义exec命令进行状况检查以确定容器\/pod运行是否正常。 部署还支持使用运行状况检查的自动回滚滚动部署，以保障每个pod部署成功

```
kind: Deployment
metadata:
  name: mywebservice-deployment
spec:
  replicas: 2 # We want two pods for this deployment
  template:
    metadata:
      labels:
        app: mywebservice
    spec:
      containers:
      - name: web-1-10
        image: nginx:1.10
        ports:
        - containerPort: 80
```

接下来它为部署提供简单的负载平衡。 部署中的所有pod将在服务进入和退出时注册到服务，服务也抽象出多个Deployment，因此如果您想运行滚动部署，您将使用相同的服务注册两个Kubernetes Deployment，然后逐步将pod添加到一个同时从其他Deployment减少pod。 您甚至可以进行蓝绿部署，在那里您可以一次性将服务指向新的Kubernetes Deployment。 最后，服务对您的Kubernetes集群中的服务发现也很有用，集群中的所有服务都获得VIP，并且作为docker link 风格的环境变量很好的集成的DNS服务器暴露给集群中的所有pod。

除了基本的服务，Kubernetes支持[Jobs](http://kubernetes.io/docs/user-guide/jobs/), [Scheduled Jobs](http://kubernetes.io/docs/user-guide/scheduled-jobs/), and [Pet Sets](http://blog.kubernetes.io/2016/07/thousand-instances-of-cassandra-using-kubernetes-pet-set.html)，[Jobs](http://kubernetes.io/docs/user-guide/jobs/)创建一个或多个pod，并等待直到它们终止。作业确保你有指定数量的pod完成作业。例如，您可以开始一个作业（job\)，在最后一天开始处理1小时的商业智能数据。它将启动一个包含前一天的24个pod的作业，一旦它们都运行完成，作业才完成。scheduled job名称暗示了计划作业是在给定计划之上自动运行的作业。在我们的例子中，我们可能使我们的BI处理器是每日计划的工作。 作业非常适合向集群发出批量处理风格的工作负载，这些负载不是总是一直启动的服务，而是需要运行到完成然后清理的任务。

Kubernetes提供给基本服务的另一个扩展是Pet Sets，（编者按：它是 Kubernetes 1.3 引入的对象类型，目的是改善对有状态服务的支持）。Pet Sets支持通常非常难以容器化的有状态服务的工作负载。这包括数据库和有实时性连接需求的应用程序。Pet Sets为集合中的每个“Pet”提供稳定的主机名。Pet是可被索引; 例如，pet5将独立于pet3可寻址，并且如果pet3容器\/pod死掉，则它将使用相同索引信息（index\)和主机名\(hostname\)的新主机上重新启动。

Pet Sets还提供了稳定的存储[持久卷](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=http://kubernetes.io/docs/user-guide/persistent-volumes/&usg=ALkJrhiWPIrc802sxcTLVayuehySWBx7Uw)，也就是说，如果PET1死亡，它将重新启动在另一个节点并重新挂载原来数据卷。此外，您还可以使用NFS或其他网络文件系统在容器之间共享卷，即使它们在不同的主机上启动。这解决了从单主机到分布式Docker环境转换时最困难的问题之一。

Pet Sets还提供对等体发现（peer-discovery），通常的服务，你可以发现其他服务（通过Docker link等），然而，发现服务内的其他容器是不可能的。这使得基于gossip协议的服务，如Cassandra和Zookeeper非常难以启动。

最后，Pet Sets提供启动和排序，这是持久，可扩展的服务如Cassandra的必要条件。Cassandra依赖一组种子节点，当您扩展服务时，必须确保种子节点是第一个启动的节点和最后一个要删除的节点。在撰写本文时，Pet Sets是Kubernetes的一大特色，因为在没有这种支持的情况下，持久的有状态工作负载几乎不可能在Docker的生产规模上运行。

Kubernetes还在群集级别上提供了命名空间（[namespaces](http://kubernetes.io/docs/user-guide/namespaces/)），隔离工作负载[的](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=http://kubernetes.io/docs/user-guide/secrets/&usg=ALkJrhhmg53EBYgXyeW1ziO-e75eG407Gg)安全管理（[secrets management](http://kubernetes.io/docs/user-guide/secrets/)）和自动扩展（[auto-scaling](http://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/)）支持。 所有这些特性更意味着Kubernetes也支持大型，多样化的工作负载，Docker Swarm目前还没有这些特性。

## 马拉松（Marathon）

![](/assets/Marathon.png)

### 基本结构

大规模集群的另一个常见的编排设置选择是在Apache Mesos之上运行Marathon。Mesos是一个开源集群管理系统，支持各种各样主机上的工作负载。Mesos在群集中的每个主机上运行的Mesos代理，向master主机报告其可用资源。 Mesos 利用多台 Mesos master 来实现高可用性（high-availability），包括一个活跃的 master （译者按：叫做 leader 或者 leading master）和若干备份 master 来避免宕机。 通过 Apache ZooKeeper 选举出活跃的 leader，然后通知集群中的其他节点，包括其他Master，slave节点和调度器（scheduler driver）。在任何时间，master节点之一使用主选举过程是活动着的。master可以向任何Mesos agent发出任务，并报告这些任务的状态。虽然您可以通过API发出任务，但正常的方法是在Mesos之上使用一个framework框架。Marathon就是一个这样的framework框架，它为运行Docker容器（以及本地Mesos容器）提供支持。

### 可用性

与Swarm相比，Marathon有一个相当陡峭的学习曲线，因为它不与Docker分享大部分概念和术语。然而，马拉松（ Marathon ）功能并不丰富，因此比起Kubernetes更容易学习。管理Marathon部署的复杂性主要来自于它是架构在Mesos之上的事实，因此有两层工具要管理。此外， 马拉松 \(Marathon\)的一些更高级功能（如负载平衡）仅支持在Marathon之上运行的附加框架提供。 某些功能（如身份验证）只有在DC\/OS上运行Marathon时才可用，而后者又在Mesos上运行 - 这为堆栈添加另一层抽象。

### 功能集

要在Marathon中定义服务，您需要使用其内部JSON格式，如下所示。 一个简单的定义，如下面的一个将创建一个服务，运行nginx容器的两个实例。

```
{
  "id": "MyService"
  "instances": 2,
  "container": {
    "type": "DOCKER",
    "docker": {
      "network": "BRIDGE",
      "image": "nginx:latest"
    }
  }
}
```

一个略微更完整的版本如下所示，我们现在添加端口映射和健康检查。 在端口映射中，我们指定一个容器端口，这是docker容器公开的端口。 主机端口定义主机公共接口上的哪个端口映射到容器端口。 如果为主机端口指定0，则在运行时分配随机端口。 类似地，我们可以可选地指定服务端口。服务端口用于服务发现和负载平衡，如本节后面所述。 使用健康检查，我们现在既可以做滚动（默认）和蓝绿色的部署

```
{
  "id": "MyService"
  "instances": 2,
  "container": {
    "type": "DOCKER",
    "docker": {
      "network": "BRIDGE",
      "image": "nginx:latest"
      "portMappings": [
        { "containerPort": 8080, "hostPort": 0, "servicePort": 9000, "protocol": "tcp" },
      ]
    }
  },
  "healthChecks": [
    {
      "protocol": "HTTP",
      "portIndex": 0,
      "path": "/",
      "gracePeriodSeconds": 5,
      "intervalSeconds": 20,
      "maxConsecutiveFailures": 3
    }
  ]
}
```

在单一服务之外，您还可以定义 马拉松 \\(Marathon\\)应用程序组\\( Application Groups \\)用于嵌套树结构的服务。在组中定义应用程序的好处是能够将整个组缩放在一起。这是非常有用的在微服务栈场景中因为单独去调整每个服务是相对困难的。到目前为止，进一步假设所有服务将以相同的速率扩展，如果您需要一个服务的“n”个实例，您将获得所有服务的“n”个实例

```
{
  "id": "/product",
  "groups": [
    {
      "id": "/product/database",
      "apps": [
         { "id": "/product/mongo", ... },
         { "id": "/product/mysql", ... }
       ]
    },{
      "id": "/product/service",
      "dependencies": ["/product/database"],
      "apps": [
         { "id": "/product/rails-app", ... },
         { "id": "/product/play-app", ... }
      ]
    }
  ]
}
```

在定义基本的服务之外，马拉松 （Marathon ）还可以做基于指定容器的约束条件调度，详见[这里](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=https://mesosphere.github.io/marathon/docs/constraints.html&usg=ALkJrhiRCoWI_pzeza9W5vjrJCNkNFVKPw)，包括指定该服务的每个实例必须在不同的物理主机 _“constraints”: \[\[“hostname”, “UNIQUE”\]\]._您可以使用_的CPU_和_mem_标签指定容器的资源利用率。每个Mesos代理报告其总资源可用性，因此调度程序可以以智能方式在主机上放置工作负载。

默认情况下，Mesos依赖于传统的Docker端口映射和外部服务发现和负载均衡机制。 然而，最近的测试版功能添加了使用基于DNS服务发现支持[Mesos DNS](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=http://mesosphere.github.io/mesos-dns/&usg=ALkJrhjJryI9-VD4A5pRC4WHKK3bFPzU5A)或负载均衡使用[Marathon LB](https://github.com/mesosphere/marathon-lb)。 Mesos DNS是一个在Mesos之上运行的应用程序，它查询Mesos API以获取所有正在运行的任务和应用程序的列表。然后，它为运行这些任务的节点创建DNS记录。之后所有Mesos代理需要手动更新使用Mesos DNS服务作为其主DNS服务器。Mesos DNS使用主机名或IP地址用于Mesos agent向master主机注册，端口映射可以查询为SRV记录。Marathon DNS使用agent的主机名，并且必须确保主机网络相应端口打开且不能发生冲突。Mesos DNS确实提供了与众不同的方法来为状态负载持续引用，例如我们将能够使用Kubernetes pet sets。此外与Kubernetes有群集内任何容器可寻址的VIP机制不同，Mesos必须手动将\/etc\/resolve.conf更新到Mesos DNS服务器集，并在DNS服务器更改时更新配置。 Marathon-lb使用Marathon Event bus 跟踪所有服务的启动和撤销。然后，它在agent节点上启动HAProxy实例，以将流量中继到必需的服务节点。

马拉松（Marathon）的测试版支持[持久卷，](https://translate.googleusercontent.com/translate_c?depth=1&hl=en&rurl=translate.google.com.hk&sl=en&tl=zh-CN&u=https://mesosphere.github.io/marathon/docs/persistent-volumes.html&usg=ALkJrhhVPrswjgn73Z0dRgQI1pjxPoNTXQ)以外部持久性卷（[external persistent volumes](https://mesosphere.github.io/marathon/docs/external-volumes.html)） 。然而，这两个特征都处于非常原始的状态。持久卷只支持容器在单个节点上重新启动时支持持久化数据卷，但是会删除卷如果删除使用它们的应用，虽然磁盘上的实际数据不会被删除，volume必须手动删除。外部持久性卷的支持则限制在需要DC\/OS之上，并且当前只允许您的服务扩展到单个实例。

## 最终结论

现在我们看了Docker容器编排的三个选项：Docker Native（Swarm），Kubernetes和Mesos\/Marathon。很难选择一个系统来推荐，因为最好的系统高度依赖于你的具体用例需求，部署规模和应用历史。此外，所有三个系统现在还都在大量开发中，上面总结的一些功能是包括了测试版本，可能会很快更改，删除或被替换。

Docker Native给你提供了最快的升级，很少甚至没有对Docker的依赖之外的供应商锁定问题。对Docker的依赖不是一个大问题，因为它已经成为了事实上的容器标准。鉴于在编排战争中缺乏明确的赢家，而且Docker本机是最灵活的方法，因此它是简单的Web及无状态应用程序的不错选择，但是，Docker Native目前是非常简单，如果你需要得到复杂的，大规模的应用程序到生产，你需要选择一个Mesos\/Marathon或Kubernetes。

在Mesos\/Marathon和Kubernetes之间也不是一个容易的选择，因为两者都有自己的优点和缺点。在两者之中Kubernetes肯定是更丰富和成熟的，但它也是一个非常有固执的软件（译者按：Kubernetes有些固执己见对于容器如何组织和网络强制了一些概念）我们认为很多这些意见是有意义的。Kubernetes没有马拉松的灵活性，尤其当你考虑那些没有Docker，没有容器化的应用程序可以运行在Mesos（例如Hadoop集群）的丰富的历史。 如果你正在做一个绿色领域实施，也没有强烈的意见，如何布局集群，或你的意见偏向同意谷歌，那么Kubernetes是一个更好的选择。相反，如果你有大的，复杂的遗留工作负载，将逐渐转移到容器，那么Mesos\/Marathon是要走的路。

另一个问题是规模：Kubernetes已经测试了数千个节点，而Mesos已经测试了成千上万的节点。如果您正在启动具有数万个节点的集群，则需要使用Mesos来获得底层基础架构的可扩展性 - 但请注意，将高级功能（例如负载平衡）扩展到该范围仍将保留。然而，在那个规模，很少有现成的解决方案，如果有的话它也需要不断仔细调整及不断的改造。

Usman是一名服务器和基础设施工程师，具有在各种云平台之上构建大规模分布式服务的经验。你可以阅读更多他的工作
原文地址：http:\/\/rancher.com\/comparing-rancher-orchestration-engine-options\/

