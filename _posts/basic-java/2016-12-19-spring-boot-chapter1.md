---
layout: post
comments: false
categories: "Spring-Boot"
date:   2016-12-19 18:30:54
title: Spring Boot Initializer
---

<div id="toc"></div>

Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

初始化一个Spring Boot项目的方式有很多种：

### Spring Initializer网页
打开[http://start.spring.io/](http://start.spring.io/), 生成一个Spring Boot的项目：

{% include image.html url="/static/img/springboot/spring-initializer-site.png" description="图1" height="400px" inline="true" %}

"Generate Project"后，会下载一个zip包到本地，解压后，就是一个简单的spring boot项目了。

进入解压的demo目录下，运行 `./gradlew bootRun` 就可以运行次项目了，默认端口是8080，[http://localhost:8080]。

在运行 `./gradlew bootRun` 时，如果嫌国内下载Java依赖库的速度太慢，可以使用国内阿里的maven库镜像。将

```java
repositories {
	mavenCentral()
}
```

都修改为
```java
repositories {
	mavenLocal()
	maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
	maven { url "https://repo.spring.io/snapshot" }
	maven { url "https://repo.spring.io/milestone" }
}
```

### Spring Initializer idea
电脑上安装有Intellij Idea，或现在下载安装Idea: [https://www.jetbrains.com/idea/](https://www.jetbrains.com/idea/)

File -> New -> Project弹出创建新项目的窗口。
{% include image.html url="/static/img/springboot/spring-initializer-idea.png" description="图2" height="400px" inline="true" %}

选择"Spring Initializer"和Java版本，在以后的配置向导中就可以配置Name, Artifact以及依赖。


### Spring Boot CLI Initializer
安装命令行的Spring Boot ClI: [Installing the Spring Boot CLI](http://docs.spring.io/spring-boot/docs/current/reference/html/getting-started-installing-spring-boot.html#getting-started-installing-the-cli)

而后可以运行下面的命令创建Spring Boot项目。

```
spring init -dweb,data-jpa,h2,thymeleaf --build gradle demo
```


### 参考资料
- [Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/html/)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
