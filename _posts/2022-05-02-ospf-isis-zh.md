---
layout    : post
title     : "动态路由协议实践与分析"
date      : 2022-05-02
lastupdate: 2022-05-02
categories: 网络 路由协议 isis ospf bgp
---

  

### Ospf基础配置

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240228230836676.png" alt="image-20240228230836676" style="zoom:50%;" />

拓扑cpe1--（eth0，eth0）--vpe1---（eth7，eth7）---vpe2----（eth0，eth0）---cpe2，所有路由器运行ospf协议。cpe1和cpe2的lo端口配置10.0.1.1/24、10.0.2.1/24，要求能互相ping通。

1. 在 CPE1 上的配置：

```
bashCopy code
! CPE1 配置
configure terminal

router ospf
 ospf router-id 1.1.1.1
 network 10.0.1.0/24 area 0
 network 192.168.1.0/30 area 0
 redistribute connected

interface eth0
 ip address 192.168.1.1/30
!

interface lo
 ip address 10.0.1.1/24
!
```

`show ip ospf interface`查看接口ospf启用情况，要把eth0的路由发布到ospf上，eth0上才会启用ospf

1. 在 VPE1 上的配置：

```
bashCopy code
! VPE1 配置

configure terminal

router ospf
 ospf router-id 2.2.2.2
 network 192.168.1.0/30 area 0
 network 192.168.3.0/30 area 0
 redistribute connected

interface eth0
 ip address 192.168.1.2/30
!

interface eth7
  ip address 192.168.3.1/30
!
```

1. 在 VPE2 上的配置：

```
bashCopy code
! VPE2 配置

configure terminal

router ospf
 ospf router-id 3.3.3.3
 network 192.168.2.0/30 area 0
 network 192.168.3.0/30 area 0
 redistribute connected

interface eth0
 ip address 192.168.2.2/30
!

interface eth7
  ip address 192.168.3.2/30
!
```

1. 在 CPE2 上的配置：

```
bashCopy code
! CPE2 配置

configure terminal

router ospf
 ospf router-id 4.4.4.4
 network 10.0.2.0/24 area 0
 network 192.168.2.0/30 area 0
 redistribute connected

interface eth0
 ip address 192.168.2.1/30
!

interface lo
 ip address 10.0.2.1/24
!
```

![image-20240229180404570](/assets/img/2021-05-02-ospf-isis-zh/image-20240229180404570.png)



`frr# show ip ospf interface` 查看ospf在端口的启用情况



### ospf邻居关系建立

#### 报文类型

1. **Hello报文**：OSPF路由器启动后，会通过OSPF接口向外发送Hello报文，用于发现邻居。收到Hello报文的OSPF路由器会检查报文中所定义的一些参数，如果双方的参数一致，就会彼此形成邻居关系。

![image-20240308161124040](/assets/img/2021-05-02-ospf-isis-zh/image-20240308161124040.png)

2. **Database Description (DD)报文**：形成邻居关系的双方不一定都能形成邻接关系，这要根据网络类型而定。只有当双方成功交换DD报文，并能交换LSA之后，才形成真正意义上的邻接关系。

![image-20240308160933153](/assets/img/2021-05-02-ospf-isis-zh/image-20240308160933153.png)

3. **Link State Request (LSR)报文**：在形成邻接关系后，如果一台路由器需要从另一台路由器获取更多的LSA信息，它会发送LSR报文来请求这些信息。

![image-20240308161219362](/assets/img/2021-05-02-ospf-isis-zh/image-20240308161219362.png)

4. **Link State Update (LSU)报文**：当一台路由器收到LSR报文后，它会使用LSU报文来响应这些请求，LSU报文包含了被请求的LSA信息。

![image-20240308161300771](/assets/img/2021-05-02-ospf-isis-zh/image-20240308161300771.png)

5. **Link State Acknowledgment (LSAck)报文**：当一台路由器收到LSU报文后，它会发送LSAck报文来确认收到这些LSA信息。

