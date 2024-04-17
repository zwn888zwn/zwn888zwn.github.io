---
layout    : post
title     : "overlay网络(tun_vxlan_gre)"
date      : 2021-03-26
lastupdate: 2021-03-26
categories: 网络 tun vxlan gre
---

#  overlay网络(tun/vxlan/gre)

![](/assets/img/2021-03-26-overlay-net-zh/16661831919269.jpg)
我们需要在已有的宿主机网络上，再通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络。所以，这种技术就被称为：Overlay Network（覆盖网络)。

## UDP/TCP封包（TUN）

在 Linux 中，TUN 设备是一种工作在三层（Network Layer）的虚拟网络设备。TUN 设备的功能非常简单，即：在操作系统内核和用户应用程序之间传递 IP 包。

当操作系统将一个 IP 包发送给 flannel0 设备之后，flannel0 就会把这个 IP 包，交给创建这个设备的应用程序，也就是 Flannel 进程。这是一个从内核态（Linux 操作系统）向用户态（Flannel 进程）的流动方向。

flanneld 在收到 container-1 发给 container-2 的 IP 包之后，就会把这个 IP 包直接封装在一个 UDP 包里，然后发送给 Node 2。不难理解，这个 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，则是 container-2 所在的宿主机 Node 2 的地址。

每台宿主机上的 flanneld，都监听着一个 8285 端口，所以 flanneld 只要把 UDP 包发往 Node 2 的 8285 端口即可。
![](/assets/img/2021-03-26-overlay-net-zh/16661837545711.jpg)

实际上，相比于两台宿主机之间的直接通信，基于 Flannel UDP 模式的容器通信多了一个额外的步骤，即 flanneld 的处理过程。而这个过程，由于使用到了 flannel0 这个 TUN 设备，仅在发出 IP 包的过程中，就需要经过三次用户态与内核态之间的数据拷贝，如下所示：
![](/assets/img/2021-03-26-overlay-net-zh/16661838777081.jpg)


## GRE/VXLAN

### VXLAN（一个VTEP支持多个VXLAN隧道）

VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络。

而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point。
![](/assets/img/2021-03-26-overlay-net-zh/16662328454788.jpg)

#### 封装外部数据帧

![](/assets/img/2021-03-26-overlay-net-zh/16661917026316.jpg)
VXLAN 头里有一个重要的标志叫作 VNI，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。

Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。分配了 4789 作为 VTEP 设备的 UDP 端口（以前 Linux VXLAN 用的默认端口是 8472，目前这两个端口在许多场景中仍有并存的情况）

这个 UDP 包该发给哪台宿主机呢？flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库。

```
# 在Node 1上，使用“目的VTEP设备”的MAC地址进行查询
$ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```

发往我们前面提到的“目的 VTEP 设备”（MAC 地址是 5e:f8:4f:00:e3:37）的二层数据帧，应该通过 flannel.1 设备，发往 IP 地址为 10.168.0.3 的主机(**主机的MAC地址早就ARP知道**)。显然，这台主机正是 Node 2，UDP 包要发往的目的地就找到了。

### 手动配置

#### 手动维护 fdb 表

如果提前知道目的容器 MAC 地址和它所在主机的 IP 地址，也可以通过更新 fdb 表项来减少广播的报文数量。

创建 vtep 的命令：

```shell
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 \
    nolearning
```

这次我们添加了 `nolearning` 参数，这个参数告诉 vtep 不要通过收到的报文来学习 fdb 表项的内容，因为我们会自动维护这个列表。

然后可以添加 fdb 表项告诉 vtep 容器 MAC 对应的主机 IP 地址：

```
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.101
$ bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.8.102
$ bridge fdb append 52:5e:55:58:9a:ab dev vxlan0 dst 192.168.8.101
$ bridge fdb append d6:d9:cd:0a:a4:28 dev vxlan0 dst 192.168.8.102
```

