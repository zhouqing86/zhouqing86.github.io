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
@Controller类其实就是一个@Component类，只是其带有Controller的语义。

### @ResponseBody

当Controller类或者Controller类中的方法注解@ResponseBody, Spring将返回值直接写入到http response中。

Spring是如何达成对@ResponseBody的支持的呢？Spring中定义了一些HttpMessageConverters，HTTPMessageConverter的职责是从将request body中的内容转换成所需的类以及根据定义的mine type返回response body。Spring会循环匹配所有的HTTPMessageConverter直到找到一个能够处理给定mime type的。

这里需要注意的是，默认是不支持将对象转换为XML的，如果需要将对象response as xml, 需要自定义XML的Converter。

### @RequestMapping
简单的@RequestMapping可以如`@RequestMapping("/home")`。

@RequestMapping 的源代码
```
public @interface RequestMapping {
    String name() default "";

    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    RequestMethod[] method() default {};

    String[] params() default {};

    String[] headers() default {};

    String[] consumes() default {};

    String[] produces() default {};
}
```

对produces和consumes的理解：

- consumes: 指定处理请求的提交内容类型（Content-Type），如设置其为`application/json`, 方法仅处理request Content-Type为`application/json`类型的请求。

- produces: 指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回， 如设置为`application/json`, 方法仅处理request请求中Accept头中包含了`application/json`的请求，同时暗示了返回的内容类型为`application/json`。

### 参数相关注解
与@RequestMapping有相关的注解还有 @RequestParams, @PathVariable 等。


## 其他
### @AutoWired
[AutoWired Annotation Document](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/beans.html#beans-autowired-annotation)

此注解在Spring中是由`AutowiredAnnotationBeanPostProcessor`来处理的。

可以在构造函数(也可以在私有构造函数上)，私有成员变量(不需要定义set方法)，setter方法，也可以在任意的配置方法上。

@Autowired 还支持在集合或数组上使用，如

```
@Autowired
private MovieCatalog[] movieCatalogs;

@Autowired
private Set<MovieCatalog> movieCatalogSets;

@Autowired
private Map<String, MovieCatalog> movieCatalogMaps;
```

Spring将所有的`ApplicationContext`里的MovieCatalog类型注入到数据或者集合参数中。如果希望注入的类有先后顺序，可以使得MovieCatalog实现Ordered接口，或使用@Order或@Priority。

对于Map来说，key值是String类型存储Bean的类名，value里存储类型。

@Autowired 会失败当没有任何匹配上的Bean时，为了不影响程序，我们可以使用`@Autowired(required=false)`。

### @Bean
[Bean Annotation Document](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-java-bean-annotation)
[Bean Scopes](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-factory-scopes)

@Bean 是一个方法级别的注解，注解上支持设置init-method, destroy-method, autowiring和name。

```
@Bean
public TransferService transferService() {
    return new TransferServiceImpl();
}
```

默认的，@Bean注解得到的实例将默认使用公有的的close或shutdown方法作为类销毁前的callback。如果类里面定义了共有的close或者shutdown方法，而你不希望在容器关闭时调用它们，可以使用`@Bean(destroyMethod="")`。在DataSource上一般都使用这种方式。

@Bean 有自己的@Scope，默认是singleton，意味着每一个Spring IoC容器中只有一个实例。但也可以设置为`@Scope("prototype")`, `@Scope("request")`, `@Scope("session")`, `@Scope("application")`, `@Scope("websocket")`等。

{% include image.html url="/static/img/springboot/singleton.png" description="singleton" height="400px" inline="true" %}

> Spring中的单例与设计模式中的单例(一个ClassLoader中只有一个类的实例)是不同的，Spring中的单例是每一个容器每一份实例，Spring中的Bean默认都是singleton。

{% include image.html url="/static/img/springboot/prototype.png" description="prototype" height="400px" inline="true" %}

Scope为prototype的类每次申请这个类的时候都会创建一个新的，即在注入到另外一个类或者调用`getBean()`方法都会创建一个新的类。

> 建议带状态的类(stateful beans)使用prototype范围，无状态的类(stateless beans)使用singleton范围。


### @Profiles
[Profiles Annotation Document](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-definition-profiles-java)

@Profile 往往用来对不同环境配置的区别对待，如Development环境与Production环境需要不同的DataSource配置。

```
@Configuration
public class AppConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean
    @Profile("production")
    public DataSource productionDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

@Profile 支持数组参数（或的关系，意味着多种环境active的情况下都可以激活此配置）`@Profile({"p1", "p2"})`以及`非`语法`@Profile({"p1", "!p2"})`。

@Profile 还支持Default配置`@Profile("default")`, 以为着如果没有环境被激活，默认配置会被激活，如果有环境激活，则默认配置不生效。

### @ExceptionHandler
[ExceptionHandler Annotation Document](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#beans-java-bean-annotation)

@ExceptionHandler 经常出现在Controller类中，如

```
@Controller
public class SimpleController {

    @ExceptionHandler(IOException.class)
    public ResponseEntity<String> handleIOException(IOException ex) {
        // prepare responseEntity
        return responseEntity;
    }
}
```

@ExceptionHandler 可以设置Exception数组， @ExceptionHandler 注解的方法返回值可以是String(对应到view路径)， ModelAndView对象，ResponseEntity，也可以在方法上注解@ResponseBody。

Spring中还有很多注解如 @Service, @Repository, @Component, @Bean, @ConfigurationProperties, @Value, @Inject, @PostConstruct 等，以及model相关的各种注解 @Entity, @Document, @Id, @Column, @ManyToMany 等等，不在此文的讨论范围，按下不表。

## 参考资料
- [Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring MVC Framework and REST](https://www.genuitec.com/spring-frameworkrestcontroller-vs-controller/)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