![image-20240308161931473](/assets/img/2021-05-02-ospf-isis-zh/image-20240308161931473.png)

[OSPF Hello - IP报文格式大全(html) - 华为 (huawei.com)](https://support.huawei.com/enterprise/zh/doc/EDOC1100174722/1596ed0f)



##### 建立邻居

hello你中有我，我中有你

![image-20240308160354195](/assets/img/2021-05-02-ospf-isis-zh/image-20240308160354195.png)

​	(1)RTA和RTB的Router ID分别为1.1.1.1和2.2.2.2。当RTA启动OSPF后，RTA会发送第一个Hello报文。此报文中邻居列表为空，此时状态为Down，RTB收到RTA的Hello报文后，状态由Down变为Init
​	(2)RTB发送Hello报文，此报文中邻居列表为空，当RTA收到报文后，状态由Down变为Init
​	(3)RTB发送邻居列表为1.1.1.1的Hello报文，RTA收到报文后，发现列表中是自己的Router ID，此时状态由Init变为2-way
​	(4)RTA发送邻居列表为2.2.2.2的Hello报文，RTB收到报文后，发现列表中是自己的Router ID，此时状体由Init变为2-way
​	(5)因为邻居未知，所有Hello报文的目的地址不是某个特点的单播地址。邻居从无到有，OSPF采用组播的形式发送Hello报文（目的地址224.0.0.5）

##### 建立邻接

选举dr，同步lsdb

![image-20240308160503176](/assets/img/2021-05-02-ospf-isis-zh/image-20240308160503176.png)

RTA和RTB都不知道对方有哪些LSA，此时RTA和RTB互相交互LSDB清单，这个清单就是DD（Database Description，数据库摘要）报文

RTA和RTB交互DD的时候有一个先后顺序，路由器ID值大的先发DD，小的后发，因此同步最开始需要通过DD报文确定主次，此时DD里面并没有LSDB信息。



#### DR和BDR

只有在**广播**或NBMA类型接口时才会选举DR，在点到点或点到多点类型的接口上不需要选举DR。

- **广播网络（broadcast)**：比如ethernet/令牌环/FDDI网络，由于广播网络中是存在多地址的，而且是广播型的，所有发送的数据包，所有的终端都能收到，每个邻居都发信息，会造成混乱和不必要。
- **NBMA(非广播多路访问)：**比如帧中继，ATM等，但没有广播的能力。这种场景下下，一个路由发送的OSPF报文，这个网络中的其他相连路由器无法收到。这个时候需要在路由器上增加额外配置，比如用指定的atm链路来建立邻居，当atm存在多个router时。这时也需要指定DR和BDR。

##### 选举过程

当一个接口变为有效时，它会开始发送Hello报文，并将DR/BDR字段设置为“0.0.0.0”。同时，它会启动一个等待计时器（**默认为40秒**）。在这个等待期间，如果接口收到的Hello报文中有路由器声称自己是DR或BDR，那么该接口会立即进行DR/BDR选举。

建立起“**Two-way**”的双向通信邻接关系（在OSPF邻居建立过程中见过），然后触发DR/BDR选举机制。

将其中未宣告DR（也就是DR字段为空）Hello报文中的BDR字段收集成一个子集。如果在这个子集中，存在一个或者多个路由器，这个子集可以包含路由器本身，只要priority！=0。则按照优先级高，则成为BDR，若优先级相同，则Router-id大优先的原则，尝试BDR选举。

**选举完了BDR，将开始选举DR**。参照步骤：2），收集DR字段形成一个子集。如果为空，则这个BDR成为DR。如果这个子集不为空，则按照优先级高，则成为DR，若优先级相同，则Router-id大优先的原则，尝试DR选举。

再次选出**BDR**



当一个多址网络中，DR/BDR已经选举完成。一个新的路由器并且优先级更高，或者router-id更高，也不会影响现有的DB

[OSPF分享-DR/BDR详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/113944942)



在OSPF协议中，DR（Designated Router，指定路由器）和BDR（Backup Designated Router，备份指定路由器。

