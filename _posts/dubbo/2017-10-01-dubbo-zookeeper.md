---
layout: post
comments: false
categories: "dubbo"
date:   2017-10-01 00:00:54
title: Dubbo源码学习（二）之改造dubbo-demo
---

<div id="toc"></div>

在[Dubbo源码学习（一）之Dubbo Demo](/2017/09/30/dubbo-demo/)中我们了解了dubbo-demo，现在我们尝试对其做一点改造：

- 改造成gradle构建

- 引入Spring Boot，Consumer提供一个/hello节点从demoService里获取结果

- 将注册中心从组播转为ZooKeeper

- 引入dubbo-admin

## Gradle构建dubbo-demo

dubbo-demo的例子是maven构建的，我们将其改造成gradle的项目。参考https://github.com/zhouqing86/learn-dubbo。

```
buildscript {
	ext {
		springBootVersion = '2.0.0.BUILD-SNAPSHOT'
	}
	repositories {
//		mavenCentral()
		mavenLocal()
		maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
		maven { url "https://repo.spring.io/snapshot" }
		maven { url "https://repo.spring.io/milestone" }
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
}

allprojects {
    apply plugin: 'idea'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'
}

subprojects {
    apply plugin: 'java'

    // In this section you declare where to find the dependencies of your project
    repositories {
        mavenCentral()
	      mavenLocal()
	      maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
    }

    ext {
      slf4jVersion = '1.7.24'
      junitVersion = '4.12'
      springBootVersion = '1.5.7.RELEASE'
      dubboVersion = '2.5.5'
      mysqlConnectorVersion = '5.1.44'
  	}

    dependencies {
        // The production code uses the SLF4J logging API at compile time
        compile "org.slf4j:slf4j-api:${slf4jVersion}"
        compile("org.springframework.boot:spring-boot-starter-parent:${springBootVersion}")
        compile("org.springframework.boot:spring-boot-starter-web:${springBootVersion}")
        compile("org.springframework.boot:spring-boot-starter-data-jpa:${springBootVersion}")
        compile("org.springframework.boot:spring-boot-starter-jdbc:${springBootVersion}")
        compile("com.alibaba:dubbo:${dubboVersion}")
        runtime("mysql:mysql-connector-java:${mysqlConnectorVersion}")
        runtime('com.h2database:h2:1.4.196')
        testCompile "junit:junit:${junitVersion}"
    }
}

project(':dubbo-demo-consumer') {
    dependencies {
        compile project(':dubbo-demo-api')
    }
}

project(':dubbo-demo-provider') {
    dependencies {
        compile project(':dubbo-demo-api')
    }
}
```

## Consumer创建/hello节点
目的是将Consumer改造成一个API，定义ConsumerApplication如:

```
@SpringBootApplication
@ImportResource(locations = {"classpath:META-INF/spring/dubbo-demo-consumer.xml"})
public class ConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

定义Controller如:

```
@RestController
public class TestController {

    @Autowired
    private DemoService demoService;

    @RequestMapping("/hello")
    public String hello() {
        String hello = demoService.sayHello("world");
        return hello;
    }

}
```

而后访问 http://localhost:8080/hello 就能获取到 `Hello world, response form provider: 192.168.0.103:20880`。

即整个调用是从 浏览器 -> Consumer的Hello节点 -> Provider的DemoServiceImpl的过程。

## 使用ZooKeeper来作为注册服务器

使用docker-compose来启动zookeeper集群，配置文件docker-compose.yml如下：

```
version: '2'
services:
    zoo1:
        image: zookeeper
        restart: always
        ports:
            - 2181:2181
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

    zoo2:
        image: zookeeper
        restart: always
        ports:
            - 2182:2181
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

    zoo3:
        image: zookeeper
        restart: always
        ports:
            - 2183:2181
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
```

启动`docker-compose up`可以启动zookeeper集群（三个zookeeper节点)。

启动后可以通过 `docker run -it --rm zookeeper zkCli.sh -server 192.168.0.103:2181` 连接到zookeeper集群，如果本地有zkCli.sh的话，可以直接在相应目录运行`zkCli.sh -server 127.0.0.1:2181`来链接到zookeeper集群。其中`192.168.0.103`为本机IP，而后可以通过如下命令测试zookeeper是否正常运行:

```
ls /
create /zktest mydata
ls /zktest
get /zktest
```

而后，在程序中修改`dubbo:registry`的配置为`<dubbo:registry address="zookeeper://127.0.0.1:2181?client=curator"/>`，在build.gradle中引入zookeeper和curator依赖:

```
compile("org.apache.zookeeper:zookeeper:3.4.10")
compile("org.apache.curator:curator-framework:4.0.0")
```

下载依赖后重新运行程序，访问http://localhost:8080/hello，返回结果正常。Bingo!

但注册中心的配置并未到此结束，为了高可靠性，需要配置集群，如:

```
<dubbo:registry protocol="zookeeper" address="127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183" client="curator" />
```

## 启动admin

首先需要获取dubbo-admin的war包，如果下载了dubbo的源码，只需要运行`mvn clean install -Dmaven.test.skip`。而后在dubbo-admin/target下就可以看到相应的war包。

这里使用docker tomcat来运行这个war包，但是注意因为要修改war包里的dubbo.properties配置，这里解压了war包，修改了dubbo.properties并通过docker-compose的volumes映射进容器里，在上面docker-compose.yml中增加tomcat:

```
tomcat:
    image: dordoka/tomcat
    restart: always
    ports:
        - 8888:8080
    volumes:
        - ./dubbo-admin-war:/opt/tomcat/webapps/ROOT

```

注意这里volums下的配置`./dubbo-admin-war:/opt/tomcat/webapps/ROOT`，是将本机当前目录dubbo-admin-war映射到容器/opt/tomcat/webapps/`ROOT`。因为是ROOT目录就可以通过URL的根路径访问了，dubbo-admin的很多URL都没有考虑contextPath，所以最好配置根路径访问。

修改的`WEB-INF/dubbo.properties`为:

```
dubbo.registry.address=zookeeper://192.168.0.103:2181
dubbo.admin.root.password=root
dubbo.admin.guest.password=guest
```

这里重点是修改`192.168.0.103`，是为了让admin能够连接上zookeeper。

而后访问http://localhost:8888/，使用用户名root密码root登陆即可。在系统管理菜单下系统状态可以看到与zookeeper的连接状态。

在搭建Admin后发现的两个问题：

- 管理控制台找不到服务

> 重新启动Provider和Consumer就可以在管理控制台上看到了。

- 禁用后Consumer仍然能访问Provider

> 暂时没有找到解决办法，可能是一个Bug


## 参考资料

- 博文中的代码请参考https://github.com/zhouqing86/learn-dubbo

- [Multi-project Builds](https://docs.gradle.org/current/userguide/multi_project_builds.html)

- [zookeeper docker](https://hub.docker.com/_/zookeeper/)

- [zookeeper 注册中心](https://dubbo.gitbooks.io/dubbo-user-book/references/registry/zookeeper.html)

- [tomcat docker](https://hub.docker.com/_/tomcat/)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
