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

6.  第二步骤，使用rancher 平台启动，定制docker compose 固定ip ，分配主机名（感谢华相，alen\)
  slave2:

   image: yktmongo110

   hostname: slave2

   container\_name: slave2

   extra\_hosts:

   - "mongodb:10.42.250.10"

   - "slave2:10.42.250.11"

   - "slave1:10.42.250.12"

   labels:

   - "io.rancher.container.requested\_ip=10.42.250.11"

  slave1:

   image: yktmongo110

   hostname: slave1

   container\_name: slave1

   extra\_hosts:

   - "mongodb:10.42.250.10"

   - "slave2:10.42.250.11"

   - "slave1:10.42.250.12"

   labels:

   - "io.rancher.container.requested\_ip=10.42.250.12"

  master:

   image: yktmongo110

   hostname: master

   environment:

   ROLE: master

   SLAVE1: slave1

   SLAVE2: slave2

   container\_name: mongodb

   extra\_hosts:

   - "mongodb:10.42.250.10"

   - "slave2:10.42.250.11"

   - "slave1:10.42.250.12"

   labels:

   - "io.rancher.container.requested\_ip=10.42.250.10"



