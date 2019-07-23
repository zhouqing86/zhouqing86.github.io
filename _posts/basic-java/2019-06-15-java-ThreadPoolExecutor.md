---
layout: post
comments: false
categories: Java基础
date:   2019-06-15 10:30:54
title: ThreadPoolExecutor源码阅读 Java 1.8
---

<div id="toc"></div>

> 本文基于Java 1.8.0_121-b13版本

线程池主要是为了解决两个问题：

- 大量异步执行的任务执行时候性能的提升

- 对于线程资源和界限的控制

线程池里线程的创建规则如下：

- 调用ThreadPoolExecutor的execute提交线程，首先检查CorePool，如果CorePool内的线程小于CorePoolSize，新创建线程执行任务

- 如果当前CorePool内的线程大于等于CorePoolSize，那么将线程加入到workQueue，其类型为BlockingQueue

- 如果不能加入workQueue，在小于MaxPoolSize的情况下创建线程执行任务

- 如果线程数大于等于MaxPoolSize，那么执行拒绝策略

## 一些常量和成员变量


## HashMap中定义的静态类


## HashMap中定义的重要方法


## 使用



## 参考文章

- [深入理解java线程池—ThreadPoolExecutor](https://www.jianshu.com/p/ade771d2c9c0)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
