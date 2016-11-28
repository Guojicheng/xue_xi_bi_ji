ou **cannot** use any shell helper \(e.g. use &lt;dbname&gt;, show dbs, etc.\) inside the JavaScript file because they are not valid JavaScript.

The following table maps the most common [mongo](https://docs.mongodb.com/manual/reference/program/mongo/#bin.mongo "mongo") shell helpers to their JavaScript equivalents.

| hell HelpersJavaScript Equivalents |  |
| --- | --- |
| show dbs, show databases | db.adminCommand\('listDatabases'\) |
| use &lt;db&gt; | db = db.getSiblingDB\('&lt;db&gt;'\) |
| show collections | db.getCollectionNames\(\) |
| show users | db.getUsers\(\) |
| show roles | db.getRoles\({showBuiltinRoles: **true**}\) |
| show log &lt;logname&gt; | db.adminCommand\({ 'getLog' : '&lt;logname&gt;' }\) |
| show logs | db.adminCommand\({ 'getLog' : '\*' }\) |
| it | cursor = db.collection.find\(\)
**if** \( cursor.hasNext\(\) \){
   cursor.next\(\);
} |







Recently Rancher introduced the Rancher catalog, an awesome feature that enables Rancher users to one-click deploy common applications and complex services from catalog templates on your infrastructure, and Rancher will take care of creating and orchestrating the Docker containers for you.

Rancher catalog offers a wide variety of applications in its out of the box catalog, including glusterfs or elasticsearch, as well as supporting private catalogs. Today I am going to introduce a new catalog template I developed for deploying a MongoDB replicaset, and show you how I built it.

Catalog Templates

A catalog template is basically a combination of two files:

docker-compose.yml

rancher-compose.yml

These files are used to launch multiple containers to serve a specific purpose, for example creating a Sysdig template will create a Sysdig service that runs on every host in your infrastructure, this is configured in the docker-compose.yml file.

The rancher-compose.yml file will contain a special directive called “.catalog” which contains information about the template like its name and description, also it has an important directive called “questions” which is responsible for collecting information from the user, for example the sysdig template will have the following “.catalog” directive:

.catalog: name: "Sysdig" version: "v0" description: "Container-Native System Visibility and Troubleshooting" uuid: sysdig-0 questions: - variable: "VERSION" label: "Sysdig version" description: "Specify a version of the sysdig container to pull \(default will pull latest stable version\)." type: "string" default: "latest" required: true - variable: "HOST\_EXCLUDE\_LABEL" label: "Host exclude label" description: "Specify a host label here that can be used to exclude deployment of the sysdig container on any given host. Eg: sysdig.exclude\_sysdig=true \(you could then add the label 'sysdig.exclude\_sysdig=true' to any host to exclude sysdig from that host\)." type: "string" default: "sysdig.exclude\_sysdig=true" required: true

The previous file has two questions, first it defines a question about the version of sysdig and store the answer from the user in “VERSION” variable which can be used later in docker-compose.yml file, the second question will ask about the host that will be excluded from running sysdig on it, so when clicking on creating a sysdig application it should look like this:

When starting Rancher platform it clones rancher\/rancher-catalog github repo that contains all services templates.

MongoDB replicaset service

A replicaset is a group of MongoDB processes that maintain the same data set, which provides high availability and redundancy between your replication nodes. Each MongoDB replicaset consists of one primary node and several secondary nodes.

Before creating the MongoDB service template, we should first create the Docker images that will be running in this service, this service will mainly create 3 MongoDB images which is a requirement to create a MongoDB replicaset, and then will initialize the replication procedure between the three nodes.

The MongoDB service creates 3 Docker containers from the MongoDB upstream image, each container will has 2 sidekick containers, that contain all the scripts needed to initialize the replicaset and store the database volume.The sidekick containers are secondary containers that are scheduled with the primary container. These secondary containers can share volumes and namespaces to build a single unit, it can be used for different purposes like defining volumes and then referencing these volumes in the primary container, for more information about Sidekicks, checkout the official documentation of Rancher.

MongoDB sidekick containers

The first sidekick container is called mongo-datavolume which basically a busybox container with a \/data\/db volume, the following is a definition of this sidekick in the docker-compose.yml:

mongo-datavolume: net: none labels: io.rancher.container.hostname\_override: container\_name io.rancher.container.start\_once: true volumes: - \/data\/db entrypoint: \/bin\/true image: busybox

Note that “net: none” will tell Docker to not take any steps to configure the network of the container. In the case of our data volume container networking is not needed. Also, using the “io.rancher.container.start\_once: true” label will make Rancher run the container and have it remain in stopped state while the service remains in active state.

The second sidekick container is called mongo-base container and its a rancher\/mongodb-conf:v0.1.0 image, it contains several shell scripts that are needed by the MongoDB container to both initialize and scale up the replicaset. These scripts are provided to the upstream MongoDB container via shared volume.

I will not go into details of writing the scripts, but I will explain the process that any newly created container in the service follows:

The mongo container will use this sidekick and run entrypoint.sh.

entrypoint.sh will run lowest\_idx.sh script which will check the container’s metadata and check if it is the first container that ran in this service \(lowest id\).

If yes, it will run the initiate.sh in the background and start MongoDB, initiate.sh will wait until all 3 of Mongo instances are up assigned an IP in Rancher environment.

Once all instances are up, it will initiate the replicaset by using rs.initiate and rs.add.

If no \(the container is not the first container in the service\), then it will start Monodb and run scaling.sh in the background.

The scaling.sh will check if there are more than 3 containers in the service.

If yes, then it will connect to the primary MongoDB container and add the new container to the replicaset.

If no, then it will simply exits.

Launching MongoDB Catalog Template

To run this template, first you need to run Rancher server, and register some nodes to it, then by accessing the Applications -&gt; Catalog tab, you will see different services, choose MongoDB template, you should see something like that:

You can also click on “Preview” to see both docker-compose.yml and rancher-compose.yml files, which describe how the containers will start in Rancher environment.

You can check “start services after creating” to run the containers after creating the service, after creating the service, make sure that there is a node registered to Rancher server and then run the service:

After starting the service, it will create 3 containers and their sidekick and will run the process we just described earlier, you should see the containers starting after few minutes:

And to make sure that the replication started, you can either examine the logs, or run the following command in Mongo shell inside any of the running containers:

rs0:SECONDARY&gt; rs.status\(\) { "set" : "rs0", "date" : ISODate\("2015-11-26T22:34:45.135Z"\), "myState" : 2, "members" : \[ { "\_id" : 0, "name" : "MongoDB\_mongo-cluster\_1:27017", "health" : 1, "state" : 1, "stateStr" : "PRIMARY", ……. }, { "\_id" : 1, "name" : "10.42.118.82:27017", "health" : 1, "state" : 2, "stateStr" : "SECONDARY", ……. }, { "\_id" : 2, "name" : "10.42.115.200:27017", "health" : 1, "state" : 2, "stateStr" : "SECONDARY", ……. } \], "ok" : 1 }

Conclusion

Rancher catalog is a great feature that helps users of Rancher platform to run complex application stacks in their environment with just basic prerequisite knowledge of Docker orchestration and management. To learn more, view a recording of our recent meetup on building your own catalog, or join us for our next monthly online meetup.

REGISTER NOW

