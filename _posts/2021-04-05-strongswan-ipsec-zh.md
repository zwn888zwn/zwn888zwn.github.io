---
layout    : post
title     : "strongswan配置ipsec"
date      : 2021-04-05
lastupdate: 2021-04-05
categories: 网络 ipsec
---

## swanctl配置模板

```yaml
connections {
    gw-gw {
        proposals = default     // 加密算法  + 认证算法 + DH 分组
        version = 2
        mobike = no
        aggressive = no         // 协商模式
        rekey_time = 86400      // SA生存周期
        local_addrs = <本地 IP>
        remote_addrs = <远端 IP>
        dpd_delay = 10s
        local {
            auth = psk      // uses pre-shared key authentication
            id = <LocalID>
        }
        remote {
            auth = psk
            id = <RemoteID>
        }
        children {
            net-net {
                rekey_time = 86400      // SA生存周期
                local_ts = <本端网段>
                remote_ts = <对端网段>
                updown = /usr/lib/ipsec/_updown iptables
                ah_proposals = default               // 认证算法
                esp_proposals = default              // 加密算法 + DH 分组
                start_action = trap
                dpd_action = clear
            }
        }
    }
}
 
secrets {
    ike-1 {
        id-1 = <LocalID>
        id-2 = <RemoteID>
        secret = <预共享密钥>
    }
}
```



卸载strongswan

apt-get remove --auto-remove strongswan 

systemctl list-unit-files --type=service

systemctl daemon-reload

## 安装strongswan

```bash
# 下载源码包
wget https://download.strongswan.org/strongswan-5.9.10.tar.gz

# 安装编译依赖
apt install gcc make libgmp-dev pkg-config libsystemd-dev

tar zxvf strongswan-5.9.10.tar.gz

 ./configure --prefix=/usr \
            --sysconfdir=/etc \
            --libexecdir=/usr/lib \
            --with-ipsecdir=/usr/lib/strongswan \
            --enable-aesni \
            --enable-systemd \
            --enable-swanctl \
            --enable-chapoly \
            --enable-cmd \
            --enable-dhcp \
            --enable-eap-dynamic \
            --enable-eap-identity \
            --enable-eap-md5 \
            --enable-eap-mschapv2 \
            --enable-eap-radius \
            --enable-eap-tls \
            --enable-farp \
            --enable-files \
            --enable-gcm \
            --enable-md4 \
            --enable-newhope \
            --enable-ntru \
            --enable-sha3 \
            --enable-shared \
            --enable-save-keys \
           --with-systemdsystemunitdir=/lib/systemd/system/  CFLAGS="-g -O0"

make
make install

# 启动strongswan
systemctl enable --now strongswan
```

