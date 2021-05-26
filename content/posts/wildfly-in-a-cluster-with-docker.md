---
title: "Wildfly in a Cluster with Docker"
date: 2019-04-26T15:52:23+02:00
draft: false
subtitle: ""
author: "Auryn Engel"
authorLink: "/about"

tags: ["Wildfly", "Coding", "Java"]
categories: ["Development"]

hiddenFromHomePage: false

# featuredImage: "/img/container.jpg"
# featuredImagePreview: "/img/container.jpg"

lightgallery: true
---

Operating Wildfly in a docker container isn’t difficult. The description can be found on the official docker image site from jboss-wildfly. This blog post describes how to start several containers with a wildfly and how to connect them to a cluster. Unfortunately, the Docker-jboss document does not describe how to merge wildflys into a cluster unless they are running on the same machine.

## Why Wildfly in the Cluster?

Wildfly itself offers the possibility to work in a cluster and replicate caches and states. [Stateful Beans](https://www.straub.as/java/ejb/3.0/stateful.html) are no longer the same on one fly, but on several wildflys. If a wildfly fails, either because of network problems or because the computer freezes, other wildflys take over the tasks without losing the state of the application (I know, never use states in a server 😉).

## Wildfly in Docker

How wildfly is brought into a docker container is described very well on the docker page of [jboss](https://hub.docker.com/r/jboss/wildfly/dockerfile). They use the jboss image and add the current wildfly. If you want to make changes, you just have to modify the image. We will use and extend the official jboss wildfly image in this post.

## Wildfly Multicast

Wildfly’s launch order is at the docker:

```Dockerfile
CMD ["/opt/jboss/wildfly/bin/standalone.sh", "-b", "0.0.0.0"]
```

This starts wildfly with its default configuration and binds it to 0.0.0.0

By default wildfly is not multicast capable and would therefore not find another wildfly and share the state. JBoss describes how to start wildfly with a high available service on there site. Under 7.2.1 is described how to use the standalone-ha.xml to run this service. To get all the features of the network, including Infinispan, we will use the standalone-full-ha.xml configuration.

So the option -Djboss.server.default.config=standalone-full-ha.xml must be added. Furthermore we have to set the name and the port offset, otherwise the two wildflys get in each other’s way.

`-Djboss.socket.binding.port-offset=<offset of your choice> -Djboss.node.name=<unique node name>`

If we now start the wildfly local twice, a cluster will be built if there is a deployment.

This should work both locally and on different machines in the same network.

"That another instance of the server can either be on the same machine or on some other machine.

At least this is the documentation of jboss (see 7.2.1). However, it’s not correct.

A few steps are missing from the documentation to make this available.

To use IPV4, we have to give Java the option: `-Djava.net.preferIPv4Stack=true`

In the second step (the most important), we need to bind jgroups to the IP that our computer has. For Linux and Mac we can simply use $(hostname -i).
`-Djgroups.bind_addr=$(hostname -i)

Furthermore, a password is required so that jboss can build a cluster.
`-Djboss.messaging.cluster.password=<PASSWORD>`

Once these steps are done, we can start wildfly on different machines on the same network and you will find yourself in a cluster.

## Wildfly in Docker with Multicast

If we apply all the options described above, we can create a docker file that creates a wildfly that is capable of multicasting. In the example we add a HelloWorld.war, which we can reach later from the outside.

```Dockerfile
FROM jboss/wildfly

ADD helloworld.war /opt/jboss/wildfly/standalone/deployments/

ARG WILDFLY_NAME
ARG CLUSTER_PW

ENV WILDFLY_NAME=${WILDFLY_NAME}
ENV CLUSTER_PW=${CLUSTER_PW}

ENTRYPOINT /opt/jboss/wildfly/bin/standalone.sh -b=0.0.0.0 -bmanagement=0.0.0.0 -Djboss.server.default.config=standalone-full-ha.xml -Djboss.node.name=${WILDFLY_NAME} -Djava.net.preferIPv4Stack=true -Djgroups.bind_addr=$(hostname -i) -Djboss.messaging.cluster.password=${CLUSTER_PW}
```

## docker-compose.yml

The above image can now be used in a docker-compose.yml to create a cluster.

```yaml
version: '3.5'
services:

  wildfly1:
    build:
      context: .
      args:
        WILDFLY_NAME: wildfly_1_test
        CLUSTER_PW: secret_password
    image: wildfly_1_test
    ports:
    - 8080:8080
    networks:
      wildfly_network:

  wildfly2:
    build: 
      context: .
      args:
        WILDFLY_NAME: wildfly_2_test
        CLUSTER_PW: secret_password
    image: wildfly_2_test
    ports:
    - 8081:8080
    networks:
      wildfly_network:

networks:
  wildfly_network:
    ipam:
      driver: default
```

Here, two wildlfy images are created and both booted in a shared network. The network mode is IPAM (very well described here). Docker makes sure that each container gets an individual IP.

Let’s start docker with docker-compose up and build both images and boot the containers.

Now the two wildflys would have to connect to a cluster, this is recognizable by the log:

```text
...
wildfly1_1  | ... INFO  [org.infinispan.CLUSTER] (MSC service thread 1-3) ISPN000094: Received new cluster view for channel ejb: [wildfly_2_test|1] (2) [wildfly_2_test, wildfly_1_test]
wildfly1_1  | ... INFO  [org.infinispan.CLUSTER] (MSC service thread 1-5) ISPN000094: Received new cluster view for channel ejb: [wildfly_2_test|1] (2) [wildfly_2_test, wildfly_1_test]
wildfly1_1  | ... INFO  [org.infinispan.CLUSTER] (MSC service thread 1-1) ISPN000094: Received new cluster view for channel ejb: [wildfly_2_test|1] (2) [wildfly_2_test, wildfly_1_test]
wildfly1_1  | ... INFO  [org.infinispan.CLUSTER] (MSC service thread 1-7) ISPN000094: Received new cluster view for channel ejb: [wildfly_2_test|1] (2) [wildfly_2_test, wildfly_1_test]
...
wildfly2_1  | ... INFO  [org.infinispan.CLUSTER] (remote-thread--p6-t2) [Context=client-mappings] ISPN100002: Starting rebalance with members [wildfly_2_test, wildfly_1_test], phase READ_OLD_WRITE_ALL, topology id 2
```

You can reach helloworld.war at localhost:8080/helloworld/HelloWorld and localhost:8081/helloworld/HelloWorld (the two ports that are forwarded to the host machine).

The used code and the example can be found in the following [Github-Repo](https://github.com/auryn31/wildfly-cluster-docker-example).

Photo by [Guillaume Bolduc](https://unsplash.com/@guibolduc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/container?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
