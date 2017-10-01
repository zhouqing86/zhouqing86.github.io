---
layout: post
comments: false
categories: "dubbo"
date:   2017-09-30 00:00:54
title: Dubbo源码学习（一）之Dubbo Demo
---

<div id="toc"></div>

> 一个好消息，dubbo正式得到官方维护与支持。

Dubbo是一个分布式服务框架，其也是一个RPC框架，我们先从RPC入手来接触Dubbo的源码。本文先通过运行dubbo提供的demo对RPC以及Provider以及Consumer有一个直观的认识。

## 构建源码

我使用的是MacOS环境，JAVA版本1.8， Maven的版本是3.5.0。

- 下载源码

	```
	git clone https://github.com/alibaba/dubbo
	```

- 构建源码，生成IDEA支持的项目文件

	```
	mvn clean install -Dmaven.test.skip
	mvn idea:idea
	```

- IDEA打开生成的dubbo-parent.ipr文件，就可以看到整个Dubbo的工程了。

## 运行dubbo-demo

### 运行Provider

需要运行的Provider的路径为`dubbo-demo/dubbo-demo-provider/src/main/java/com/alibaba/dubbo/demo/provider/Provider.jav`，其源代码为：

```
public class Provider {

    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();

        System.in.read(); // 按任意键退出
    }

}
```

> 运行时，可能报Can't assign requested address，需要在IDEA上更新Provider的运行配置，VM选项上增加-Djava.net.preferIPv4Stack=true

直接在IDEA上运行后，可以看到log如:

{% include image.html url="/static/img/java/dubbo-provider.png" description="Dubbo Demo Provider" width="800px" inline="true" %}

### 运行Consumer

需要运行的Consumer的路径为`dubbo/dubbo-demo/dubbo-demo-consumer/src/main/java/com/alibaba/dubbo/demo/consumer/Consumer.java`， 其源代码为:

```
public class Consumer {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-consumer.xml"});
        context.start();

        DemoService demoService = (DemoService) context.getBean("demoService"); // 获取远程服务代理
        String hello = demoService.sayHello("world"); // 执行远程方法

        System.out.println(hello); // 显示调用结果
    }
}
```

> 运行时，可能报Can't assign requested address，需要在IDEA上更新Consumer的运行配置，VM选项上增加-Djava.net.preferIPv4Stack=true

直接在IDEA上运行，可以看到其log如:

{% include image.html url="/static/img/java/dubbo-consumer.png" description="Dubbo Demo Consumer" width="800px" inline="true" %}

而在Provider运行的terminal上，可以看到的log包含:

```
[22:06:03] Hello world, request from consumer: /192.168.0.103:57558
```

### Demo运行解释

实际上上面的步骤中已经完成了一次Consumer对Provider的RPC调用。那么如何从代码理解这所谓的RPC调用呢，我们首先需要知道的是DemoService是一个接口，其路径为`dubbo/dubbo-demo/dubbo-demo-api/src/main/java/com/alibaba/dubbo/demo/DemoService.java`， 其源码为:

```
public interface DemoService {

    String sayHello(String name);

}
```

而DemoService的实现DemoServiceImpl.java与Provider.java在同一目录下，路径为`dubbo/dubbo-demo/dubbo-demo-provider/src/main/java/com/alibaba/dubbo/demo/provider/DemoServiceImpl.java`， 其源码为:

```
public class DemoServiceImpl implements DemoService {

    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response form provider: " + RpcContext.getContext().getLocalAddress();
    }

}
```

这时我们就大概明白这个DEMO是干了什么事：启动Consumer主线程时需要调用Provider中的DemoServiceImpl的sayHello方法并获取其返回值。

### Demo中的核心配置

如果只是上面贴出的Provider、Consumer、DemoService和DemoServiceImpl代码，是完不成这个RPC调用的。Dubbo在默默的为这个RPC调用做了很多的工作，我们首先来看下Provider和Consumer的配置文件:

Provider的"META-INF/spring/dubbo-demo-provider.xml":

```
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="demo-provider"/>

    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234"/>

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- 和本地bean一样实现服务 -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```

Consumer的"META-INF/spring/dubbo-demo-consumer.xml":

```
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="demo-consumer"/>

    <!-- 使用multicast广播注册中心暴露发现服务地址 -->
    <dubbo:registry address="multicast://224.5.6.7:1234"/>

    <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
    <dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>

</beans>
```

这两个文件都是通过ClassPathXmlApplicationContext来在Provider和Consumer中读取解析的。熟悉Spring的话应该对ClassPathXmlApplicationContext不会陌生，因为Spring MVC在加载application.xml时用的也是这个类。

在加载完配置，运行`context.start()`后，dubbo就介入了: 服务端打开端口，客户端发起与服务端的连接等等。并使得在Consumer端调用`demoService.sayHello("world")`进行远程调用。

在两者的配置中都有 `<dubbo:registry address="multicast://224.5.6.7:1234"/>` 这一条，其表示配置了一个注册中心，组播受网络结构限制，只适合小规模应用或开发阶段使用。组播地址段: 224.0.0.0 - 239.255.255.255。

Provider与Consumer配置了相同的组播地址后，就可以相互发现。

## Dubbo架构

运行了demo后，我们再来看看Dubbo目前的架构:

{% include image.html url="/static/img/java/dubbo-architecture.jpg" description="Dubbo Architecture" width="400px" inline="true" %}

Provider，Consumer以及注册中心都能在demo中都接触到了。

## 参考资料

- [Multicast 注册中心](https://dubbo.gitbooks.io/dubbo-user-book/references/registry/multicast.html)

- [组播协议原理讲解](http://blog.csdn.net/liu251890347/article/details/39211685)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
