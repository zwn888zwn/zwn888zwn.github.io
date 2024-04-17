---
layout    : post
title     : "ElasticSearch7.7安装与使用小结"
date      : 2020-05-28
lastupdate: 2020-05-28
categories: es
---

  ## 1 安装ElasticSearch
1官网下载
https://www.elastic.co/cn/downloads/elasticsearch
![在这里插入图片描述](/assets/img/2020-05-28-es-install-zh/1.png)

2 安装中文分词
下载https://github.com/medcl/elasticsearch-analysis-ik/releases，解压到到es的plugins下ik文件夹

3 运行
 bin/elasticsearch (or bin\elasticsearch.bat on Windows)

4 测试访问 http://localhost:9200/
![在这里插入图片描述](/assets/img/2020-05-28-es-install-zh/2.png)

## 2 安装kibana图形界面工具
下载地址：https://www.elastic.co/cn/downloads/kibana
解压，运行bin下面的kibana.bat
方便进行命令测试 http://localhost:5601/app/kibana#/dev_tools/console
![在这里插入图片描述](/assets/img/2020-05-28-es-install-zh/3.png)
![在这里插入图片描述](/assets/img/2020-05-28-es-install-zh/4.png)
从这里我们就可以大概看出，ES类似于一个数据库。问题来了，数据从哪来呢？可以自己put插入也可以从数据库同步。

学习命令操作中文文档：
https://www.elastic.co/guide/cn/elasticsearch/guide/current/running-elasticsearch.html
## 3 Logstash同步MySql数据到ES
步骤可以查看参考链接，亲测有效。其中mysql-connector-java-5.1.38-bin.jar需要下载放到指定目录。

参考链接：
https://blog.csdn.net/qq_43229020/article/details/93199070
https://my.oschina.net/xiaowangqiongyou/blog/1812708
https://www.elastic.co/guide/en/logstash/7.x/plugins-inputs-jdbc.html
https://www.elastic.co/guide/en/logstash/current/configuration.html

## 4 客户端查询ES数据
maven：刚开始用官网里的例子报错，改成下面就好了。
```java
        <!-- ES客户端 -->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>7.0.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.elasticsearch</groupId>
                    <artifactId>elasticsearch</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.elasticsearch.client</groupId>
                    <artifactId>elasticsearch-rest-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>7.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>7.0.0</version>
        </dependency>
```
后面内容可以看下参考链接里的博客，builderQuery，Spring Boot从ES查询数据。

参考链接：
https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html
https://blog.csdn.net/qq_43229020/article/details/97009307