如果知道了对方的 MAC 地址，vtep 搜索 fdb 表项就知道应该发送到哪个对应的 vtep 了。需要注意是的，这个情况还是需要默认的表项（那些全零的表项），在不知道容器 IP 和 MAC 对应关系的时候通过默认方式发送 ARP 报文去查询对方的 MAC 地址。

#### 手动维护 ARP 表

除了维护 fdb 表，arp 表也是可以维护的。如果能通过某个方式知道容器的 IP 和 MAC 地址对应关系，只要更新到每个节点，就能实现网络的连通。

但是这里有个问题，我们需要维护的是每个容器里面的 ARP 表项，因为最终通信的双方是容器。到每个容器里面（所有的 network namespace）去更新对应的 ARP 表，是件工作量很大的事情，而且容器的创建和删除还是动态的，。linux 提供了一个解决方案，vtep 可以作为 arp 代理，回复 arp 请求，也就是说只要 vtep interface 知道对应的 `IP - MAC` 关系，在接收到容器发来的 ARP 请求时可以直接作出应答。这样的话，我们只需要更新 vtep interface 上 ARP 表项就行了。

创建 vtep interface 需要加上 `proxy` 参数：

```
$ ip link add vxlan0 type vxlan \
    id 42 \
    dstport 4789 \
    dev enp0s8 \
    nolearning \
    proxy
```

这条命令和上部分相比多了 `proxy` 参数，这个参数告诉 vtep 承担 ARP 代理的功能。如果收到 ARP 请求，并且自己知道结果就直接作出应答。

然后，还需要为 vtep 添加 arp 表项，所有要通信容器的 `IP - MAC`二元组都要加进去。

```
$ ip neigh add 10.20.1.3 lladdr d6:d9:cd:0a:a4:28 dev vxlan0
$ ip neigh add 10.20.1.4 lladdr 52:5e:55:58:9a:ab dev vxlan0
```

[linux 上实现 vxlan 网络 | Cizixs Write Here](https://cizixs.com/2017/09/28/linux-vxlan/)



## GRE（点对点）

以下配置使能GRE的三个选项字段：Checksum、Key和Sequence Number。其中Linux内核支持三个参数的方向性配置，例如从A到B的秘钥使用0x222222，从B到A的秘钥使用0x111111，如果两个方向秘钥相同可使用key参数指定一次即可。校验和csum（icsum和ocsum）和序列号seq（iseq和oseq）同样支持类似的方向性配置。

配置

```bash
#设备A：
$ sudo ip tunnel add gre01 mode gre remote 172.16.0.5 local 172.16.0.3 ttl 255 ikey 0x111111 okey 0x222222 csum seq
$ sudo ip addr add 10.1.1.1/24 dev gre01
$ sudo ip link set gre01 up 

#设备B：
$ sudo ip tunnel add gre01 mode gre remote 172.16.0.3 local 172.16.0.5 ttl 255 ikey 0x222222 okey 0x111111 csum seq
$ sudo ip addr add 10.1.1.2/24 dev gre01
$ sudo ip link set gre01 up 
```

### 协议栈处理细节

![](/assets/img/2021-03-26-overlay-net-zh/16661956984801.jpg)

### FAQ

#### 需要获取内部帧mac嘛？

从抓包结果看，不需要，GRE直接封装IP包。（有的厂家的mGRE也会有二层头）

![](/assets/img/2021-03-26-overlay-net-zh/16662318566566.jpg)

#### GRE头部拿掉后，数据帧如何处理？

通过源码推测，ipgre_rcv() -> ip_tunnel_rcv()后，通过skb_queue_tail(&cell->napi_skbs, skb)又送回NAPI接受队列里去了。

[GRE隧道封装协议及内核处理解析](https://blog.csdn.net/sinat_20184565/article/details/83280247)


### 参考链接

https://community.fs.com/blog/nvgre-vs-vxlan-whats-the-difference.html

