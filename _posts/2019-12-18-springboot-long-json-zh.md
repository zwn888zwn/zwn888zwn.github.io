---
layout    : post
title     : "SpringBoot2.0中使用Jackson导致Long型数据精度丢失问题"
date      : 2019-10-17
lastupdate: 2019-10-17
categories: java springboot
---



## 问题描述
使用SpringBoot将Long类型值返回JSON到前端，发现数值最后几位显示为0，出现精度丢失问题。
![在这里插入图片描述](/assets/img/2019-12-18-springboot-long-json-zh/1.png)

## 解决方法
只需要在springboot中配置自定义类型转换器，将Long类型转换为String类型即可。代码如下：


```java
@SpringBootApplication(exclude = {MultipartAutoConfiguration.class})
public class DemoApplication {

    /**
     * 解决Jackson导致Long型数据精度丢失问题
     * @return
     */
    @Bean("jackson2ObjectMapperBuilderCustomizer")
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        Jackson2ObjectMapperBuilderCustomizer customizer = new Jackson2ObjectMapperBuilderCustomizer() {
            @Override
            public void customize(Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder) {
                jacksonObjectMapperBuilder.serializerByType(Long.class, ToStringSerializer.instance)
                        .serializerByType(Long.TYPE, ToStringSerializer.instance);
            }
        };
        return customizer;
    }

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}

```