---
layout: post
comments: false
categories: Java基础
date:   2018-12-09 16:30:54
title: Spring boot jar 启动与重启
---

<div id="toc"></div>

Spring Boot打成Jar包后，可以通过`nohup java -jar xxx.jar --spring.profiles.active=test &`的方式启动。但是自动化部署时，将如何进行新的版本的安装和部署呢？

## 纯shell脚本

java -cp /Users/qzhou/.gradle/caches/modules-2/files-2.1/org.jasypt/jasypt/1.9.2/91eee489a389faba9fc57bfee75c87c615c19cd7/jasypt-1.9.2.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input="test123" algorithm=PBEWithMD5AndDES

## Centos自定义系统服务


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
