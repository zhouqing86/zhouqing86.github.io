---
layout: post
comments: false
categories: "测试JS"
date:   2017-4-7 18:30:54
title: JS测试之测试断言
---

<div id="toc"></div>

断言一词来自逻辑学，在逻辑学中，`断言`是`断定一个特定前提为真的陈述`，在软件测试中也是类似的含义。测试中断言语句的一般形式为`assert 表达式`，其中的`表达式`就是逻辑学中的`陈述`，表达式的值为真(true)的时候该断言才能通过，否则就断言失败。

在[/2017/03/23/js-test1/](JS测试之自写断言)中，介绍了如何自定义断言。

JS的生态系统中有很多断言库如

- Node.js自带的[assert模块](https://nodejs.org/api/assert.html)，

- C语言风格的[better-assert库](https://github.com/tj/better-assert)

- BDD `should`风格的[should.js断言库](https://github.com/shouldjs/should.js)

- BDD `expect`风格的[expect.js工具套件](https://github.com/Automattic/expect.js)

- BDD `expect`风格的[unexpected.js工具套件](http://unexpected.js.org/)

- BDD/TDD测试风格都支持的[Chai断言框架](https://github.com/chaijs/chai)

断言框架很多，好消息是上述的断言框架都能够引入到[mocha测试框架](http://mochajs.org/)上使用。

下面我们就来看看这些断言框架的测试都是什么风格。

### Node.js assert模块

#### 介绍

Node.js的assert模块下定义了一系列assert相关函数，如assert.ok()，assert.equal, assert.fail,  assert.deepEqual, assert.deepStrictEqual, assert.doesNotThrow, assert.ifError, assert.notDeepEqual, assert.notDeepStrictEqual, assert.notEqual, assert.notStrictEqual, assert.strictEqual, assert.throws等方法。

当断言不为真时，这些函数都会抛出js异常。 如定义index.js文件如:

```
const assert = require('assert');
assert.strictEqual(1, '1');
```

运行`node index.js`会抛出异常:

```
assert.js:90
  throw new assert.AssertionError({
  ^
AssertionError: 1 === '1'
......
```

#### mocha中使用

```
const assert = require('assert');

describe('nodejs assert module', () => {
  it('strict equal assertion will fail', () => {
    assert.strictEqual(1, '1');
  });

  it('throw error assertion will pass', () => {
    assert.throws(
      () => {
        throw new Error('Wrong value');
      },
      /value/
    );
  });
});
```

### better-assert
#### 介绍

`npm install -D better-assert` 来安装。

C语言风格的断言如下index.js:

```
const assert = require('better-assert');

assert(1 == '1');
assert(1 === '1');
```

运行`node index.js`，将抛出异常:

```
throw err;
  ^
AssertionError: 1 === '1'
......
```

#### mocha中使用

```
const assert = require('better-assert');

describe('better-assert', () => {
  it('will assert fail on the strict equal expression', () => {
    assert(1 === '1');
  });
});
```

### shoud.js
#### 介绍

`npm install -D should` 来安装。

此库的目的是`The main goals of this library are to be expressive and to be helpful.`. 这个库使得所写的测试语义化，其功能非常强大。

默认的，当你调用`require('should')`时，其将扩展`Object.prototype`, 此时，就可以在继承自Object(JS中所有对象的基类)的对象上使用should方法，如

```
(1).should.be.exactly(1).and.be.a.Number();
({ name : 'paul' }).should.have.property('name').which.is.a.String()
"abc".should.be.eql('abc');
false.should.not.be.ok();
(new Promise(function(resolve, reject) { resolve(10); })).should.be.a.Promise()
```

如果不希望扩展`Object.prototype`，可以`const should = require('should/as-function')`, 而后使用

```
should(1).be.exactly(1);
shoud(true).be.ok();
```

创建index.js如：

```
const should = require('should');
(1).should.be.exactly('1');
```

`node index.js`将抛出异常:

```
throw new AssertionError(params);
      ^
AssertionError: expected 1 to be '1'
.....
```

> should.js还支持在浏览器上运行，只需要引入浏览器上使用的should.js或自己构建最新的should.js，并使用script tag将其引入html中， 就可以在window对象中获取should方法。


should.js定义了许多有用的函数，参考[shouldjs文档](http://shouldjs.github.io/)

- be,and,a,have,is,it 这些只是简单的链或者叫别名，反正都是 be 的别名而已，文档里有说明的

#### 汉化should.js?

```
var should = require('should');
var Assertion = should.Assertion;

should.extend('应该', Object.prototype); // should 别名

['是', '个', '并且'].forEach(function(name) { // 属性别名
    Assertion.addChain(name);
});

// 方法别名
Assertion.alias('not', '不');
Assertion.alias('equal', '等于');
Assertion.alias('Number', '数字');
Assertion.alias('String', '字符串');

// 测试例子
(5).应该.等于(5).并且.是.个.数字();
'5'.应该.等于('5').并且.是.个.字符串();
(5).应该.等于(5).并且.不.是.个.数字();

console.log('完成!');
```


#### mocha中使用

### expect.js
#### 介绍


#### mocha中使用

### unexpected.js
#### 介绍


#### mocha中使用

### Chai
#### 介绍


#### mocha中使用

### 参考资料

- [https://zh.wikipedia.org/wiki/%E9%80%BB%E8%BE%91%E6%96%AD%E8%A8%80](https://zh.wikipedia.org/wiki/%E9%80%BB%E8%BE%91%E6%96%AD%E8%A8%80)

- [JavaScript Object Prototypes](https://www.w3schools.com/js/js_object_prototypes.asp)

- [几款前端测试断言库(Assertions lib)的选型总结](http://blog.lvscar.info/post/tech/assertions_lib/)

- [Should.js 的中文打开方式](http://www.tuicool.com/articles/AR7fa2v)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
