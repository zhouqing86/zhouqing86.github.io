---
layout: post
comments: false
categories: Java基础
date:   2019-01-15 10:30:54
title: litemall源码阅读
---

<div id="toc"></div>

最近自娱自乐做了个小程序，正好也研究下网上的开源代码，学习小程序开发的同时巩固下自己的Java知识。[https://github.com/linlinjava/litemall](https://github.com/linlinjava/litemall)。

## 工程的多模块化

### Maven

在工程的主目录的pom.xml文件中，定义了

```
<modules>
    <module>litemall-core</module>
    <module>litemall-db</module>
    <module>litemall-wx-api</module>
    <module>litemall-admin-api</module>
    <module>litemall-all</module>
</modules>
```

同时定义了所有的对外依赖。

## Spring

### Spring Boot Test

Spring Boot测试步骤，直接在测试类上面加上如下2个注解：

```
@RunWith(SpringRunner.class)
@SpringBootTest
```

就能取到spring中的容器的实例，如果配置了@Autowired那么就自动将对象注入。

### Spring profiles

`spring.profiles.active=db, core, admin, wx` 设置多个active profile的目的就是在不同的module的配置文件。


### MyBatis
### MyBatis-spring

MapperScan注解在使用Java Config时可以使用。使用的方式如:
```
@MapperScan("org.mybatis.spring.sample.mapper")
public class AppConfig {
}
```


其定义为:

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {
  ...
}
```

Import注解可以用来导入一个常规类如`@Import({A.class,B.class})`，而MapperScannerRegistrar类非常规类，其是ImportBeanDefinitionRegistrar接口的实现。

那么，将使用MapperScannerRegistrar.class

### MyBatis generator



<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
