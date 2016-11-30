---
layout: post
comments: false
categories: Java基础
date:   2016-11-29 10:30:54
title: Java 缺憾
---

<div id="toc"></div>


用了一段时间Java，想总结下Java用起来不那么趁手的地方，请直接下看文。

### 垃圾回收器问题
垃圾回收器有时会清除整个Java进程，所以尽可能避免使用内存缓存，除非你的内存缓存非常小且稳定。

你的Android程序可能会随机的挂住几秒，当你找不出原因的时候，很可能垃圾回收器就是那个罪魁祸首。

### 糟糕的异常机制
无处不在的Throw Exceptions。


### Java类型擦除
当你用泛型时，会发现List<Long>和List<String>对Java函数参数来说是同一个东西。当你在同一个类中定义如下两个函数:

```java
public void func(List<Long> listOfLongs) { }
public void func(List<String> listOfStrings) { }
```

编译错误！

### LinkedList不是List
你不相信，看看这个代码在你的IDE会有什么错：

```java
List<List<Person>> list = new LinkedList<LinkedList<Person>>();
```

### 静态成员和方法不可覆盖不可派生
静态变量和静态方法是完全特定属于某个类的。继承自父类的类访问到的静态变量也是它父亲的。你可以申明自己的静态变量覆盖父类的静态变量，但是如果你用的是继承自父类的静态方法获取静态变量，你最终得到的也是父类的静态变量。

### 没有import as
如果在同一个文件中import两个名字相同的类时，就必须在程序中加上包全路径来做区分。


### 初始化一个list这么麻烦
如需要创建一个list为`[1, 1, 2, 3, 5, 8, 13]`， 符合直觉的方式应时`LinkedList<Integer> l = new LinkedList({1,1,2,3,5,8,13})`。但Java创建一个list要比这复杂很多。

### Arrays.asList返回的结果可能不是你想要的ArrayList

String [] array = {"a", "b", "c"};
ArrayList<String> lst = Arrays.asList(array);

编译器报错了，Arrays.asList将返回一个java.util.Arrays$ArrayList对象，没有add方法，这表示它不能动态扩展内部存储的数组。

### 无法运算符重载
运算符重载对于数值算法的编程来说有很大意义，读这行试试看

```java
a.add(b.minus(h).multiply(d.divide(i.add(j))).add(e)).add(c.‌​multiply(f.minus(g.d‌​ivide(j))))
```

Java语言被设计舍弃重载是为了避免出现C++中重载带来的复杂性和各种问题。但Scala用了一种其他的方式来解决这个问题：

```scala
case class A(value: Int) {
   def +(other: A) = new A(value + other.value)
}

scala> new A(1) + new A(3)                                                           
res0: A = A(4)
```

### Java没有复合类型
拿HashMap作为例子，内部实现为HashMap.Entry对象数组。从HashMap中查找数据将需要讲过两层间接层：

- 数据中的HashMap.Entry对象指针找到HashMap.Entry对象
- HashMap.Entry对象中找到key或者value的指针
- 根据key或value的指针找到对象




### 参考资料
[Why Java Sucks](http://tech.jonathangardner.net/wiki/Why_Java_Sucks)
[Criticism of Java](https://en.wikipedia.org/wiki/Criticism_of_Java)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
