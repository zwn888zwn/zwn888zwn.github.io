---
layout    : post
title     : "DGraph恢复单一故障节点"
date      : 2023-02-14
lastupdate: 2023-02-14
categories: dgraph
---

----

# DGraph恢复单一故障节点

## 现象

查看pods，发现dgraph的某一alpha节点Crash了

![image-20230213145940985](/assets/img/2023-02-14-dgraph-recover-zh/image-20230213145940985.png)



## 恢复方法

**建议操作前先进行数据备份**。

恢复原理是3个alpha节点属于同一个组，一个组内的多个服务器会复制相同的数据，以实现数据的高可用性和冗余性，详情见参考链接。

初始化节点通过删除alpha节点工作目录下的**p**和**v**文件夹，新节点加入集群后会进行数据同步。

具体做法如下：

1. 通过`kubectl get pv -o wide`命令，查看alpha节点所在pv

![image-20230213150753435](/assets/img/2023-02-14-dgraph-recover-zh/image-20230213150753435.png)

2. 执行`kubectl describe pv pvc-1364ffd5-219f-4f99-94c3-02c8c26dba71`命令，在source -> path下获取路径`/var/openebs/local/pvc-1364ffd5-219f-4f99-94c3-02c8c26dba71`		

![image-20230213150849853](/assets/img/2023-02-14-dgraph-recover-zh/image-20230213150849853.png)

3. ssh至`s2-node4`，cd至`/var/openebs/local/pvc-1364ffd5-219f-4f99-94c3-02c8c26dba71`目录，删除**p**和**v**文件夹。
4. 回到`s2-master`节点删除pod `wdgraph-dgraph-alpha-0`，等待节点自动重启，发现节点已恢复

![image-20230213150333505](/assets/img/2023-02-14-dgraph-recover-zh/image-20230213150333505.png)



## 参考链接

https://dgraph.io/docs/deploy/dgraph-administration/#delete-database 

