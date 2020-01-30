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

线程池的状态:

```
private static final int COUNT_BITS = Integer.SIZE - 3;

//Accept new tasks and process queued tasks
private static final int RUNNING    = -1 << COUNT_BITS;

//Don't accept new tasks, but process queued tasks
private static final int SHUTDOWN   =  0 << COUNT_BITS;

//Don't accept new tasks, don't process queued tasks,and interrupt in-progress tasks
private static final int STOP       =  1 << COUNT_BITS;

//All tasks have terminated, workerCount is zero,the thread transitioning to state TIDYING will run the terminated() hook method
private static final int TIDYING    =  2 << COUNT_BITS;

terminated() has completed
private static final int TERMINATED =  3 << COUNT_BITS;
```

这里需要复习下源码、补码、反码：

- 正数的原码、反码、补码都是一样的

- 在计算机底层，所有的数都是用补码来表示的

- 对于负数，反码为原码的除符号位以外全部取反，而补码为反码+1

  ```
  -1的源码为10000000 00000000 00000000 00000001
  -1的反码为11111111 11111111 11111111 11111110
  -1的补码为11111111 11111111 11111111 11111111
  ```

上面线程池的状态就是用整数的前三位表示。

而对于线程的容量，用后面的29位表示：

```
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

其他重要的变量：

```
private final BlockingQueue<Runnable> workQueue;
```

## ThreadPoolExecutor相关接口

### ExecutorService



## ThreadPoolExecutor中定义的静态类


## HashMap中定义的重要方法


## 使用

### Executors




## 参考文章

- [深入理解java线程池—ThreadPoolExecutor](https://www.jianshu.com/p/ade771d2c9c0)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