[Installation Documentation :: strongSwan Documentation](https://docs.strongswan.org/docs/5.9/install/install.html)



### 配置log

vi /etc/strongswan.d/charon-logging.conf

```bash
charon {
    filelog {
        wan_charon_log {
            path = /var/log/charon.log
            default = 1
            time_format = %b %e %T
            append = yes
            ike_name = yes
            time_add_ms = yes
            flush_line = yes
            time_add_ms = yes
        }
    }
    syslog {
        daemon {
            default = -1
        }
    }
}
```



journalctl -u strongswan 查看日志



## 配置swanctl

### swanctl conf

#### R1

vi /etc/swanctl/conf.d/server-ipsec.conf

```bash
connections {
        r1 {
                proposals = default
                version = 2
                mobike = no
                local_addrs = 172.16.0.3
                remote_addrs = 0.0.0.0
                dpd_delay = 10s
                aggressive = yes
                local {
                        auth = psk
                        id = server
                }
                remote {
                        auth = psk
                        id = client
                }
                children {
                        net {
                                local_ts = 100.90.0.1/32
                                remote_ts = 100.90.0.2/32
                                updown = /usr/lib/ipsec/_updown iptables
                                esp_proposals = aes128gcm128-x25519
                                start_action = trap
                                dpd_action = clear
                        }
                }
        }
}

secrets {
        ike-client {
                id-client = client
                id-server = server
                secret = 0sFpZAZqEN6Ti9sqt4ZP5EWcqx
        }
}
```

#### R2

vi /etc/swanctl/conf.d/client-ipsec.conf

```bash
connections {
        r2 {
                proposals = default
                version = 2
                mobike = no
                local_addrs = 100.90.0.2
                remote_addrs = 192.168.28.27
                dpd_delay = 10s
                aggressive = yes
                local {
                        auth = psk
                        id = client
                }
                remote {
                        auth = psk
                        id = server
                }
                children {
                        home {
                        				local_ts = 100.90.0.2/32
                                remote_ts = 100.90.0.1/32
                                updown = /usr/lib/ipsec/_updown iptables
                                esp_proposals = aes128gcm128-x25519
                                start_action = trap
                                dpd_action = clear

                        }
                }
        }
}

secrets {
        ike-server {
                id-client = client
                id-server = server
                secret = 0sFpZAZqEN6Ti9sqt4ZP5EWcqx
        }
}
```

#### 连接

```bash
# server
swanctl -c
# client
swanctl -c
swanctl -i -c home
```



### 其他配置

#### server 

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -t nat -I POSTROUTING -m policy --dir out --pol ipsec -j ACCEPT
iptables -t nat -I POSTROUTING -s 100.90.0.1/32 -d 100.90.0.2/32 -j ACCEPT

ip addr add 100.90.0.1/32 dev lo 

ip r add 100.90.0.2/32 dev eth0 src 100.90.0.1
```

#### client

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -t nat -I POSTROUTING -m policy --dir out --pol ipsec -j ACCEPT
iptables -t nat -I POSTROUTING -s 100.90.0.2/32 -d 100.90.0.1/32 -j ACCEPT

ip addr add 100.90.0.2/32 dev lo 

ip r add 100.90.0.1/32 dev eth0 src 100.90.0.2
```



[Test ikev2/nat-rw-psk (strongswan.org)](https://www.strongswan.org/testing/testresults/ikev2/nat-rw-psk/index.html)



## 关闭连接

Let’s assume we have an `IKE SA` named `**home**` with a `CHILD SA` named `**net**`.

- Terminate the `IKE SA` called `**home**` together with its dependent `CHILD SA` `**net**`

```
$ swanctl --terminate --ike home
```

- Terminate the `CHILD SA` `**net**` only, leaving the parent `IKE SA` `**home**` installed.

```
$ swanctl --terminate --child net
```



## 验证分析

### xfrm配置查看

```bash
root@i-oqnoscve:~# ip x s
src 100.90.0.2 dst 192.168.28.27
        proto esp spi 0xc63791a3 reqid 1 mode tunnel
        replay-window 0 flag af-unspec
        aead rfc4106(gcm(aes)) 0x09ca0bb785145aafa4a2b3ba3e0a889d58115932 128
        encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
        anti-replay context: seq 0x0, oseq 0x0, bitmap 0x00000000
src 192.168.28.27 dst 100.90.0.2
        proto esp spi 0xc54accc8 reqid 1 mode tunnel
        replay-window 32 flag af-unspec
        aead rfc4106(gcm(aes)) 0xdbad21f3250dcf303c42e7ff832b04cf307d240a 128
        encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
        anti-replay context: seq 0x0, oseq 0x0, bitmap 0x00000000
root@i-oqnoscve:~# ip x p
src 100.90.0.2/32 dst 100.90.0.1/32 
        dir out priority 367231 
        tmpl src 100.90.0.2 dst 192.168.28.27
                proto esp spi 0xc63791a3 reqid 1 mode tunnel
src 100.90.0.1/32 dst 100.90.0.2/32 
        dir fwd priority 367231 
        tmpl src 192.168.28.27 dst 100.90.0.2
                proto esp reqid 1 mode tunnel
src 100.90.0.1/32 dst 100.90.0.2/32 
        dir in priority 367231 
        tmpl src 192.168.28.27 dst 100.90.0.2
                proto esp reqid 1 mode tunnel
```

### Ikev2抓包

使用`tcpdump -i any host 192.168.28.27 -nel -w log.pcap`命令抓包



### 解密ike

`/etc/strongswan.d/charon.conf`增加如下配置

```
charon {
        load_modular = yes
        plugins {
                save-keys {
                        esp = yes
                        ike = yes
                        load = yes
                        wireshark_keys = /tmp
                }
        }
```



将`/tmp/ikev2_decryption_table`文件拷贝至Windows下的`C:\Users\zwn88\AppData\Roaming\Wireshark\ikev2_decryption_table`目录下，打开wireshark，可以看到ike包已经被解密

mac下 `/Users/zhangweinan/.config/wireshark/ikev2_decryption_table`

#### 发送和接收方 id

![image-20230529180302299](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230529180302299.png)



#### psk 认证数据

![image-20230529180317717](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230529180317717.png)



#### esp 加密提议

![image-20230529180330470](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230529180330470.png)



#### 发送方感兴趣流

<img src="/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230529180418397.png" alt="image-20230529180418397" style="zoom:67%;" />



#### 接收方感兴趣流

![image-20230529180647917](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230529180647917.png)



https://www.cnblogs.com/hugetong/p/10150992.html

### 解密esp

将对称加密算法的 key 填入 wireshark



```bash
root@i-oqnoscve:~# ip x s
src 100.90.0.2 dst 192.168.28.27
        proto esp spi 0xc7bdc03c reqid 1 mode tunnel
        replay-window 0 flag af-unspec
        aead rfc4106(gcm(aes)) 0xe876b2337833c5e265e15b90474e4f717549123d 128
        encap type espinudp sport 4500 dport 4500 addr 0.0.0.0
        anti-replay context: seq 0x0, oseq 0x3, bitmap 0x00000000
```





![image-20230413231610795](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230413231610795.png)



![image-20230413231730947](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230413231730947.png)



# strongswan算法加密流程

## ipsec双边解密算法配置

**proposals = sm4cbc-sm3-prfsm3-modp2048**（加密算法，完整性算法，伪随机数， DH算法）

**esp_proposals = sm4cbc-sm3**（加密算法，完整性算法）



#### 国密算法

国密算法是国家密码局制定标准的一系列算法。其中包括了对称加密算法，椭圆曲线非对称加密算法，杂凑算法，：

SM1：对称加密算法，加密强度为128位，采用硬件实现，算法不对外公开。
SM2：为国家密码管理局公布的公钥算法，其加密强度为256位。
SM3：密码杂凑算法，可使用软件实现，杂凑值长度为32字节。
SM4：对称加密算法，可使用软件实现，加密强度为128位。



### 1.strongswan启动

  strongswan启动的时候会根据配置文件去加载算法的plugins,每种plugin的代码里面都会注册各种类型的一些算法,以上配置相关算法:

SIGNER: AUTH_HMAC_SM3

CRYPTER: ENCR_SM4_ECB,ENCR_SM4_CBC

HASHER: HASH_SM3

PRF: PRF_HMAC_SM3

DH: MODP_2048_BIT



### 2.双边加载完配置以后,请求方执行swanctl -i -c xxx



##### ①请求方准备配置,发送ikev2协议第一个包

1. 根据-c传入的名称,查找对应的配置,检查配置是否支持,创建新的ike sa,其中会初始化一堆相关任务加载到队列中待合适时候执行.
2. 开始build数据包,会执行一些刚刚的任务:
3. 配置的ike算法(IKE:SM4_CBC_X_128/HMAC_SM3_X/PRF_HMAC_SM3_X/MODP_2048),用来生成payload:SECURITY_ASSOCIATION
4. 计算DH的密钥交换材料**key_exchange_dataA=DH(pubkey,mykeyA),**用来生成payload:KEY_EXCHANGE
5. 在hash.c里面定义了几个函数,里面定义了支持的hash签名算法,用来生成payload:NOTIFY.SIGNATURE_HASH_ALGORITHMS
6. 生成随机数,用来生成payload:NONCE
7. 其它payload包含比如:nat源/目的IP的一些hash数据,重定向相关信息等



#####  ②接收方处理收到包,发送ikev2协议第二个包

1. 根据数据包的信息,找到匹配的配置.
2. 选择匹配的ike算法(IKE:SM4_CBC_X_128/HMAC_SM3_X/PRF_HMAC_SM3_X/MODP_2048)
3. 根据nat的hash信息判断远端是否为nat
4. 根据数据包里的KEY_EXCHANGE算出DH的共享密钥**dh_secret = DH(key_exchange_dataA,mykeyB) = g^ir**
5. 根据数据包NONCE随机数字,PRF算法,以及dh_secret算出**SKEYSEED = prf(Ni | Nr, g^ir) = prf(随机数, dh_secret)**,所有的密钥都是从这个密钥推导出来的
6. 计算出几个key**:{SK_d | SK_ai | SK_ar | SK_ei | SK_er | SK_pi | SK_pr} = prf+(SKEYSEED, Ni | Nr | SPIi | SPIr)**
   SK_d 用来生成CHILD SA,
   SK_ei/SK_er 用来加密
   SK_ai/SK_ar 用来完整性校验
   SK_pi/SK_pr 用来认证
7. 计算DH的密钥交换材料**key_exchange_dataA=DH(pubkey,mykeyA)**,用来生成payload:KEY_EXCHANGE
8. 在hash.c里面定义了几个函数,里面定义了支持的hash签名算法,用来生成payload:NOTIFY.SIGNATURE_HASH_ALGORITHMS
9. 生成随机数,用来生成payload:NONCE
10. 其它payload包含比如:nat源/目的IP的一些hash数据,重定向相关信息等



#####  ③请求方处理收到包,发送ikev2第三个包(加密)

1. 确认选择匹配的ike算法(IKE:SM4_CBC_X_128/HMAC_SM3_X/PRF_HMAC_SM3_X/MODP_2048)

2. 根据数据包NONCE随机数字,PRF算法,以及**dh_secret算出SKEYSEED = prf(Ni | Nr, g^ir) = prf(随机数, dh_secret),**所有的密钥都是从这个密钥推导出来的

3. 计算出几个key:**{SK_d | SK_ai | SK_ar | SK_ei | SK_er | SK_pi | SK_pr} = prf+(SKEYSEED, Ni | Nr | SPIi | SPIr)**
   SK_d 用来生成CHILD SA,
   SK_ei/SK_er 用来加密
   SK_ai/SK_ar 用来完整性校验
   SK_pi/SK_pr 用来认证

4. 根据nat的hash信息判断远端是否为nat

5. 根据配置文件得到自己的psk:IDx'

6. 在hash.c里面定义了几个函数,里面定义了支持的hash签名算法,用来生成payload:NOTIFY.SIGNATURE_HASH_ALGORITHMS

7. 计算**octets=message + nonce + prf(Sk_p, IDx')**

8. 从配置文件中获取secret,计算出自己的**认证信息**:AUTH=prf(prf(secret, keypad), octets),用来生成payload:AUTH

9. 其他payload包括:
   TS_INITIATOR/TS_RESPONDER 根据配置文件traffic selectors
   SECURITY_ASSOCIATION CHILD SA加密算法的信息(SM4_CBC_C_128/HMAC_SM3_X/NO_EXT_SEQ),RNG算法算出spi等信息
   等

10. 生成以上payload以后会使用之前配置的ike算法(SM4_CBC_X_128/HMAC_SM3_X)去做加密和签名,然后发送出去
    加密算法sm4对数据部分进行加密,其中使用的key是**SK_e**
    hash算法sm3对加密后的部分进行签名,其中使用的key是**SK_a**

    

##### ④接受方处理收到包,发送ikev2第四个包(加密)

1. 用配置的ike算法解密数据包
2. 根据③中的算法算出对端的AUTH,与数据包中的AUTH做对比来认证
3. 根据配置文件确认选择生成CHILD SA的选路器,算法,spi等
4. 根据双边的NONCE生成**seed=Ni+Nr**
5. 计算出**encr_i_key ,integ_i_key,encr_r_key,integ_r_key =prf(SK_d,seed)**
6. 然后根据以上配置生成真实的CHILD SA(如xfrm)
7. 计算出自己的认证信息AUTH,用来生成payload:AUTH
8. 其他payload包括:
   TS_INITIATOR/TS_RESPONDER 根据配置文件traffic selectors
   SECURITY_ASSOCIATION RNG算法算出spi等信息
   等
9. 生成以上payload以后会使用之前配置的ike算法(SM4_CBC_X_128/HMAC_SM3_X)去做加密和签名,然后发送出去
   加密算法sm4对数据部分进行加密
   hash算法sm3对加密后的部分进行签名



#####  ⑤请求方处理收到的包

1. 根据③中的算法算出对端的AUTH,与数据包中的AUTH做对比来认证
2. 计算出encr_i_key ,integ_i_key,encr_r_key,integ_r_key等信息,配置真实的CHILD SA(如xfrm)





# xfrm框架原理

## xfrm发包

### 路由查找

4层，udp_sendmsg

```c
int ip_route_output_flow(struct rtable **rp, struct flowi *flp, struct sock *sk, int flags)
{
    int err;

    if ((err = __ip_route_output_key(rp, flp)) != 0)
        return err;

    if (flp->proto) {
        if (!flp->fl4_src)
            flp->fl4_src = (*rp)->rt_src;
        if (!flp->fl4_dst)
            flp->fl4_dst = (*rp)->rt_dst;
        return xfrm_lookup((struct dst_entry **)rp, flp, sk, flags);
    }

    return 0;
}
```

## xfrm_lookup

### 查找匹配xfrm policy

调用 xfrm_lookup()在 SPD 中查找 xfrm_policy

```c
//file: net/xfrm/xfrm_policy.c
int xfrm_lookup(struct dst_entry **dst_p, struct flowi *fl,
		struct sock *sk, int flags)
{
	struct xfrm_policy *policy;
	struct xfrm_state *xfrm[XFRM_MAX_DEPTH];
	struct dst_entry *dst, *dst_orig = *dst_p;
	
	policy = NULL;
	if (sk && sk->sk_policy[1])
		policy = xfrm_sk_policy_lookup(sk, XFRM_POLICY_OUT, fl);
		
}
```

根据IP五元组、协议等进行匹配

```c
//file: include/linux/xfrm.h
static inline int
__xfrm4_selector_match(struct xfrm_selector *sel, struct flowi *fl)
{
	return  addr_match(&fl->fl4_dst, &sel->daddr, sel->prefixlen_d) &&
		addr_match(&fl->fl4_src, &sel->saddr, sel->prefixlen_s) &&
		!((xfrm_flowi_dport(fl) ^ sel->dport) & sel->dport_mask) &&
		!((xfrm_flowi_sport(fl) ^ sel->sport) & sel->sport_mask) &&
		(fl->proto == sel->proto || !sel->proto) &&
		(fl->oif == sel->ifindex || !sel->ifindex);
}

```

 policy 中主要包含 目的地址、源地址、spi、加密方式（ah、esp）、reqid等

```c
//file: include/linux/xfrm.h
struct xfrm_id
{
	xfrm_address_t	daddr;
	__u32		spi;
	__u8		proto;
};
struct xfrm_tmpl
{
/* id in template is interpreted as:
 * daddr - destination of tunnel, may be zero for transport mode.
 * spi   - zero to acquire spi. Not zero if spi is static, then
 *	   daddr must be fixed too.
 * proto - AH/ESP/IPCOMP
 */
	struct xfrm_id		id;

/* Source address of tunnel. Ignored, if it is not a tunnel. */
	xfrm_address_t		saddr;

	__u32			reqid;

/* Mode: transport/tunnel */
	__u8			mode;

/* Sharing mode: unique, this session only, this user only etc. */
	__u8			share;

/* May skip this transfomration if no SA is found */
	__u8			optional;

/* Bit mask of algos allowed for acquisition */
	__u32			aalgos;
	__u32			ealgos;
	__u32			calgos;
};
```

### 根据 policy 查找 state

`nx = xfrm_tmpl_resolve(policy, fl, xfrm, family);`

主要根据 reqid、IP 五元组、加密方式等匹配

设置 state 过期时间

```c
//file: net/xfrm/xfrm_state.c
struct xfrm_state *
xfrm_state_find(xfrm_address_t *daddr, xfrm_address_t *saddr, 
		struct flowi *fl, struct xfrm_tmpl *tmpl,
		struct xfrm_policy *pol, int *err,
		unsigned short family)
{
	unsigned h = xfrm_dst_hash(daddr, family);
	struct xfrm_state *x;
	int acquire_in_progress = 0;
	int error = 0;
	struct xfrm_state *best = NULL;
  
  list_for_each_entry(x, xfrm_state_bydst+h, bydst) {
		if (x->props.family == family &&
		    x->props.reqid == tmpl->reqid &&
		    xfrm_state_addr_check(x, daddr, saddr, family) &&
		    tmpl->mode == x->props.mode &&
		    tmpl->id.proto == x->id.proto) {
			/* Resolution logic:
			   1. There is a valid state with matching selector.
			      Done.
			   2. Valid state with inappropriate selector. Skip.

			   Entering area of "sysdeps".

			   3. If state is not valid, selector is temporary,
			      it selects only session which triggered
			      previous resolution. Key manager will do
			      something to install a state with proper
			      selector.
			 */
    }
```

### 设置dst_entry

 设置dst_entry的dst_output函数为xfrm4_output，处理 AH 或 ESP 封装

```c
//file: net/ipv4/xfrm4_policy.c

static int
__xfrm4_bundle_create(struct xfrm_policy *policy, struct xfrm_state **xfrm, int nx,
		      struct flowi *fl, struct dst_entry **dst_p)
{
		struct dst_entry *dst, *dst_prev
		dst_prev->output	= xfrm4_output;
}
```

### xfrm封装

```c
int xfrm4_output(struct sk_buff *skb)
{
	struct dst_entry *dst = skb->dst;
	struct xfrm_state *x = dst->xfrm;

	xfrm4_encap(skb); //设置 IP 头

	err = x->type->output(skb); //执行 esp 或者 ah 封装
	if (err)
		goto error;

}

```

#### ah 封装

```c
//file: net/ipv4/ah4.c
static struct xfrm_type ah_type =
{
	.description	= "AH4",
	.owner		= THIS_MODULE,
	.proto	     	= IPPROTO_AH,
	.init_state	= ah_init_state,
	.destructor	= ah_destroy,
	.input		= ah_input,
	.output		= ah_output
};
```

#### esp 封装

```c
//file: net/ipv4/esp4.c
static struct xfrm_type esp_type =
{
	.description	= "ESP4",
	.owner		= THIS_MODULE,
	.proto	     	= IPPROTO_ESP,
	.init_state	= esp_init_state,
	.destructor	= esp_destroy,
	.get_max_size	= esp4_get_max_size,
	.input		= esp_input,
	.post_input	= esp_post_input,
	.output		= esp_output
};
```

### 重新执行 output

`err = NET_XMIT_BYPASS;`



[IPSEC实现 - SuperKing - 博客园 (cnblogs.com)](https://www.cnblogs.com/super-king/p/3290848.html)

## xfrm 收包



![image-20230505174856145](/assets/img/2021-04-05-strongswan-ipsec-zh/image-20230505174856145.png)
