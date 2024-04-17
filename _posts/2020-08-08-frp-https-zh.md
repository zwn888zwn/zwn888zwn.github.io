---
layout    : post
title     : "https内网穿透-使用“飞鸽”-微信小程序"
date      : 2020-08-08
lastupdate: 2020-08-08
categories: 网络
---

微信小程序需要https才能访问，如何在我们没有服务器的情况下，直接从小程序访问后台接口呢？这里我们使用内网穿透，具体流程如下：
## 1 注册飞鸽
打开“飞鸽”内网穿透官网，注册登录。
https://www.fgnwct.com/
然后点击【开通隧道】
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/1.png)
在页面中依次选择http-自定义域名-本地ip端口443
这里自定义域名填写阿里云域名中的一个cname别名，如 bieming.xxx.com
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/2.png)
在【隧道管理】中点 【+】号，记录下方框中的地址【free.vipnps.vip】
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/3.png)
详细步骤可以查看：https://www.yuque.com/fgnwct/help/about

## 2 设置域名cname，并申请SSL证书
打开阿里云，选择【域名】，点击【添加记录】
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/4.png)

选择cname，设置自己的主机记录，记录值填写刚刚提到方框内容【free.vipnps.vip】
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/5.png)

随后在【域名】申请一个免费证书
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/6.png)

申请成功后绑定刚刚的域名，会添加一个txt记录，然后点击【下载】，这里我们选择NGINX![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/7.png)

## 3 NGINX配置
官网下载NGINX，打开conf文件夹，建立一个cert目录，把证书解压进去。
复制以下内容至nginx.conf，并根据注释修改。
主要关注【server_name、ssl_certificate、ssl_certificate_key、proxy_pass】
```

	# 以下属性中以ssl开头的属性代表与证书配置有关，其他属性请根据自己的需要进行配置。
	server {
		listen 443 ssl;   #SSL协议访问端口号为443。此处如未添加ssl，可能会造成Nginx无法启动。
		server_name  localhost;  #将localhost修改为您证书绑定的域名，例如：www.example.com。
		ssl_certificate cert/4327388_domain.pem;   #将domain name.pem替换成您证书的文件名。
		ssl_certificate_key cert/4327388_domain.key;   #将domain name.key替换成您证书的密钥文件名。
		ssl_session_timeout 5m;
		ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;  #使用此加密套件。
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;   #使用该协议进行配置。
		ssl_prefer_server_ciphers on;   
		location / {
			   proxy_pass http://127.0.0.1:9099; #这里是本地后台访问地址
			}
	}    
```
配置完成后，启动NGINX,打开127.0.0.1:80，启动成功！
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/8.png)

## 4 启动飞鸽，进行内网穿透
进入官网https://www.fgnwct.com/，下载客户端
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/9.png)
解压，并在当前路径打开命令行窗口，输入【隧道管理】中的启动命令
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/10.png)
至此，已经全部配置完毕，可以用https方式访问自定义的域名。
![在这里插入图片描述](/assets/img/2020-08-08-frp-https-zh/11.png)