##### DR的作用

- DR是一个广播性、多接入网络中的指定路由器。
- DR负责更新其他所有OSPF路由器的信息。
- DR的存在，可以减少OSPF邻接关系，减少了泛洪流量和重复接受的数据库，从而节省了设备资源和带宽资源

DR和BDR的选举过程是先选BDR再选DR14。此外，DR没有抢占性，也就是说，一旦一个路由器被选为DR，它会一直保持DR的状态，除非它挂掉或者被手动更改。

##### 组播地址

在OSPF协议中，224.0.0.5和224.0.0.6是两个特殊的组播地址：

- 224.0.0.5：这个地址被所有运行OSPF协议的路由器接收。它用于发送Hello包和在洪泛过程中发送特定的OSPF协议数据包1234。这就意味着DR路由器使用这个地址对所有的路由器进行路由的通告。
- 224.0.0.6：这个地址只有DR和BDR路由器能够接收。这就意味着DR other通过这个地址和DR路由器进行交换路由信息。

### LSA类型

[图解 OSPF ：什么是 LSA ? - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/463267606)

[全网最牛逼的OSPF LSA类型详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/679196659)

[OSPF LSA Types Explained⁤ (networklessons.com)](https://networklessons.com/ospf/ospf-lsa-types-explained)

#### 区域内LSA

##### Type1

点对点网络

主要的**功能有以下两点：**

1. 描述路由器的特殊角色，如**Virtual-link、ABR、ASBR**这是通过1类LSA中相关的V、B、E位的置1来体现的，例如如果本台设备是ABR，那么它产生的1类LSA中B位会置1。

2. 描述本路由器在某个区域内部的**直连链路（接口）**及接口COST值

该LSA在区域内透传，例如RTA发给RTB之后，RTB还会原封不动地透传给RTC、RTD。

![image-20240311123821780](/assets/img/2021-05-02-ospf-isis-zh/image-20240311123821780.png)



##### Type2

非点对点网络，LSA由DR生成

描述其在该MA网络上连接的所有路由器的**RouterID**（包括DR自己）以及该MA网络的掩码

知道有多少路由器在 MA 网络中

![image-20240311123911565](/assets/img/2021-05-02-ospf-isis-zh/image-20240311123911565.png)





#### 区域间LSA

##### Type3

abr跨区域传递时，会将type1转为type3

![image-20240311110821286](/assets/img/2021-05-02-ospf-isis-zh/image-20240311110821286.png)

##### Type4

当引入外部rip路由后，r1在lsa中反转1个标志位，表示自己为**ASBR (Autonomous System Border Router)**，经过r2，type1 LSA会转为type4。

![image-20240311110938267](/assets/img/2021-05-02-ospf-isis-zh/image-20240311110938267.png)

##### Type5

type4仅仅是用来定位R1是ASBR，并没有发布路由

外部路由（5.5.5.0/24）由R1直接通过type5进行发布

![image-20240311111251401](/assets/img/2021-05-02-ospf-isis-zh/image-20240311111251401.png)



[OSPF知识点_马广杰——博客的技术博客_51CTO博客](https://blog.51cto.com/maguangjie/1791875)

[OSPFB笔记-五个报文【超详细】[Hello报文，DD报文,LSR报文，LSU报文，LSAck报文\]_ospf 每个network地址都发hello消息-CSDN博客](https://blog.csdn.net/weixin_58757687/article/details/122948223)

#### Stub区域

区域配置成Stub区域，配置了之后，Type4和Type5 LSA都会被过滤掉，只生成一条默认路由



## 路由计算

### 区域内路由

经过type1和type2消息后，区域内所有路由器的lsdb内容一致，拓扑如下

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312102729629.png" alt="image-20240312102729629" style="zoom:50%;" />



根据带宽，计算cost值，带宽越大cost越小

- 和别的路由器互联的**网段**叫transit区域（N6），出方向cost为计算值，入方向cost为0
- 不和和别的路由器互联的**网段**叫stub区域（断株），如lo口的网段，p2p（特殊的一种）

#### 生成有向图

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312102507414.png" alt="image-20240312102507414" style="zoom:50%;" />

#### 生成最短路径树

各个路由器以自己为根节点，先处理transit区域节点，跑最短路径算法，生成最短路径树。

计算过程中，同时保存下一条的网关ip

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312103627214.png" alt="image-20240312103627214" style="zoom:50%;" />

最后把stub区域的网段挂在路由器上，网段下一条就是对应端点id的下一跳，生成完整路由表

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312103825939.png" alt="image-20240312103825939" style="zoom:50%;" />

### 区域外路由

区域外路由只发布路由明细，路由条目挂在abr节点上



# bgp

### 基础配置

cpe和vpe间改为bgp交换路由，cpe as号65535，vpe as号65001。接口IP配置同上

在 CPE1 上的 BGP 配置：

```
bashCopy code
! CPE 配置

router bgp 65535
 neighbor 192.168.1.2 remote-as 65001
 !临时不配置策略
 no bgp ebgp-requires-policy
 !
 address-family ipv4 unicast
  network 10.0.1.0/24
 exit-address-family
!
```

在 VPE1 上的 BGP 配置：

```
bashCopy code
! VPE 配置

router bgp 65001
 no bgp ebgp-requires-policy
 neighbor 192.168.1.1 remote-as 65535
 !
 address-family ipv4 unicast
  !引入内核、直连、ospf路由
  redistribute kernel
  redistribute connected
  redistribute ospf
 exit-address-family
!

router ospf
 redistribute bgp
```

` show ip bgp neighbors 192.168.1.1 advertised-routes` 查看cpe1发布的路由



### 触发路由更新

最开始收不到路由，因为策略没有变化`clear ip bgp 192.168.1.1`

[BGP 维护-触发-重置 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv14834199/)



### 报文抓包

![image-20240307143521746](/assets/img/2021-05-02-ospf-isis-zh/image-20240307143521746.png)

#### open

先建立tcp连接，然后发送open报文。报文中包含自己的as、holdtime、bgp id和 capability等信息。

双方都会发起

![image-20240307151501461](/assets/img/2021-05-02-ospf-isis-zh/image-20240307151501461.png)

#### keepalive

![image-20240307150957265](/assets/img/2021-05-02-ospf-isis-zh/image-20240307150957265.png)

#### update

发布路由

![image-20240307151134676](/assets/img/2021-05-02-ospf-isis-zh/image-20240307151134676.png)

#### refresh

要求对端重新发布路由，即重发update消息

#### notification

clear ip bgp后，断开tcp连接

![image-20240307150603535](/assets/img/2021-05-02-ospf-isis-zh/image-20240307150603535.png)



[BGP——图解5种报文_bgp报文-CSDN博客](https://blog.csdn.net/m0_49864110/article/details/123631210)

[BGP UPDATE - IP报文格式大全(html) - 华为 (huawei.com)](https://support.huawei.com/enterprise/zh/doc/EDOC1100174722/fe267bec)

[BGP基础知识_51CTO博客_BGP知识点](https://blog.51cto.com/maguangjie/2316897)



# isis

## 基础配置

![image-20240313135839899](/assets/img/2021-05-02-ospf-isis-zh/image-20240313135839899.png)

```
# rta
interface lo
 ip router isis sdwan
 isis circuit-type level-1
 ip address 1.1.1.1/32
 
interface eth1
 ip router isis sdwan
 isis circuit-type level-1
 ip address 10.0.0.1/24

router isis sdwan
	net 49.0000.0000.0001.00
	is-type level-1
	
# rtb 
interface lo
 ip router isis sdwan
 isis circuit-type level-1-2
 ip address 2.2.2.2/32
 
interface eth1
 ip router isis sdwan
 isis circuit-type level-1
 ip address 10.0.0.2/24

interface eth2
 ip router isis sdwan
 isis circuit-type level-2-only
 ip address 20.0.0.1/24

router isis sdwan
	net 49.0000.0000.0002.00
	is-type level-1-2
	
# rtc
interface lo
 ip router isis sdwan
 isis circuit-type level-2-only
 ip address 3.3.3.3/32
 
interface eth2
 ip router isis sdwan
 isis circuit-type level-2-only
 ip address 20.0.0.2/24

router isis sdwan
	net 50.0000.0000.0003.00
	is-type level-2-only
```









## 基本概念

### NET结构

网络实体标识(Network Entity Titles,NET)

![image-20240312163711837](/assets/img/2021-05-02-ospf-isis-zh/image-20240312163711837.png)

在实际应用中，System ID可以由MAC或者IP来填充，如下：

​	1921.6804.3114，最终NET:49.0001.1921.6804.3114.00

​	49.0001.6480.99F8.12B2.00



### 网络分层

Level1-普通区

Level2-骨干区

连接Level-1区域和Level-2区域的路由器叫Level-1-2路由器。

默认情况下，所有路由器都是Level-1-2路由器

- **L1/L2 routers** can have the same Area ID as their local area (L1), or a different Area ID (L2) if they connect to multiple areas.



### IS-IS协议格式

![image-20240312165702463](/assets/img/2021-05-02-ospf-isis-zh/image-20240312165702463.png)

#### PDU

type不同，专用头部(Specific Header)内容也不同

![image-20240312165721839](/assets/img/2021-05-02-ospf-isis-zh/image-20240312165721839.png)

IS-IS协议使用的协议数据单元(Protocol Data Unit,PDU)有以下4种

##### Hello

IIH(IS-IS Hello)用来建立和维护邻接关系。不同网络中使用的IIH不一样，广播网络中的Level-1使用的是Level-1 LAN IIH,Level-2使用的是Level-2 LAN IIH，点到点网络则使用P2P IIH。

##### LSP

链路状态报文(Link State PDU,LSP)：分为Level-1 LSP、Level-2 LSP。

##### CSNP

序列号协议数据包(Complete Sequence Numbers Protocol Data Unit,CSNP)：类似于OSPF中的**DD**报文，用来简要描述LSDB数据库中的所有条目。

##### PSNP

部分序列号协议数据包(Partial Sequence Numbers Protocol Data Unit,PSNP)：类似于OSPF中的**LSR**，用来请求具体的LSDB条目



#### TLV

变长字段部分(Variable Length Fields)：也称为类型-长度-取值(Type-Length-Value,TLV)，用来存放参数

![image-20240312185540249](/assets/img/2021-05-02-ospf-isis-zh/image-20240312185540249.png)

举例如下：
TLV：(1,2,0004)表示这是区域地址，长度为2字节，地址是0004；
TLV：(132,4,192.168.0.1)表示这是IP接口地址，长度为4字节，取值为192.168.0.1。
使用TLV结构的好处是灵活性和扩展性好，将来如果有新的特性，则只要新增TLV即可，不需要改变报文的整体结构。



### 同步路由过程

#### 建立邻接

同ospf，你中有我，我中有你

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312224634151.png" alt="image-20240312224634151" style="zoom:80%;" />

##### 目的地址

ISIS所有报文都是通过组播发送的
level1：0**1**:80:c2:00:00:14
level2：01:80:c2:00:00:15

#### 选举DIS

DIS（ospf DR）的ID:49.1921.6800.1001.0**1**（注意最后一个数字的区别）。

与OSPF不同，网络中**只有一个DIS**，没有备份DIS，而且选举是**抢占**式，优先级高的路由器会马上成为DIS

#### 同步LSDB数据库

LSDB同步过程如下：

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312225214401.png" alt="image-20240312225214401" style="zoom:50%;" />

①RTC新加入网络中，接口从Down变成Up，触发**泛洪自己的LSP**；

②DIS把收到的LSP更新到LSDB中，等待CSNP报文定时器超时，然后发送**CSNP**；

③RTC收到CSNP，对比自己的LSDB数据库，然后向DIS发送PSNP请求缺少的LSP；

④DIS收到PSNP后，根据PSNP里面的描述，将完整的条目内容通过LSP发给RTC。

广播网络中，**CSNP**报文由DIS发送，**每10s**发送一次。

#### 路由计算

IS-IS协议也使用SPF算法计算生成树，区域内计算过程和OSPF类似

区域间路由是如何学习的呢？

<img src="/assets/img/2021-05-02-ospf-isis-zh/image-20240312230336946.png" alt="image-20240312230336946" style="zoom:50%;" />

L1里的RTB如何学习外部路由：RTC是L1/2边界路由器，往RTB发送的L1 LSP里面，将**ATT位置1**（ATT(Attachment)：由Level-1-2路由器产生，用来指明始发路由器是否与其他区域相连），该LSP在Area 49.0001里泛洪，RTB收到该LSP后，知道RTC是边界路由器，然后**生成一条默认路由，下一跳指向RTC**,RTA与此类似的。

再来看RTD、RTE如何学习全网路由：RTC把Area 49.0001的路由明细以叶节点的方式挂载在Level-2 LSP中，并发**到骨干区域泛洪**，最终RTD、RTE通过计算**学习到全网路由**。

普通区域以**默认路由**方式学习去往其他区域的路由，骨干区域可以学到全网的**路由明细**，因此对骨干区域的路由器性能要求也会高一些。

[『无题』 » Blog Archive » CCIE SP-ISIS 抓包分析邻接关系 (zhaocs.info)](https://www.zhaocs.info/ccie-sp-isis.html)



## isis和ospf对比

- **骨干概念**：OSPF有严格的骨干区域概念，区域0必须是骨干。而IS-IS的任何区域都可以是骨干，骨干区域是L2路由器的集合。
- LSA与LSP：OSPF使用多种LSA来描述区域内、区域间、外部和NSSA外部信息，需要更多内存来存储LSDB。1IS-IS只有两种LSP（LSP-1和LSP-2，相当于OSPF中的LSA），因此LSDP大大减少；特别是在ABR上，节省了内存。
- 链路成本计算：OSPF在计算链路成本时更灵活，可以基于速度或带宽。IS-IS的度量是静态的（默认为10），不考虑速度，可能导致次优路由。
- 安全性：IS-IS更安全，因为它在数据链路层运行，不可能像OSPF那样使用IP攻击IGP。
- NBMA和点对多点链接：OSPF支持NBMA和点对多点链接，而IS-IS不支持。
- DR和DIS：OSPF选举DR和BDR，而IS-IS只选举一个叫做DIS的DR。
- 区域归属：OSPF路由器可以属于多个区域，区域ID不匹配会阻止邻居关系，而IS-IS路由器只能属于一个区域。
- **虚拟链接**：IS-IS没有虚拟链接的概念，因此在IS-IS中不会因为骨干区域的分割而出现问题。              

当然，这篇文章还提到了IS-IS和OSPF在实施和维护方面的不同：

- **实施复杂性**：OSPF的实施相对复杂，因为它需要更多的规划，特别是在大型网络中。IS-IS的配置和维护相对简单，因为它的层次结构较少。
- **多拓扑支持**：IS-IS支持多拓扑路由，这意味着它可以同时支持IPv4和IPv6，而不需要运行两个独立的协议实例。OSPFv3引入了对IPv6的支持，但OSPFv2和v3是分开的。
- **扩展性**：IS-IS天生支持大型网络，因为它的设计允许更多的灵活性和扩展性。OSPF在大型网络中可能需要更多的细分和规划。
- **路由汇总**：两者都支持路由汇总，但在OSPF中，只有ABR可以进行路由汇总，而在IS-IS中，任何路由器都可以进行汇总。

[OSPF IS-IS comparison - Cisco Community](https://community.cisco.com/t5/networking-blogs/ospf-is-is-comparison/ba-p/4442298)              





vrf区分租户，租户内使用ospf