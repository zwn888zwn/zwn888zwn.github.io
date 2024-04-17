---
layout    : post
title     : "导出数据包至用户空间解析的技术选型"
date      : 2023-12-18
lastupdate: 2023-12-18
categories: 网络 防火墙
---

----


{:toc}

----

将反病毒、入侵防御、行为管控、邮件过滤等安全功能组件嵌入企业站点的SD-WAN网关，以docker形式部署。首先需要把包导入到用户空间分析，要考虑对接方式、性能、安全和开发工作量。

# cpe 侧

主要关注从 lan 口出去的流量

## 包从内核拷贝到用户空间

### NFQUEUE-流量镜像

NFQUEUE 是一种 iptables 和 ip6tables 的目标（an iptables and ip6tables target），将网络包处理决定委托给用户态软件。比如，下面的规则将所有接收到的网络包（all packet going to the box）委托给监听中的用户态程序去决策。

```bash
iptables -A INPUT -j NFQUEUE --queue-num 0 
```

用户态软件必须监听队列 0（connect to queue 0），并从内核获取消息；然后给每个网络包给出判决（verdict）。



NFQUEUE 因为需要将网络包发送给用户程序，所以它的性能并不高；但相比于堆叠 iptables 规则，NFQUEUE 的处理方式更加灵活。而对于 `XDP`、`xt_bpf` 等拥有同等灵活性的新技术而言，NFQUEUE 的适用范围更广、能够适配很多老旧系统。

