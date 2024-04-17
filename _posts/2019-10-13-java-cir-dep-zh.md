---
layout    : post
title     : "Spring解决循环依赖"
date      : 2019-10-17
lastupdate: 2019-10-17
categories: java spring
---



### 解决循环依赖

#### **Use** ***@Lazy***

打破循环的一个简单方法是告诉 Spring 延迟初始化一个 bean。因此，它不会完全初始化该 bean，而是创建一个代理将其注入到另一个 bean 中。注入的 Bean 仅在第一次需要时才会完全创建。

```SQL
@Component
public class CircularDependencyA {

    private CircularDependencyB circB;

    @Autowired
    public CircularDependencyA(@Lazy CircularDependencyB circB) {
        this.circB = circB;
    }
}
```

#### **使用 Setter/Field 注入**

构造函数注入

```SQL
@Component
public class CircularDependencyA {

    private CircularDependencyB circB;

    @Autowired
    public CircularDependencyA(CircularDependencyB circB) {
        this.circB = circB;
    }
}
```

setter注入

```SQL
@Component
public class CircularDependencyA {

    private CircularDependencyB circB;

    @Autowired
    public void setCircB(CircularDependencyB circB) {
        this.circB = circB;
    }

    public CircularDependencyB getCircB() {
        return circB;
    }
}
```

测试

```SQL
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = { TestConfig.class })
public class CircularDependencyIntegrationTest {

    @Autowired
    ApplicationContext context;

    @Bean
    public CircularDependencyA getCircularDependencyA() {
        return new CircularDependencyA();
    }

    @Bean
    public CircularDependencyB getCircularDependencyB() {
        return new CircularDependencyB();
    }

    @Test
    public void givenCircularDependency_whenSetterInjection_thenItWorks() {
        CircularDependencyA circA = context.getBean(CircularDependencyA.class);

        Assert.assertEquals("Hi!", circA.getCircB().getMessage());
    }
}
```

https://www.baeldung.com/circular-dependencies-in-spring