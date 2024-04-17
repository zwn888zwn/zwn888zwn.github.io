---
layout    : post
title     : "微信小程序中上传图片到SpringBoot后台"
date      : 2019-12-18
lastupdate: 2019-12-18
categories: java 微信小程序
---

最近做了一个微信小程序，里面需要一个上传图片功能，查阅相关资料实现，下面直接贴代码。

## 小程序端
小程序端js代码，利用**wx.uploadFile**上传图片
```javascript
uploadImg:function(path,itemId,failCallback){
    var that=this;
    wx.uploadFile({
      url: `${that.globalData.url}/imgUpload`,
      filePath: path,//wx.chooseImage返回的tempFilePaths
      header: {
      	//自定义header，可以用来身份认证
        //'Cookie': 'JSESSIONID='+this.globalData.Cookie
      },
      name: 'file',
      formData: {
      	//在上传图片时一起传送表单数据
        //'itemId': itemId
      },
      success: function (res) {
        var data = res.data
        console.log(data)
      },fail:function(err){
        failCallback(err)
        console.log(err)
      }
    })
  }
```
## Java后端
### 1、上传文件配置 
```java
@Configuration
public class UploadConfig {
    //显示声明CommonsMultipartResolver为mutipartResolver
    @Bean(name = "multipartResolver")
    public MultipartResolver multipartResolver() {
        CommonsMultipartResolver resolver = new CommonsMultipartResolver();
        resolver.setDefaultEncoding("UTF-8");
        //resolveLazily属性启用是为了推迟文件解析，以在在UploadAction中捕获文件大小异常
        resolver.setResolveLazily(true);
        resolver.setMaxInMemorySize(40960);
        //上传文件大小 5M 5*1024*1024
        resolver.setMaxUploadSize(5 * 1024 * 1024);
        return resolver;
    }
}
```

### 2、上传文件处理类
@RequestParam("file")中的file对应小程序的name: 'file' ，并且接受表单数据。最终图片上传文件到E:/imgUpload/目录下，并且在数据库中保存item和其对应的图片路径。
```java
    @RequestMapping(value = "imgUpload")
    public HashMap<String,String> imgUpload(@RequestParam("file") MultipartFile multipartFile, HttpServletRequest request,String itemId) throws IOException {
        HashMap<String,String> resultJson=new HashMap();
        if (!StringUtils.isEmpty(itemId) ){//equals null
       		//文件处理类，在数据库中保存文件路径
            //Files files=new Files();
            //files.setItemId(Long.parseLong(itemId));
            //将当前上下文初始化给  CommonsMutipartResolver （多部分解析器）
            CommonsMultipartResolver multipartResolver=new CommonsMultipartResolver(
                    request.getSession().getServletContext());
            //检查form中是否有enctype="multipart/form-data"
            if(multipartResolver.isMultipart(request))
            {
                //将request变成多部分request
                MultipartHttpServletRequest multiRequest=(MultipartHttpServletRequest)request;
                //获取multiRequest 中所有的文件名
                Iterator iter=multiRequest.getFileNames();
                while(iter.hasNext())
                {
                    //一次遍历所有文件
                    MultipartFile file=multiRequest.getFile(iter.next().toString());
                    if(file!=null)
                    {
                        String path="E:/imgUpload/"+file.getOriginalFilename();
                        //上传
                        file.transferTo(new File(path));
					    files.setFileName(file.getOriginalFilename());//设置文件名 后期改为具体路径？
					    //文件处理类，在数据库中保存文件路径
                        filesService.save(files);
                    }

                }

            }

            resultJson.put(SessionConstant.API_RESULT,SessionConstant.API_SUCCESS);
        }else {
            resultJson.put(SessionConstant.API_RESULT,SessionConstant.API_FAIL);
        }
        return resultJson;
    }
```

### 3、获取图片文件
在**application.properties**中配置静态资源映射路径。从loaclhost:9090/imagesApi/文件名/  可以直接访问图片。
```
#静态资源地址暴露  图片
spring.mvc.static-path-pattern=/imagesApi/**
spring.resources.static-locations=file:E://imgUpload//  

```