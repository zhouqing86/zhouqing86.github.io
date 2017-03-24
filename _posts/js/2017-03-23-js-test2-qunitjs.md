---
layout: post
comments: false
categories: "测试JS"
date:   2017-03-23 18:30:54
title: JS测试之QUnit
---

<div id="toc"></div>

如果对[JS测试之自写断言](/2017/03/23/js-test1/)有印象的话，我们自己定义assertEqual函数来对add函数做测试。其打印的结果在浏览器控制台中，非常不友好。此时的你有想造轮子的冲动么，让我们先了解下一个古老的轮子QUnit。

[QUnit](http://qunitjs.com/)是一个古老的JS测试框架，其提供了类似assertEqual，assertTrue等基本的断言函数的实现，并且能为我们提供美观的结果页面。

### QUnit例子

这里写了一个QUnit的例子，首先创建math.js:

```
function add(a, b) {
  return a + b;
}
```

创建执行测试的index.html文件:

```
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>qunit Test examples</title>
    <link rel="stylesheet" href="https://code.jquery.com/qunit/qunit-2.2.1.css">
    <script src="https://code.jquery.com/qunit/qunit-2.2.1.js"></script>
    <script src="math.js"></script>
    <script>
    QUnit.test('test add function', function(assert) {
      assert.equal(add(2,3), 4);
      assert.equal(add(6,7), 13);
      assert.equal(add(7,7), 10);
    });
    </script>
</head>
<body>
<div id="qunit"></div>
</body>
</html>

```

- 引入QUnit的依赖js和css

- 使用QUnit中的test方法， 在其回调函数中来进行测试

- 在html body中增加`<div id="qunit"></div>`

在浏览器中显示的结果如：

{% include image.html url="/static/img/js/qunit-test-example.png" description="QUnit测试结果" height="400px" inline="true" %}

### QUnit源码

在[qunit.js](https://github.com/qunitjs/qunit/blob/master/src/qunit.js)中:

```
import "./core";
import "../runner/runner";
import "../reporter/reporter";
```

core.js中定义了QUnit对象(对象拥有test, todo, skip, assert等方法)，runner中调用了QUnit.testStart， reporter负责打印测试结果。

### 参考资料

[QUnit源码](https://github.com/qunitjs/qunit)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
