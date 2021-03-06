# ELK 整理（Rancher 平台部署）

## 参考资源

[elasticsearch 权威指南](https://www.gitbook.com/book/looly/elasticsearch-the-definitive-guide-cn/details "gitbook地址")    https:\/\/www.gitbook.com\/book\/looly\/elasticsearch-the-definitive-guide-cn\/details

[ ELK STACK 中文指南](http://kibana.logstash.es/content/index.html "中文指南")     http:\/\/kibana.logstash.es\/content\/index.html

## 本文目标

1. 实现elk 监控rancher 平台数据库 （PXC \)
2. 整理elk 在rancher 平台的部署实施策略
3. 编写Rancher elk日志管理平台 操作手册 ， 运维手册
4. 整理及编写新生镜像与elk 平台如何整合，制定相关镜像制作规范，制定相应elk 平台的部署实施规范。
5. 尽量寻找及测试elk 平台的中文界面方案，实现平台可视中文化
6. 整理日志平台操作脚本及图形化模板

## 第一章  整理elk在Rancher 平台的部署策略

![](/assets/logstash-arch.jpg)

Logstash 社区通常习惯用 _shipper_，_broker_ 和 _indexer_ 来描述数据流中不同进程各自的角色。如下图：

[logstash 中文手册](http://www.kancloud.cn/hanxt/elk/155902 "中文手册")   http:\/\/www.kancloud.cn\/hanxt\/elk\/155902

[Rancher 官网资源](http://rancher.com/running-our-own-elk-stack-with-docker-and-rancher/ "实践1 ") http:\/\/rancher.com\/running-our-own-elk-stack-with-docker-and-rancher\/

[ELASTIC 官网](https://www.elastic.co/ "官网")   https:\/\/www.elastic.co\/

### 部署logstash

#### 安装脚本

```
#！/bin/bash
rpm --import http://packages.elasticsearch.org/GPG-KEY-elasticsearch
cat > /etc/yum.repos.d/logstash.repo <<EOF
[logstash-5.0]
name=logstash repository for 5.0.x packages
baseurl=http://packages.elasticsearch.org/logstash/5.0/centos
gpgcheck=1
gpgkey=http://packages.elasticsearch.org/GPG-KEY-elasticsearch
enabled=1
EOF
yum clean all
yum install logstash
```

#### 测试指令1

```
 bin/logstash -e 'input{stdin{}}output{stdout{codec=>rubydebug}}'
```

此命令启动logstash ， 之后你输入hello word  ， 但看输出

![](/assets/logstash1.png)

#### 检查命令2

top   输入Ｈ　查看进程

![](/assets/logstash2.png)

_小贴士：logstash 很温馨的给每个线程都取了名字，输入的叫xx，过滤的叫\|xx_

#### 插件安装

##### 获取插件地址 https:\/\/github.com\/logstash-plugins

```
在线安装  /plugin install logstash-pluinput-jdbc
```

```
升级插件 /plugin update logstash-input-jdbc
```

```
 卸载插件 /plugin uninstall logstash-input-jdbc
```

##### 更换淘宝源  https:\/\/ruby.taobao.org

`yum install -y gem`

`gem sources --remove http://rubygems.org/`

`gem sources -a https://ruby.taobao.org/`

`gem sources -l`

#### 管道设计原理



