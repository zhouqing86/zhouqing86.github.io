---
layout: post
comments: false
categories: "测试JS"
date:   2017-4-8 18:30:54
title: JS测试之Mock Library
---

<div id="toc"></div>

JS中有很多用来做mock的库: sinon, proxyquire, rewire, mockery。这里对这些库做一一介绍。

<br/>

## 准备
### 待测程序

<br/>

### 基本概念

#### stub, mock, spy

为了测试目的而定义替代"真"对象的任何"假"对象可以被称为 `Double` 。stub/mock/spy都属于 `Double` 。

它们之间的区别：

- stub 可以看做一个假的实现（假在其有行为，但其行为并不是真正经过一系列代码处理的行为，其可能是忽略处理逻辑直接返回结果），在Java中其是一个假的对象（对象中的方法直接返回测试需要的值；在JS中其可以是假的对象或者方法，假的方法是可以直接假定接收什么样的参数以及返回什么样的值。

- mock 不仅仅为假的方法提供假的行为，也可以在测试程序中添加期望来测试假的方法被调用。

- 所以stub和mock运用场景的一个大的区别： A方法时调用了B，只测试A方法的行为可以使用stub; 如果希望测试到A方法在调用时也调用了，需要使用mock。

- spy是仅仅提供假的方法，但并不提供假的实现。如可以基于spy来测试某个方法调用的次数。

## sinon

<br/>

## proxyquire

<br/>

## rewire

<br/>

## mockery

<br/>

## 参考资料

- [Comparing mockery vs. proxyquire vs. rewire vs. sinon](https://npmcompare.com/compare/mockery,proxyquire,rewire,sinon)

- [mock, stub, spy](http://stackoverflow.com/questions/24413184/can-someone-explain-the-difference-between-mock-stub-and-spy-in-spock-framewor)

- [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
