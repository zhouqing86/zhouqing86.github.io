---
layout: post
comments: false
categories: "Spring-Boot"
date:   2016-12-19 18:30:54
title: Spring Boot常用注解
---

<div id="toc"></div>

Spring中有大量的注解完成了很多Magic的功能。

## @SpringBootApplication
此注解注解在Spring Boot的XXXApplication类（有main函数，程序启动的入口）上，其结合了三个注解: @Configuration, @EnableAutoConfiguration 和 @ComponentScan。

```
@SpringBootApplication // same as @Configuration @EnableAutoConfiguration @ComponentScan
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

### @Configuration

[@Configuration API文档](http://docs.spring.io/spring-framework/docs/4.3.5.RELEASE/javadoc-api/org/springframework/context/annotation/Configuration.html)

[GitHub Test Example](https://github.com/zhouqing86/spring-boot-annotation/blob/master/src/test/java/com/example/AnnotationConfigApplicationContextTest.java)

Spring Boot中使用AnnotationConfigApplicationContext来实现基于Java的配置类加载。在这里做个简单的使用测试：

定义TestConfiguration类并使用注解@Configuration:

```
@Configuration
public class TestConfiguration {

    @Bean
    public String message() {
        return "Hello,World!";
    }
}
```

使用AnnotationConfigApplicationContext来获取TestConfiguration中的Bean。
```
@Test
public void testGetBeanMessage() throws Exception {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(TestConfiguration.class);
    assertEquals("Hello,World!", ctx.getBean("message"));
}
```

AnnotationConfigApplicationContext也可以扫描整个包。
```
@Test
public void testScanPackages() throws Exception {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.example");
    ctx.refresh();
    assertEquals("Hello,World!", ctx.getBean("message"));
    //assertEquals("Hello,World!", ctx.getBean(String.class));
}
```

如果希望使用XML的Spring Boot配置，仍建议从@Configuration类开始，使用@ImportResouce注解加载XML配置文件。
```
@Configuration
@ImportResource(locations={"classpath:application-bean.xml"})
//@ImportResource(locations={"file:/configuration/application-bean1.xml"})
public class ConfigClass {
}
```

### @EnableAutoConfiguration
Refer to [Spring Boot Auto Configuration](http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#using-boot-auto-configuration).  建议将此注解加在最主要的@Configuration类，如XXXApplication上。

Spring Boot将尝试根据项目引入的依赖来配置程序，如引入了HSQLDB，不需要手动配置任何数据库连接的类，Spring Boot会自动搞定这些。

那么问题来了，你是否怀疑自动配置的侵入性太强？ 其实还好:

- 当你定义了自己的DataSource类时，Spring Boot就不会使用默认的DataSource配置。

- 使用exclude或excludeName来告诉SpringBoot不要自动加载某一些配置。
`@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})`

- properties中设置`spring.autoconfigure.exclude`。

```
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
    org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
    org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration
```
或者

```
spring.autoconfigure.exclude[0]=org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
spring.autoconfigure.exclude[1]=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
spring.autoconfigure.exclude[2]=org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
spring.autoconfigure.exclude[3]=org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration
```

### @ComponentScan


[@ComponentScan Example](http://www.javarticles.com/2016/01/spring-componentscan-annotation-example.html)

[@ComponentScan API Doc](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/ComponentScan.html#resourcePattern--)

在@Configuration类上使用，使用如`@ComponentScan("org.my.pkg")`，`@ComponentScan(basePackages = "org.my.pkg")`。

@ComponentScan会打开默认的过滤开关`useDefaultFilters=true`, 使用默认的resourcePattern=`**/*.class`。

如果不想让一些类被扫描到，可以使用excludeFilters, 可以使用excludeFilters属性，下面的例子是过滤注解为@Service的所有类。我们也可以过滤自定义注解，但是自定义的注解必须实现TypeFilter。

```
@ComponentScan(
    basePackages = {"foo.bar", "foo.baz"},
    excludeFilters = @ComponentScan.Filter(
       value= Service.class,
       type = FilterType.ANNOTATION
    )
 )
```

如果想让一些带有自定义注解的类被扫描到，可以使用includeFilters。

```
@Configuration
@ComponentScan(basePackages = { "foo.bar","foo.baz" },
        includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, value = Custom.class),
        useDefaultFilters = false)
```

## @RestController
Spring 4.0引入@RestController，此注解是@Controller与@ResponseBody的组合。

### @Controller


### @ResponseBody

当Controller类或者Controller类中的方法注解@ResponseBody, Spring将返回值直接写入到http response中。

Spring是如何达成对@ResponseBody的支持的呢？Spring中定义了一些HttpMessageConverters，HTTPMessageConverter的职责是从将request body中的内容转换成所需的类以及根据定义的mine type返回response body。Spring会循环匹配所有的HTTPMessageConverter直到找到一个能够处理给定mime type的。


### @RequestMapping

## 参考资料
- [Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring MVC Framework and REST](https://www.genuitec.com/spring-frameworkrestcontroller-vs-controller/)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
