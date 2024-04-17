---
layout    : post
title     : "记录：使用mybatis插入枚举类型数据时遇到的问题"
date      : 2019-10-17
lastupdate: 2019-10-17
categories: java mybatis
---



报错如下：不能将枚举类型转换为String

```
org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.type.TypeException: Could not set parameters for mapping: ParameterMapping{property='userSex', mode=IN, javaType=class java.lang.String, jdbcType=null, numericScale=null, resultMapId='null', jdbcTypeName='null', expression='null'}. 
Cause: org.apache.ibatis.type.TypeException: Error setting non null for parameter #3 with JdbcType null . Try setting a different JdbcType for this parameter or a different configuration property. Cause: java.lang.ClassCastException: com.example.hello.enums.UserSexEnum cannot be cast to java.lang.String 

	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:78)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:440)
	at com.sun.proxy.$Proxy100.insert(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.insert(SqlSessionTemplate.java:271)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:62)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:57)
	at com.sun.proxy.$Proxy102.insert(Unknown Source)
	at com.example.hello.MybatisXMLTest.testInsert(MybatisXMLTest.java:41)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
```
看到报错就知道是类型转换出了问题。

## 解决方法如下：

手动指定typehandler为typeHandler=org.apache.ibatis.type.EnumTypeHandler
![在这里插入图片描述](/assets/img/2019-10-17-mybatis-eumn-insert-prob-zh/1.png)

## DEBUG调试：

![在这里插入图片描述](/assets/img/2019-10-17-mybatis-eumn-insert-prob-zh/2.png)
在SqlSourceBuilder类中对Mapper中的标签语句进行解析，把#{}标记的语句转换为preparestatement里用问号作占位符的SQL语句，在转换过程中确定属性的类型。如下图所示，默认情况下mybatis已经把枚举类型转换为String类型，所以后面获取的typehandler也是StringTypeHandler，才会出现上面的报错情况。

![在这里插入图片描述](/assets/img/2019-10-17-mybatis-eumn-insert-prob-zh/3.png)
在手动指定typeHandler之后，可以看到把枚举类型typehandler注册进去了。
![在这里插入图片描述](/assets/img/2019-10-17-mybatis-eumn-insert-prob-zh/4.png)
==这里有个疑问，为什么默认情况下枚举类型会使用String的typeHandler而不去使用默认的枚举EnumTypeHandler呢？欢迎大家讨论。==