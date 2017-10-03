---
layout: post
comments: false
categories: "dubbo"
date:   2017-10-01 00:00:54
title: Dubbo源码学习（三）之准备Debug
---

<div id="toc"></div>

在[Dubbo源码学习（一）之Dubbo Demo](/2017/09/30/dubbo-demo/)中我们了解了dubbo-demo。
在[Dubbo源码学习（二）之改造dubbo-demo](/2017/10/01/dubbo-zookeeper/)我们尝试对demo做一点改造，也部署了dubbo-admin。

## Log
个人习惯是先从程序的log开始探索框架的基本流程。

### dubbo-provider
demo中dubbo-provider默认配置log等级为info，运行后我们可以看到log如：

```
support.ClassPathXmlApplicationContext: Refreshing org.springframework.context.support.ClassPathXmlApplicationContext
xml.XmlBeanDefinitionReader: Loading XML bean definitions from class path resource [META-INF/spring/dubbo-demo-provider.xml]
logger.LoggerFactory: using logger: com.alibaba.dubbo.common.logger.log4j.Log4jLoggerAdapter
config.AbstractConfig:  [DUBBO] The service ready on spring started. service: com.alibaba.dubbo.demo.DemoService, dubbo version: 2.0.0, current host: 127.0.0.1
config.AbstractConfig:  [DUBBO] Export dubbo service com.alibaba.dubbo.demo.DemoService to local registry, dubbo version: 2.0.0, current host: 127.0.0.1
config.AbstractConfig:  [DUBBO] Export dubbo service com.alibaba.dubbo.demo.DemoService to url dubbo://192.168.0.103:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=86462&side=provider&timestamp=1506999958995, dubbo version: 2.0.0, current host: 127.0.0.1
config.AbstractConfig:  [DUBBO] Register dubbo service com.alibaba.dubbo.demo.DemoService url dubbo://192.168.0.103:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=86462&side=provider&timestamp=1506999958995 to registry registry://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=86462&registry=multicast&timestamp=1506999938958, dubbo version: 2.0.0, current host: 127.0.0.1
transport.AbstractServer:  [DUBBO] Start NettyServer bind /0.0.0.0:20880, export /192.168.0.103:20880, dubbo version: 2.0.0, current host: 127.0.0.1
multicast.MulticastRegistry:  [DUBBO] Load registry store file:....
multicast.MulticastRegistry:  [DUBBO] Register:....
multicast.MulticastRegistry:  [DUBBO] Send broadcast message: register....
DubboMulticastRegistryReceiver  INFO multicast.MulticastRegistry:  [DUBBO] Receive multicast message: register....
multicast.MulticastRegistry:  [DUBBO] Subscribe: provider://192.168.0.103:20880/com.alibaba.dubbo.demo.DemoService....
DubboMulticastRegistryReceiver  INFO multicast.MulticastRegistry:  [DUBBO] Receive multicast message: subscribe provider://192.168.0.103:20880/com.alibaba.dubbo.demo.DemoService....
multicast.MulticastRegistry:  [DUBBO] Notify urls for subscribe url provider://192.168.0.103:20880/com.alibaba.dubbo.demo.DemoService....
```

我们先粗略来看下牵涉到的类，从log里获取粗略理解：
- 首先ClassPathXmlApplicationContext将读取META-INF/spring/dubbo-demo-provider.xml文件。
- AbstractConfig来获取dubbo相关配置处理dubbo service。
- AbstractServer来启动一个NettyServer，绑定端口。
- MulticastRegistry发送多播消息
- DubboMulticastRegistryReceiver接受多播消息

而后我们根据log具体信息来寻找代码所在地，从第一条AbstractConfig相关信息开始:
- 关于第一步的疑问，ClassPathXmlApplicationContext是Spring提供的类，其加载配置后如何识别dubbo:application, dubbo:registry, dubbo:protocol, dubbo:service节点呢。
其其实使用了[扩展spring schema文件](https://segmentfault.com/a/1190000007047168)的方式在dubbo-config-spring中定义了一套Spring的Schema扩展。

	在DubboNamespaceHandler类中可以看到，"service"对应的解析器为`new DubboBeanDefinitionParser(ServiceBean.class, true)`:

	```
	public void init() {
	  registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
	  registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
	  registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
	  registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
	  registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
	  registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
	  registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
	  registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
	  registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
	  registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
	}
	```

- com.alibaba.dubbo.config.spring.ServiceBean里有onApplicationEvent。ServiceBean继承自ServiceConfig并实现了ApplicationListener接口，如果在上下文中部署一个实现了ApplicationListener接口的bean,那么每当在一个ApplicationEvent发布到 ApplicationContext时，这个bean得到通知。其实这就是标准的Obeserver设计模式。

- onApplicationEvent方法中调用了位于ServiceConfig子类的export()方法。将Bean对象转换URL格式，所有 Bean属性转成URL的参数。


设置`log4j.properties`里的logger等级`log4j.rootLogger=debug, stdout`。而后运行Provider来查看更详细的log。


### dubbo-consumer

这里就不列出consumer端的log信息了。


## Debug
### dubbo-demo之deubg

基于dubbo源码，以及上面log信息中获得的一些代码的位置，我们就可以开始我们的debug之旅了。

### dubbo-admin之debug
基于[Dubbo源码学习（二）之改造dubbo-demo](/2017/10/01/dubbo-zookeeper/)中搭建的环境，我们回到dubbo的源码库中：
- 源码库中执行`mvn tomcat:run`。
- IDEA中增加remote debug的配置，端口配置为8000, (IDEA上的remote debug配起来非常方便，只需要修改端口即可，会生成如`-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000`)。
- 在Providers类的disable方法的第一行上下断点。
- 在页面上禁用demoService的Provider，将发送GET请求如`http://localhost:8888/governance/providers/2/disable`。
- 程序将停在断点处


## 参考资料
- [Webx框架指南](http://www.360doc.com/content/15/0323/10/22409769_457343362.shtml)
- [扩展spring schema文件](https://segmentfault.com/a/1190000007047168)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
