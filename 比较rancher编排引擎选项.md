# **比较Rancher编排引擎选项（**Kubernetes，Docker Native OrchestrationMarathon）

Rancher的最新版本增加了对几个常见编排引擎的支持。 除了Cattle之外新加支持三个Docker社区中使用最广泛的编排引擎，包括Swarm（Docker Native Orchestration）、Kubernetes和Mesos，它们将为用户提供了更多的功能及可用的编排选项。虽然Docker现在已经发展成为容器化领域的事实标准，但在细分的docker编排引擎市场中目前并没有明确的赢家。在本文中，我们将讨论三个编排引擎系统的功能和特性，并给出它们可能适用的场景用例的建议。

Docker Native Orchestration目前是相对来说项目较新，但它成长速度很快并不断的加入新功能。由于它是Docker官方本身推出的，因此有良好的工具和社区支持，是许多开发人员的默认选择。

Kubernetes是当今最广泛使用的容器编排系统之一，并且得到了Google的支持。

最后Mesos与Mesosphere（或Marathon，其开放源代码版本）支持分层、分类的服务管理方法，许多管理功能可基于独立的插件和应用程序。 因为架构的灵活性这使得它更容易深度定制化部署。当然，这也意味着需要更多的整合工作量。而Kubernetes的常见案例则更偏向于如何构建集群和交付完整统一的综合系统。

## Docker Native Orchestration

![](/assets/dockernative.png)

### 基本结构

Docker Engine 1.12 集成了原生的编排引擎，用以替换了之前独立的Docker Swarm项目。Docker原生集群（Swarm）同时包括了（Docker Engine \/ Daemons），这使原生docker可以任意充当集群的管理\(manager\)或工（worker\)节点角色。工作节点将负责执行任务，运行你启动的容器，而管理节点从用户那里获得任务说明，负责整合现有的集群，维护集群状态。​并且您可以有多个管理节点以支持高可用，但推荐的数量不超过七个。管理节点会保障集群内部环境状态的强一致性，以及基于 [Raft](https://raft.github.io) 协议实现状态备份 ，与所有一制性算法一样，拥有更多manager节点意味着更多的性能开销，同时在manager节点内部保持一致性表明Docker原生编排没有外部依赖性，集群管理更容易。

### 可用性

Docker将使用单节点Docker的概念扩展到Swarm集群。如果你熟悉docker那你学习swarm相当容易。你想要使docker节点环境到迁移到运行swarm集群群也相当简单，（译者按：如果现有系统使用Docker Engine，则可以平滑将Docker Engine切到Swarm上，无需改动现有系统。）你只需要在其中的一个docker节点运行使用 docker swarm init命令创建一个集群，在您要添加任何其他节点，使用，通过docker swarm join命令加入到刚才创建的集群中。之后您可以象在一个独立的docker环境下一样使用同样的Docker Compose的模板及相同的docker命令行工具。

### 功能集

Docker Native Orchestration 使用与Docker Engine和Docker Compose来支持。您仍然可以使用象原先在单个节点一样的链接服务（link\)，创建卷\(volume\)和定义公开端口\(expose）功能。 除了这些，还有两个新的概念，服务（ services ）和网络（ networks ）。

docker服务是在您的节点上启动的一定数量的始终保持运行的一组容器，并且如果其中一个容器死了，它支持自动更换。有两种类型的服务，复制（replicated）或全局（global）。 复制（replicated）服务在集群中维护指定数量的容器（意味着这个服务可以随意 scale），全局（global）服务在每个群集节点上运行容器的一个实例（译者按：不多, 也不少）。 要创建复制（replicated）服务，请使用以下命令。



