---
layout: post
comments: false
categories: "测试JS"
date:   2017-03-26 18:30:54
title: JS测试之Mocha
---

<div id="toc"></div>

Mocha是一个非常流行的JavaScript 测试框架，可以运行在 Node.js和浏览器中。测试用例的语法与[Jasmine](/2017/03/27/js-test3-jasmine/)类似。测试用例如:

```
var assert = require('assert');
describe('Array', function() {
  describe('#indexOf()', function() {
    it('should return -1 when the value is not present', function() {
      assert.equal(-1, [1,2,3].indexOf(4));
    });
  });
});
```

> 实际上这个测试用例也完全可以放到jasmine下运行。


### Node.js后台运行Mocha测试

安装mocha
```
npm install -g mocha

```

当前目录创建test.js文件，文件内容如上举例的测试用例。

`mocha .` 可以运行测试并看到结果。

> mocha默认会从test目录下读取测试文件。

另在npm管理的项目中，可以`npm install -D mocha`将mocha加到package.json的devDependencies中，并在package.json中设置test如:

```
"scripts": {
    "test": "mocha ."
}
```

而后 `npm test` 或 `npm run test` 即可以读取当前目录下的测试文件并执行测试。

### Node.js + mocha + chai

Mocha库中没有自己的断言，但是支持使用第三方的断言：node.js自带的断言(assert)，expect.js的断言(expect)， chai库的(expect, assert, should style), better-asset, unpected. 关于断言，请参考[JS测试之断言](/2017/04/08/js-test5-assertions/)。

下面是使用流行的断言库chai与mocha集成的例子。

- `npm init`创建一个项目，会在当前目录生成package.json文件来管理库信息，依赖以及npm脚本。

- `npm install -D mocha chai` 安装依赖包mocha以及chai, 并将安装的版本信息写到package.json的devDependencies中。

- `mkdir test && touch test.js` 创建测试文件:

  ```
  var expect = require('chai').expect;
  describe('Array', function() {
    describe('#indexOf()', function() {
      it('should return -1 when the value is not present', function() {
        expect([1,2,3].indexOf(4)).to.equal(-1);
      });
    });
  });
  ```
- 在package.json中设置test任务脚本:

  ```
  "scripts": {
      "test": "mocha"
  }
  ```

- `npm test` 或 `npm run test`即可以运行测试。


### 浏览器运行Mocha测试

- `mocha init test/` 在test目录下生成mocha浏览器测试的相关文件index.html, mocha.css, mocha.js以及tests.js.

- index.html中会调用`mocha.run()`来运行测试，其文件内容如下:

  ```
  <!DOCTYPE html>
  <html>
    <head>
      <title>Mocha</title>
      <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <link rel="stylesheet" href="mocha.css" />
    </head>
    <body>
      <div id="mocha"></div>
      <script src="mocha.js"></script>
      <script>mocha.setup('bdd');</script>
      <script src="http://chaijs.com/chai.js"></script>
      <script src="tests.js"></script>
      <script>
        mocha.run();
      </script>
    </body>
  </html>
  ```

  > `<script src="http://chaijs.com/chai.js"></script>` 是手动加上去的，因为在浏览器中需要一个全局的chai.expect对象。

- 将测试用例放入tests.js中，注意将`var expect = require('chai').expect;`修改为`var expect = chai.expect`。

- 浏览器打开index.html，即可看到测试结果。

{% include image.html url="/static/img/js/mocha-test-result.png" description="图1 Mocha浏览器测试结果" width="600px" inline="true" %}


### 引入babel测试ES6

Node.js中只支持一部分的ES6语法，但是一部分是不支持的。如`import`关键字。

- 将测试文件中的`var expect = require('chai').expect;`修改为`import { expect } from 'chai';`。

- `npm test` 将会报错:

  ```
  (function (exports, require, module, __filename, __dirname) { import { expect } from 'chai';
                                                                ^^^^^^
  SyntaxError: Unexpected token import
  ....
  ```

- 要解决上一步的问题，需要引入babel, 首先`npm install -D babel-core babel-preset-es2015`安装babel依赖。

- 项目目录下创建`.babelrc`配置文件
  ```
  {
    "presets": [ "es2015" ]
  }
  ```

- `mocha --compilers js:babel-core/register`跑测试，测试不会再报异常。

> 从Babel 6.0开始，不再直接提供浏览器版本，而是要用构建工具构建出来。最好的使用方法是先将代码打包成浏览器可以使用的版本，如使用Babel和Browserify

### Mocha异步测试

#### mocha原生异步测试

mocha为it方法提供了一个回调函数done，如:

```
describe('#save()', () => {
  it('should save without error', () => {
    var user = new User('Luna');
    user.save((err) => {
      if (err) done(err);
      else done();
    });
  });
});
```

配置timeout:

```
it('should take less than 500ms', function(done){
  this.timeout(500);
  setTimeout(done, 300);
});
```


#### chai-as-promised异步测试

`npm install -D  chai-as-promised` 安装依赖。

如果引入了chai-as-promised, 测试Promise可以不使用回调函数done, 如:

```
import chai from 'chai';
import chaiAsPromised from 'chai-as-promised';
chai.use(chaiAsPromised);
const expect = chai.expect;

const testPromise = () => Promise.resolve([1, 2, 3]);

describe('#testPromise', function() {
  it('respond with matching records', function() {
    return expect(testPromise()).to.eventually.have.length(3);
  });
});
```

### Mocha常用命令/函数

#### Mocha常用命令

- `mocha --recursive` 测试test子目录下面所有的测试用例。

- `mocha --watch` 用来监视测试脚本的变化，每个变化都会重新运行测试。

- `mocha --bail` 用来`fail fast`，即当一个测试用例失败时，就停止执行后面的所有测试。

- `mocha 'test/**/*.@(js|jsx)'` 测试指定运行test目录下面任何子目录中、文件后缀名为js或jsx的测试脚本。

- `mocha --reporter [报告样式]`可以设置[报告样式](http://mochajs.org/#reporters)，其默认值为`spec`, 也可以设置为`tap`，`dot`, `nyan`, `list`, `progress`, `json`等。

#### Mocha测试用例中常用函数

- describe, describe.only, describe.skip

- it, it.only, it.skip

- before()、after()、beforeEach()和afterEach()

- 在describe中使用`this.retries(4);`，设置当测试失败时测试的重复次数，不建议在单元测试中使用

- 可以在describe中动态生成测试用例，如下代码将生成3个测试

  ```
  var expect = require('chai').expect;

  function add(array) {
    return array.reduce(function(prev, curr) {
      return prev + curr;
    }, 0);
  }

  describe('#add', () => {
    var tests = [
      {args: [1, 2],       expected: 3},
      {args: [1, 2, 3],    expected: 6},
      {args: [1, 2, 3, 4], expected: 10}
    ];

    tests.forEach(function(test) {
      it('correctly adds ' + test.args, function() {
        var res = add(test.args);
        expect(res).to.equal(test.expected);
      });
    });
  });
  ```

### 参考资料

- [Mocha主页](https://mochajs.org/)

- [Nodejs下的ES6兼容性与性能分析](https://my.oschina.net/zhangstephen/blog/541276)

- [ECMAScript compatibility table](https://kangax.github.io/compat-table/es6/)

- [Chai Assertions for Promises](https://github.com/domenic/chai-as-promised)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