[learn-by-example/nfqueue/main.go at main · Asphaltt/learn-by-example · GitHub](https://github.com/Asphaltt/learn-by-example/blob/main/nfqueue/main.go)

```go
package main

import (
	"context"
	"encoding/binary"
	"fmt"
	"net"

	"github.com/florianl/go-nfqueue"
)

type packet []byte

func (p packet) srcIP() net.IP {
	return net.IP(p[12:16])
}

func (p packet) dstIP() net.IP {
	return net.IP(p[16:20])
}

func (p packet) srcPort() uint16 {
	tcphdr := p[20:]
	return binary.BigEndian.Uint16(tcphdr[:2])
}

func (p packet) dstPort() uint16 {
	tcphdr := p[20:]
	return binary.BigEndian.Uint16(tcphdr[2:4])
}

func handlePacket(q *nfqueue.Nfqueue, a nfqueue.Attribute) int {
	if a.Payload != nil && len(*a.Payload) != 0 {
		pkt := packet(*a.Payload)
		fmt.Printf("tcp connect: %s:%d -> %s:%d\n", pkt.srcIP(), pkt.srcPort(), pkt.dstIP(), pkt.dstPort())
	}
	_ = q.SetVerdict(*a.PacketID, nfqueue.NfAccept)
	return 0
}

func main() {
	cfg := nfqueue.Config{
		NfQueue:     1,
		MaxQueueLen: 2,
		Copymode:    nfqueue.NfQnlCopyPacket,
	}

	nfq, err := nfqueue.Open(&cfg)
	if err != nil {
		fmt.Println("failed to open nfqueue, err:", err)
		return
	}

	ctx, stop := context.WithCancel(context.Background())
	defer stop()
	if err := nfq.RegisterWithErrorFunc(ctx, func(a nfqueue.Attribute) int {
		return handlePacket(nfq, a)
	}, func(e error) int {
		return 0
	}); err != nil {
		fmt.Println("failed to register handlers, err:", err)
		return
	}

	select {}
}
```



```bash
# 0.0000-10.0092 sec  3.64 GBytes  3.12 Gbits/sec
# 0.0000-10.3263 sec  42.6 MBytes  34.6 Mbits/sec

iptables -t raw -I PREROUTING -j NFQUEUE --queue-num=1 --queue-bypass
iptables -D PREROUTING -j NFQUEUE --queue-num 1 --queue-bypass -t raw
```



[iptables-nfqueue 的使用 - LeonHwang's Blogs (asphaltt.github.io)](https://asphaltt.github.io/post/iptables-nfqueue-usage/)

[使用 Go 对接 iptables NFQUEUE 的例子 - LeonHwang's Blogs (asphaltt.github.io)](https://asphaltt.github.io/post/go-nfnetlink-example/)

[一文吃透 iptables-nfqueue - LeonHwang's Blogs (asphaltt.github.io)](https://asphaltt.github.io/post/iptables-nfqueue/#复制到用户程序的网络包的大小)

[subgraph/go-nfnetlink: A library for communicating with Linux netfilter subsystems over netlink sockets. (github.com)](https://github.com/subgraph/go-nfnetlink)

[Using NFQUEUE and libnetfilter_queue – To Linux and beyond ! (regit.org)](https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/)



### Bridge的强制泛洪-流量镜像

当一个网络包经过 bridge 而 bridge 在其 fdb 中找不到对应记录，即 bridge 不确定该网络包该通过那个网口发出去时，bridge 会将该网络包通过除收包网口外的其他所有网口发出去，俗称泛洪。 当将 ageing time 设置为 0 时， 意即 bridge 无需维持 fdb ，每当收到一个网络包，都将这个网络包泛洪出去。

```bash
brctl setageing docker0 0

brctl setageing docker0 300

#[  1] 0.0000-10.0108 sec  18.1 GBytes  15.5 Gbits/sec
#[  1] 0.0000-10.0085 sec  6.49 GBytes  5.57 Gbits/sec
```



```go
package main

import (
	"log"

	"github.com/songgao/packets/ethernet"
	"github.com/songgao/water"
)

func main() {
	config := water.Config{
		DeviceType: water.TAP,
	}
	config.Name = "O_O"

	ifce, err := water.New(config)
	if err != nil {
		log.Fatal(err)
	}
	var frame ethernet.Frame

	for {
		frame.Resize(1500)
		n, err := ifce.Read([]byte(frame))
		if err != nil {
			log.Fatal(err)
		}
		frame = frame[:n]
		log.Printf("Dst: %s\n", frame.Destination())
		log.Printf("Src: %s\n", frame.Source())
		log.Printf("Ethertype: % x\n", frame.Ethertype())
		log.Printf("Payload: % x\n", frame.Payload())
	}
}
```

缺点：如果目标mac地址为bridge本身的网关地址，则不会泛洪（比如由lan出internet或隧道），只能拿到回来的包



[Linux bridge 强制泛洪实验 - LeonHwang's Blogs (asphaltt.github.io)](https://asphaltt.github.io/post/linux-bridge-flood-experiment/)

### tc-流量镜像

[Port mirroring with Linux bridges « \1 (backreference.org)](https://backreference.org/2014/06/17/port-mirroring-with-linux-bridges/)



## 零拷贝

### AF_PACKET

linux提供了原始套接字RAW_SOCKET，可以抓取数据链路层的报文。这样可以对报文进行深入分析。今天介绍一下AF_PACKET的用法，分为两种方式。

- 第一种方法是通过套接字，打开指定的网卡，然后使用recvmsg读取，实际过程需要需要将报文从内核区拷贝到用户区。
- 第二种方法是使用packet_mmap，使用**共享内存**方式，在内核空间中分配一块内核缓冲区，然后**用户空间程序调用mmap映射到用户空间**。将接收到的skb拷贝到那块内核缓冲区中，这样用户空间的程序就可以直接读到捕获的数据包了。

PACKET_MMAP减少了系统调用，不用recvmsg就可以读取到捕获的报文，相比原始套接字+recvfrom的方式，减少了一次拷贝和一次系统调用。libpcap就是采用第二种方式。suricata默认方式也是使用packet mmap抓包



export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890





```go
// Copyright 2018 Google, Inc. All rights reserved.
//
// Use of this source code is governed by a BSD-style license
// that can be found in the LICENSE file in the root of the source
// tree.

// afpacket provides a simple example of using afpacket with zero-copy to read
// packet data.
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"runtime/pprof"
	"time"

	"github.com/google/gopacket"
	"github.com/google/gopacket/afpacket"
	"github.com/google/gopacket/layers"
	"github.com/google/gopacket/pcap"
	"golang.org/x/net/bpf"

	_ "github.com/google/gopacket/layers"
)

var (
	iface      = flag.String("i", "any", "Interface to read from")
	cpuprofile = flag.String("cpuprofile", "", "If non-empty, write CPU profile here")
	snaplen    = flag.Int("s", 0, "Snaplen, if <= 0, use 65535")
	bufferSize = flag.Int("b", 64, "Interface buffersize (MB)")
	filter     = flag.String("f", "port not 22", "BPF filter")
	count      = flag.Int64("c", -1, "If >= 0, # of packets to capture before returning")
	verbose    = flag.Int64("log_every", 1, "Write a log every X packets")
	addVLAN    = flag.Bool("add_vlan", false, "If true, add VLAN header")
)

type afpacketHandle struct {
	TPacket *afpacket.TPacket
}

func newAfpacketHandle(device string, snaplen int, block_size int, num_blocks int,
	useVLAN bool, timeout time.Duration) (*afpacketHandle, error) {

	h := &afpacketHandle{}
	var err error

	if device == "any" {
		h.TPacket, err = afpacket.NewTPacket(
			afpacket.OptFrameSize(snaplen),
			afpacket.OptBlockSize(block_size),
			afpacket.OptNumBlocks(num_blocks),
			afpacket.OptAddVLANHeader(useVLAN),
			afpacket.OptPollTimeout(timeout),
			afpacket.SocketRaw,
			afpacket.TPacketVersion3)
	} else {
		h.TPacket, err = afpacket.NewTPacket(
			afpacket.OptInterface(device),
			afpacket.OptFrameSize(snaplen),
			afpacket.OptBlockSize(block_size),
			afpacket.OptNumBlocks(num_blocks),
			afpacket.OptAddVLANHeader(useVLAN),
			afpacket.OptPollTimeout(timeout),
			afpacket.SocketRaw,
			afpacket.TPacketVersion3)
	}
	return h, err
}

// ZeroCopyReadPacketData satisfies ZeroCopyPacketDataSource interface
func (h *afpacketHandle) ZeroCopyReadPacketData() (data []byte, ci gopacket.CaptureInfo, err error) {
	return h.TPacket.ZeroCopyReadPacketData()
}

// SetBPFFilter translates a BPF filter string into BPF RawInstruction and applies them.
func (h *afpacketHandle) SetBPFFilter(filter string, snaplen int) (err error) {
	pcapBPF, err := pcap.CompileBPFFilter(layers.LinkTypeEthernet, snaplen, filter)
	if err != nil {
		return err
	}
	bpfIns := []bpf.RawInstruction{}
	for _, ins := range pcapBPF {
		bpfIns2 := bpf.RawInstruction{
			Op: ins.Code,
			Jt: ins.Jt,
			Jf: ins.Jf,
			K:  ins.K,
		}
		bpfIns = append(bpfIns, bpfIns2)
	}
	if h.TPacket.SetBPF(bpfIns); err != nil {
		return err
	}
	return h.TPacket.SetBPF(bpfIns)
}

// LinkType returns ethernet link type.
func (h *afpacketHandle) LinkType() layers.LinkType {
	return layers.LinkTypeEthernet
}

// Close will close afpacket source.
func (h *afpacketHandle) Close() {
	h.TPacket.Close()
}

// SocketStats prints received, dropped, queue-freeze packet stats.
func (h *afpacketHandle) SocketStats() (as afpacket.SocketStats, asv afpacket.SocketStatsV3, err error) {
	return h.TPacket.SocketStats()
}

// afpacketComputeSize computes the block_size and the num_blocks in such a way that the
// allocated mmap buffer is close to but smaller than target_size_mb.
// The restriction is that the block_size must be divisible by both the
// frame size and page size.
func afpacketComputeSize(targetSizeMb int, snaplen int, pageSize int) (
	frameSize int, blockSize int, numBlocks int, err error) {

	if snaplen < pageSize {
		frameSize = pageSize / (pageSize / snaplen)
	} else {
		frameSize = (snaplen/pageSize + 1) * pageSize
	}

	// 128 is the default from the gopacket library so just use that
	blockSize = frameSize * 128
	numBlocks = (targetSizeMb * 1024 * 1024) / blockSize

	if numBlocks == 0 {
		return 0, 0, 0, fmt.Errorf("Interface buffersize is too small")
	}

	return frameSize, blockSize, numBlocks, nil
}

func main() {
	flag.Parse()
	if *cpuprofile != "" {
		log.Printf("Writing CPU profile to %q", *cpuprofile)
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal(err)
		}
		if err := pprof.StartCPUProfile(f); err != nil {
			log.Fatal(err)
		}
		defer pprof.StopCPUProfile()
	}
	log.Printf("Starting on interface %q", *iface)
	if *snaplen <= 0 {
		*snaplen = 65535
	}
	szFrame, szBlock, numBlocks, err := afpacketComputeSize(*bufferSize, *snaplen, os.Getpagesize())
	if err != nil {
		log.Fatal(err)
	}
	afpacketHandle, err := newAfpacketHandle(*iface, szFrame, szBlock, numBlocks, *addVLAN, pcap.BlockForever)
	if err != nil {
		log.Fatal(err)
	}
	err = afpacketHandle.SetBPFFilter(*filter, *snaplen)
	if err != nil {
		log.Fatal(err)
	}
	source := gopacket.ZeroCopyPacketDataSource(afpacketHandle)
	defer afpacketHandle.Close()

	bytes := uint64(0)
	packets := uint64(0)
	for ; *count != 0; *count-- {
		data, _, err := source.ZeroCopyReadPacketData()
		if err != nil {
			log.Fatal(err)
		}
		bytes += uint64(len(data))
		packets++
		if *count%*verbose == 0 {
			_, afpacketStats, err := afpacketHandle.SocketStats()
			if err != nil {
				log.Println(err)
			}
			log.Printf("Read in %d bytes in %d packets", bytes, packets)
			log.Printf("Stats {received dropped queue-freeze}: %d", afpacketStats)
		}
	}
}

```





```bash
apt install libpcap-dev
./afpacket -i docker0 -log_every 5000
#[  1] 0.0000-10.0258 sec  4.13 GBytes  3.54 Gbits/sec
#[  1] 0.0000-10.0212 sec  4.28 GBytes  3.67 Gbits/sec
```

[suricata抓包方式之一 AF_PACKET - Rabbit_Dale - 博客园 (cnblogs.com)](https://www.cnblogs.com/Anker/p/6040873.html)

[Packet MMAP — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/networking/packet_mmap.html)



### netmap

Netmap 是一个高性能收发原始数据包的框架，由 Luigi Rizzo 等人开发完成，其包含了内核模块以及用户态库函数。其目标是，不修改现有操作系统软件以及不需要特殊硬件支持，实现用户态和网卡之间数据包的高性能传递。其原理图如下，数据包不经过操作系统内核进行处理，用户空间程序收发数据包时，直接与网卡进行通信。

在 Netmap 框架下，内核拥有数据包池，发送环接收环上的数据包不需要动态申请，有数据到达网卡时，当有数据到达后，直接从数据包池中取出一个数据包，然后将数据放入此数据包中，再将数据包的描述符放入接收环中。内核中的数据包池，通过 mmap 技术映射到用户空间。用户态程序最终通过 netmap_if 获取接收发送环 netmap_ring，进行数据包的获取发送。

[GitHub - luigirizzo/netmap: Automatically exported from code.google.com/p/netmap](https://github.com/luigirizzo/netmap)

[netmap(4) (freebsd.org)](https://man.freebsd.org/cgi/man.cgi?query=netmap&sektion=4)

[使用 netmap 提升 ixgbe 性能_netmap 性能-CSDN博客](https://blog.csdn.net/fengfengdiandia/article/details/52594758)

[netmap 使用小例子_python3 netmap 例子-CSDN博客](https://blog.csdn.net/liyu123__/article/details/80938494)

[高性能网络I/O框架-netmap源码分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/436737795)

[Netmap性能优化 ：网卡多队列 - 哔哩哔哩 (bilibili.com)](https://www.bilibili.com/read/cv15487599/)

[深入理解用户态协议栈之TCP/IP 的设计——高性能收发原始数据包的框架(Netmap) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/347726528)



### eBPF

TODO



### DPDK

TODO



## 通过打流测试性能影响

```bash
sysctl -w net.ipv4.ip_forward=1
systemctl restart docker

iperf -s
iperf -c 192.168.9.134
# 0.0000-10.0106 sec  3.69 GBytes  3.17 Gbits/sec

# 启动10个nginx
docker run -d nginx

```

**结论**：

NFQUEUE方式只适合处理sync握手包，如果处理所有流量，性能损耗了99%

bridge的强制泛洪-流量镜像，转发性能由原来的15g变成5g。缺点：如果目标mac地址为bridge自身mac地址，则不会泛洪（比如由lan出internet或隧道），只能拿到回来的包。

af_packet抓包的防火墙方案，使用共享内存方式，性能损耗大约5%。还需注意和iptables过滤的处理顺序

