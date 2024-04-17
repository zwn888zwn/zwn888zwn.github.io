---
layout    : post
title     : "微信小程序登录状态维护-java后台"
date      : 2020-03-12
lastupdate: 2020-03-12
categories: java 微信小程序
---

**先上一张小程序官方的登录时序图**（https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html）
![在这里插入图片描述](/assets/img/2020-03-12-wx-mini-session-zh/1.png)
因为**http是无状态的**，所以维护登录状态的目的是为了标识，每一次请求是哪个用户发送过来的。

一般的javaweb开发在我们第一次访问页面时，java后台会自动生成一个会话ID保存在浏览器的cookie中，来标识每个用户。再次访问时，服务器可以根据这个jsessionID找到session对象，读取或存储数据。
![在这里插入图片描述](/assets/img/2020-03-12-wx-mini-session-zh/2.png)
而我们用微信小程序发送请求是不保存cookie的，那要怎么维护状态呢？一个最简单的做法就是在微信小程序端记录**jsessionID**值，然后每次请求附加在http请求头上，这样登录状态还是交给java原生session来维护，非常方便。当然也可以在请求头里自定义某个字段，使用Redis等分布式缓存充当session。
![在这里插入图片描述](/assets/img/2020-03-12-wx-mini-session-zh/3.png)
![在这里插入图片描述](/assets/img/2020-03-12-wx-mini-session-zh/4.png)
**服务端代码片段：**

```java
        HttpSession session=request.getSession();
        HttpGet httpGet=new HttpGet("https://api.weixin.qq.com/sns/jscode2session?appid="+appId+"&secret="+appSecret+"&js_code="+code+"&grant_type=authorization_code");
        CloseableHttpResponse httpResponse=(CloseableHttpResponse) httpClient.execute(httpGet);
        String result=EntityUtils.toString(httpResponse.getEntity());
        //检查请求是否成功
        JSONObject resObj=JSON.parseObject(result);
        String openId=resObj.getString("openid");
        String session_Key=resObj.getString("session_key");
        
        //成功就模拟登录保存用户信息openid到数据库，返回服务器sessionID，session_key和openid保存到服务器维护登录状态
        
        resMap.put("sessionId",session.getId());
        return resMap;
```