1. 集群使用centos 5.6   \(redhat 6.4  ssl 包有问题\)

2. 需求： 
  加访问控制  keyfile  及密码 认证  auth
  一键部署

3. 遇到的问题
  加认证后，集群不成功（因地址变更或host文件变动导致） ， 用户创建集群失败，无法创建新的管理员及管理用户及密码（因为auth 强制要求用户登录必需要密码，而全局用户在集群启动前无权限创建新用户）

4. 解决思路

  1 。 创建集群使用固定IP及主机名（rancher\)
  2.     创建用户信息及保存用户密码之后再开启auth 及access control 。并保证重启服务后地址及host不变。

  基本部署

5. 构建基础镜像

  `docker build -t yktmongo110 .`

  `Dockerfile :`

  `FROM hasedon/centos6.5`

  `MAINTAINER wise2c`

  `## 修改容器时钟信息`

  `RUN rm -rf /etc/localtime && ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`

  `##准备mongo 安装包`

  `ADD mongodb-linux-x86_64-3.0.8.tar /ykt/`

  `RUN mv /ykt/mongodb-linux-x86_64-3.0.8 /ykt/mongodb`

  `#WORKDIR /ykt/node/bin/`

  `#创建mongodb 存放数据文件夹`

  `RUN mkdir -p /ykt/mongodb/data/db`

  `VOLUME /ykt/mongodb/data/db`

  `##加mongodb 配置文件`

  `ADD mongo.conf /ykt/mongodb/`

  `## 加mongodb启动脚`

  `ADD startmongo.sh /root/startmongo.sh`

  `##创建用户密码脚本`

  `ADD createuserdb.js /opt/createuserdb.js`

  `## 加容器启动脚本`

  `ADD run.sh /run.sh`

  `RUN chmod +x /root/startmongo.sh && chmod +x /run.sh`

  `## 加key备用`

  `ADD mongodb-keyfile /root/`

  `RUN chmod 600 /root/mongodb-keyfile`

  `## 加载环境变量`

  `ENV MONGODBADMINPASS kYbB66ubZNYGmRuH #设置admin用户密码`

  `ENV MONGODBUSERPASS JsCUgh9z8ihotL9U #设置ykt用户密码`

  `#ENV PRIMARYIP mongocluster_master_1 #主节点mongoIP地址`

  `#ENV SECONDARY1IP mongocluster_slaveA_1 #从节点1mongoIP地址`

  `#ENV SECONDARY2IP mongocluster_slaveB_2 #从节点2mongoIP地址`

  `EXPOSE 27017`

  `EXPOSE 28017`

  `#### 加端口宣告到主机`

  `#CMD /run.sh`

  `CMD ["bash","/root/startmongo.sh"]`

6. startmong.sh 脚本内容：
  bash-4.1\# cat \/root\/startmongo.sh

  \#\/ykt\/mongodb\/bin\/mongod --config \/ykt\/mongodb\/mongo.conf --smallfiles --nojournal

  \#!\/bin\/bash

  set -x

  MONGO="\/ykt\/mongodb\/bin\/mongo"

  MONGOD="\/ykt\/mongodb\/bin\/mongod"

  $MONGOD --fork --config \/ykt\/mongodb\/mongo.conf --noprealloc --smallfiles

  sleep 30

  \#checkSlaveStatus\(\){

  \#$MONGO --host slave1 --eval db

  \#while \[ "$?" -ne 0 \]

  \#do

  \#echo "Waiting for slave to come up..."

  \#sleep 15

  \#$MONGO --host slave2 --eval db

  \#done

  \#}

  if \[ "$ROLE" == "master" \]

  then

  $MONGO --eval "rs.initiate\(\)"

  \#checkSlaveStatus

  $MONGO --eval "rs.add\(\"slave1:27017\"\)"

  $MONGO --eval "rs.add\(\"slave2:27017\"\)"

  fi

  tailf \/dev\/null



1. 第二步骤，使用rancher 平台启动，定制docker compose 固定ip ，分配主机名（感谢华相，alen\)
  slave2:

  image: yktmongo110

  hostname: slave2

  container\_name: slave2

  extra\_hosts:

  * "mongodb:10.42.250.10"

  * "slave2:10.42.250.11"

  * "slave1:10.42.250.12"



labels:

* "io.rancher.container.requested\_ip=10.42.250.11"

  slave1:

  image: yktmongo110

  hostname: slave1

  container\_name: slave1

  extra\_hosts:

* "mongodb:10.42.250.10"

* "slave2:10.42.250.11"

* "slave1:10.42.250.12"


labels:

* "io.rancher.container.requested\_ip=10.42.250.12"

  master:

  image: yktmongo110

  hostname: master

  environment:

  ROLE: master

  SLAVE1: slave1

  SLAVE2: slave2

  container\_name: mongodb

  extra\_hosts:

* "mongodb:10.42.250.10"

* "slave2:10.42.250.11"

* "slave1:10.42.250.12"


labels:

* "io.rancher.container.requested\_ip=10.42.250.10"

RANCHER-COMPOSE

```
  slave2:
```

scale: 1

slave1:

scale: 1

mongodb:

scale: 1

OK , 现在正常启动的， 我们先使用命令行，去验证用户及产生集群的脚本是否执行正确。

首先，查看集群现在状态：

`bash-4.1# /ykt/mongodb/bin/mongo --eval "rt.startSet()"`

MongoDB shell version: 3.0.8

connecting to: test

2017-01-10T09:11:08.871+0800 E QUERY ReferenceError: rt is not defined

at \(shell eval\):1:1

集群未定义， 现在创建集群：

`bash-4.1# /ykt/mongodb/bin/mongo -eval "rs.initiate()"`

MongoDB shell version: 3.0.8

connecting to: test

\[object Object\]

不知道这个提示是否正常，也许集群是只有管理员可建 ？ 但登录验证已经有primay 的信息：

第二步骤 ， 为集群加节点：

`/ykt/mongodb/bin/mongo`

`MongoDB shell version: 3.0.8`

`connecting to: test`

`Welcome to the MongoDB shell.`

`For interactive help, type "help".`

`For more comprehensive documentation, see`

`http://docs.mongodb.org/`

`Questions? Try the support group`

`http://groups.google.com/group/mongodb-user`

`Server has startup warnings:`

`2017-01-10T09:05:07.030+0800 I CONTROL [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.`

`2017-01-10T09:05:07.030+0800 I CONTROL [initandlisten]`

`ykt:PRIMARY> use adminuse admin`

`switched to db admin`

`ykt:PRIMARY>`

