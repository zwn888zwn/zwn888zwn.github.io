---
layout    : post
title     : "DGraph集群机制"
date      : 2023-02-14
lastupdate: 2023-02-14
categories: dgraph
---

----

# DGraph集群机制

A Dgraph cluster consists of the following:

- **Dgraph Alpha database server nodes**: The Dgraph Alpha server nodes in your deployment host and serve data. These nodes also host an `/admin` HTTP and GRPC endpoint that can be used for data and node administration tasks such as backup, export, draining, and shutdown.

- **Dgraph Zero management server nodes**: The Dgraph Zero nodes in your deployment control the nodes in your Dgraph cluster. *Dgraph Zero automatically moves data between different Dgraph Alpha instances* based on the volume of data served by each Alpha instance.

  https://dgraph.io/docs/deploy/overview/



组是什么意思？一个数据的多个副本构成一个组，每个alpha节点服务一个组。

When a new Alpha joins the cluster, it is assigned to a **group** based on the replication factor. If the replication factor is set to `1`, then each Alpha node will serve a different group. If the replication factor is set to `3` and you then launch six Alpha nodes, *the first three Alpha nodes will serve group 1 and next three nodes will serve group 2*. Zero monitors the space occupied by predicates in each group and moves predicates between groups as-needed to rebalance the cluster.

https://dgraph.io/docs/deploy/dgraph-zero/



#### Set up Dgraph Alpha group

The number of replica members per Alpha group depends on the setting of Dgraph Zero’s `--replicas` flag. Above, it is set to 3. So when Dgraph Alphas join the cluster, Dgraph Zero will assign it to an Alpha group to fill in its members up to the limit per group set by the `--replicas` flag.

https://dgraph.io/docs/deploy/production-checklist/



##  Group

Every Alpha server belongs to a particular group, and each group is responsible for serving a particular set of predicates. *Multiple servers in a single group replicate the same data to achieve high availability and redundancy of data.*

https://dgraph.io/docs/design-concepts/concepts/#group

## Replication and Server Failure

Each group should typically be served by at least 3 servers, if available. In the case of a machine failure, other servers serving the same group can still handle the load in that case.

## New Server and Discovery

Dgraph cluster can detect new machines allocated to the [cluster](https://dgraph.io/docs/deploy/cluster-setup/), establish connections, and transfer a subset of existing predicates to it based on the groups served by the new machine.



问题：删除了一个alpha节点，数据会不会丢？

可以看到目前所有alpha节点都在一个组内，组内的每个节点数据都是复制的。

![image-20230214113734831](/assets/img/2023-02-14-dgraph-recover-zh/image-20230214113734831.png)





