---
layout    : post
title     : "keepalived优先级策略"
date      : 2023-04-01
lastupdate: 2023-04-01
categories: linux keepalived
---

----

# keepalived优先级策略

## 策略配置

### 优化v1

原本逻辑:
默认主设备优先级为100，备设备为50；

1. 10秒ping一次对端隧道的bgpip，如果3次ping不通，认为隧道不可用，优先级-100
2. 只要有一个wan口（wan_data下）的状态为up，或者unknown且有地址，则为正常；否则优先级-120 

优化后的逻辑：

1. tracker去ping2条隧道的bgpip，如果2条都连续3次ping不通，优先级-100
2. 如果pop组的wan口都为down，优先级-120
3. 起一个gorouting定时上报延时

### 优化v2

1. tracker改为公用，为mwan和ha等服务提供ping功能，支持动态增加修改ping ip，已存在的ip修改时monitors不重置；
2. tracker获取指定网卡的monitors数据（map）；定时读取g1 g2的monitors数据，更新ha优先级；

mwan通常6秒切换完成，如果切换后还不通，则切换ha，预计10秒以内

### 调整优先级方法

**Track script方式：**
如果脚本执行结果为0，并且weight配置的值大于0，则优先级相应的增加；
如果脚本执行结果非0，并且weight配置的值小于0，则优先级相应的减少；
其他情况，维持原本配置的优先级，即配置文件中priority对应的值。

```shell
vrrp_script itf_check_script {
    script "/usr/bin/bash -c '[ -f /wan/data/itf_down ] && exit 1 || exit 0'"
        interval 1
        timeout 3
        weight -120
        rise 3
        fall 3
}
track_script {
        itf_check_script
}
```

**Track file方式:**
文件的值是一个number，程序会读取它；
如果weight为0，则文件中的非零值将被视为失败状态。否则该值将乘以track_file语句中配置的权重。如果结果小于-253，任何监视脚本的 VRRP实例或同步组都将转换为故障状态；
在本例中，track文件weight为1。当file为0时，优先级 = priority；当file为-100时，优先级 = 1 * -100 + priority

```shell
vrrp_track_file track_tunnel_status_file {
    file /wan/data/tunnel
    weight 1
}
track_file {
    track_tunnel_status_file
}
```

优先级只会增减一次，不会循环叠加；
实验发现，如果weight=1且track_file都为-100时，是一主一备，vip还在（优先级均为1，先进入备状态的设备为备）；如果weight=0且track_file都为-100时，进入fault状态，两台设备都无vip

**通过tcpdump查看优先级，当前prio为100**
vrrp协议号112

```shell
root@i-p00lytew:~# tcpdump proto 112
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0:ha-i, link-type EN10MB (Ethernet), capture size 262144 bytes
17:21:20.305313 IP i-p00lytew > vrrp.mcast.net: VRRPv2, Advertisement, vrid 196, prio 100, authtype simple, intvl 1s, length 20
```


### 开启keepalived日志

`vim /etc/rsyslog.d/50-default.conf `把下面四行前面的 # 都去掉

```shell
#*.=info;*.=notice;*.=warn;\
#       auth,authpriv.none;\
#       cron,daemon.none;\
#       mail,news.none          -/var/log/messages
```

`service rsyslog restart` 重启rsyslog服务
`tail -f /var/log/messages` 就能看到 keepalived日志



## 附完整keepalived.conf配置

```shell
global_defs {
    vrrp_skip_check_adv_addr
    script_user root
    enable_script_security
}
vrrp_track_file track_tunnel_status_file {
    file /wan/data/tunnel
    weight 1
}
vrrp_script itf_check_script {
    script "/usr/bin/bash -c '[ -f /wan/data/itf_down ] && exit 1 || exit 0'"
        interval 1
        timeout 3
        weight -120
        rise 3
        fall 3
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 196
    preempt_delay 10
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass Zhu88jie
    }
    virtual_ipaddress {
        172.17.10.252/24 dev eth0
    }
    
    unicast_src_ip 10.12.13.24
    unicast_peer {
                10.12.13.20
    }
    
    garp_master_delay 1
    garp_master_refresh 5
    track_interface {
        eth0
    }
    track_file {
        track_tunnel_status_file
    }
    track_script {
       itf_check_script
    }
    notify_master "/etc/keepalived/notify.sh MASTER"
    notify_backup "/etc/keepalived/notify.sh BACKUP"
    notify_fault "/etc/keepalived/notify.sh FAULT"
    notify_stop "/etc/keepalived/notify.sh STOP"
}
```

preempt_delay 300 #表示的含义是，我当前是backup身份，但是我发现对方的master不如我，即优先级比我低，那么我不会立马去抢占，而是等五分钟后再去抢占

## 参考链接

[keepalived之vrrp_script详解](https://www.cnblogs.com/arjenlee/p/9258188.html)

