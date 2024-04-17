---
layout    : post
title     : "微信小程序ColorUI使用自定义tabbar"
date      : 2019-12-18
lastupdate: 2019-12-18
categories: java 微信小程序
---

原本直接使用demo里的tabbar，发现渲染出来的页面都是component形式的，无法使用Page里的一些事件方法，所以打算使用自定义tabbar改写，这其中又出现了一些问题，和大家分享一下。

### 自定义tabbar
这部分根据官方文档，把tabbar的内容拷过来就可以了。
官方链接: [自定义 tabBar](https://developers.weixin.qq.com/miniprogram/dev/framework/ability/custom-tabbar.html).

![在这里插入图片描述](/assets/img/2019-12-18-wx-mini-color-ui-zh/1.png)
### 样式丢失
随后发现样式丢失，在index.wxss文件中重新引用样式表。
```css
@import "../colorui/main.wxss";
@import "../colorui/icon.wxss";
```
这时候发现大部分样式已经好了，其中还有一些不起作用，比如颜色之类的，再根据自己的需要手动添加，如：
```css
.bg-white {
    background-color: #ffffff;
    color: #666666;
}
.text-green,
.line-green,
.lines-green {
    color: #39b54a;
}
```
### 切换出错
样式调好了之后，点击切换，发现字体颜色乱跳，查了相关文档发现**每个 tab 页下的自定义 tabBar 组件实例是不同的**。
![在这里插入图片描述](/assets/img/2019-12-18-wx-mini-color-ui-zh/20191218190637643.png)
这就意味着我单纯的通过在tabbar里js中一个**selected**变量是判断不出在哪个页面的，只能在进入某个页面时对其进行修改。
tabbar的wxml文件
![在这里插入图片描述](/assets/img/2019-12-18-wx-mini-color-ui-zh/20191218191256638.png)
参照了网上的资料，实现如下：

1. tabbar的js文件中**selected**初始设为null
```javascript
  data: {
    selected: null
  }
```


2. 在每个tabbar页面的**onshow**方法中初始化**selected**的值，第一个页面就设置为0、第二个页面就设置为1其他同理。

```javascript
  onShow:function(){
     this.getTabBar().setData({
       selected: 0
     })
  }
  
```