---
layout: post
comments: false
categories: Java基础
date:   2016-12-11 18:30:54
title: Guava的使用
---

<div id="toc"></div>

Guava是Google的Java开发人员经常使用的库。

Guava的集合操作(Ordering、MultiSet、BiMap、MutliMap、Table、Iterators)可参考[Guava集合操作](https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/GuavaTest.java)

Guava的IO操作(Files.write、Files.readLines、Files.copy)可参考[Guava IO](https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/GuavaFileIOTest.java)

Guava的反射操作(TypeToken、Modifier、Invokable、Reflection)可参考[Guava Reflection](https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/GuavaReflectionTest.java)

Guava的Cache(CacheBuilder.newBuilder()、CacheLoader)可参考[Guava Cache](https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/GuavaCacheTest.java)

Guava的并行操作参考[Guava Parallel](https://github.com/zhouqing86/three-java/blob/master/src/test/java/medium/GuavaParallelProgrammingTest.java)

## 参考资料
- [Google Guava官方教程（中文版](http://ifeve.com/google-guava/)
- [Guava Wiki](https://github.com/google/guava/wiki)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
