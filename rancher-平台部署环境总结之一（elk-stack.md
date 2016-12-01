# rancher elk 部署步骤：

###       rancher 在应用商店提供了elk 应用， 本次部署的日志分析平台（由应用商店内三个独立的应用栈构成，分别是elasticsearch , logstash , kibana 构成。同时我们增加了logspout 应用在每台主机，用于负责监听及采集主机及主机之上的rancher 启动的容器动作并将日志发送至elk 平台进行分析。

         

