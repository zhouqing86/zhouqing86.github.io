---
layout: post
comments: false
categories: Java基础
date:   2019-06-15 10:30:54
title: ThreadPoolExecutor源码阅读 Java 1.8
---

<div id="toc"></div>

> 本文基于Java 1.8.0_121-b13版本

AbstractQueuedSynchronizer类是用来构建锁或者其他同步组件的基础框架类。

因为此文较长，后续的针对成员变量和逻辑的分析需要花时间阅读。大部分人应该只是想大概了解下AbstractQueuedSynchronizer的使用，可以直接跳到使用章节。

AbstractQueuedSynchronizer被许多并发工具类类所使用或继承，如`ReentrantLock`,`ReentrantReadWriteLock`,`CountDownLatch`, `Semaphore`和`ThreadPoolExecutor.Worker`。

## 什么是AbstractQueuedSynchronizer

这里先引用其源码类里的一段英文说明:

```
Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state.
```

`AbstractQueuedSynchronizer`是一个`抽象类`，但是英文说明称其为`框架`，瞬间高大上了。这个类为`依赖先进先出队列`的`阻塞锁`和`相关同步器`提供了基础实现。对于大部分同步器，只是依赖于对于一个整型`state`的同步修改。

这里其实可以得到很多的信息：

- `AbstractQueuedSynchronizer`中使用了先进先出队列

- `AbstractQueuedSynchronizer`有一个整型的`state`




## 一些常量和成员变量



## HashMap中定义的静态类


## HashMap中定义的重要方法


## 使用

定义子类来继承`AbstractQueuedSynchronizer`，覆盖如下方法：

- tryAcquire

- tryRelease

- tryAcquireShared

- tryReleaseShared

- isHeldExclusively

## 参考文章

- [AbstractQueuedSynchronizer源码解读](https://www.cnblogs.com/micrari/p/6937995.html)

- [Doug Lea的AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
