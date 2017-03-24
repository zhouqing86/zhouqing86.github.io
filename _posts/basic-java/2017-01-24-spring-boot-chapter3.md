---
layout: post
comments: false
categories: "Spring-Boot"
date:   2017-01-23 18:30:54
title: Spring Boot Debug and Log
---

<div id="toc"></div>

当需要在Spring Boot中定位问题时，主要的两种方式是远程Debug和根据Log查找线索。

## 远程Debug
### 设置bootRun任务中的jvmArgs属性

在build.gradle中

```
bootRun {
    jvmArgs "-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005"
}
```

而后在Intellij Idea中配置Debug的配置，Run -> Edit Configurations而后增加一条Remote Debug的配置。

{% include image.html url="/static/img/springboot/idea-spring-boot-debug.png" description="Idea Remote Debug配置" width="920px" inline="true" %}

在相应代码位置增加断点就可以进行调试了。对于Spring程序，基本所有的Request都会被DisptachServlet中doDispatch方法处理，可以在此函数中下断点。如下是访问本地Spring Boot程序 http://localhost:8080/users/abc 时断点停在了Controller中。

{% include image.html url="/static/img/springboot/idea-spring-boot-debug-2.png" description="Idea Remote Debug例子" width="920px" inline="true" %}

### 命令行

```
./gradlew bootRun --debug-jvm
```

而后在Idea中增加Remote Debug的配置，并进行debug。

## Log
### application.properties配置logger

配置`logging.level.root=debug` 或 `debug=true`，启动Spring Boot程序时可以看到Spring Boot以及Spring框架的Debug级别的Log。

或者可以配置某个Package的Log级别，如`logging.level.root.cn.xxx=debug`。

如果想让console打印的log有颜色上的区别，可以设置`spring.output.ansi.enabled=always`。

如果要将Log打印到某个文件中，设置`logging.path=/tmp`, 启动程序后会发现在/tmp目录下产生了sprint.log。设置`logging.file=test.log`不起作用，可以参考[Spring Boot - no log file written (logging.file is not respected)](http://stackoverflow.com/questions/38527175/spring-boot-no-log-file-written-logging-file-is-not-respected)


## 参考资料
- [Spring Boot Reference Guide](http://docs.spring.io/spring-boot/docs/current/reference/html/)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
