---
layout: post
comments: false
categories: "测试JS"
date:   2017-03-26 18:30:54
title: JS测试之Jasmine
---

<div id="toc"></div>

Jasmine是面向行为驱动开发(BDD)的Javascript单元测试框架，它不依赖于任何其他Javascript框架，也不依赖浏览器、DOM。

### Jasmine和BDD有什么关系

关于什么是行为驱动开发(BDD)，我们暂时不必要太深入的了解其概念。我们只需要了解我们可以在测试中使用"通用语言"来定义系统/系统的行为。

所谓"通用语言"，我们可以理解为我们的自然语言。 一个BDD"通用语言""风格的测试描述："如果给定两个数字，传入自动以的Add方法，其得到的结果将是两个数字之和"。

基于Jasmine框架开发的测试用例，其实只是BDD"通用语言"描述的单元测试用例。其不同于严格意义上的BDD测试用例，也不同通常理解的Junit测试等。 也许例子的对比会是最好的说明：

- BDD自动化测试框架Cucumber的测试用例，这个用例是开发人员、产品经理乃至销售人员都能看懂的:

  ```
  Feature: Addition
    In order to avoid silly mistakes
    As a math idiot
    I want to be told the sum of two numbers

    Scenario: Add two numbers
    Given I have entered 50 into the calculator
    And I have entered 70 into the calculator
    When I press add
    Then the result should be 120 on the screen
  ```

- Jasmine的测试:

  ```
  describe('JavaScript addition operator', function () {
      it('adds two numbers together', function () {
          expect(1 + 2).toEqual(3);
      });
  });
  ```

- Java Junit测试：

  ```
  public class Test {
      @org.junit.Test
      public void testAdd() throws Exception {
          Assert.assertEquals(3, 1+2);
      }
  }
  ```

### Node.js后台运行Jasmine测试

- 安装jasmine `npm install -g jasmine`.

- 初始化 `jasmine init`, 将会生成一个spec/support/jasmin.json的文件:

  ```
  {
    "spec_dir": "spec",
    "spec_files": [
      "**/*[sS]pec.js"
    ],
    "helpers": [
      "helpers/**/*.js"
    ],
    "stopSpecOnExpectationFailure": false,
    "random": false
  }
  ```

- 使用 `jasmine examples` 创建测试用例的例子，此命令将生成Player, Song类以及PlaySpec, 还自定义了一个Matcher: toBePlaying。为了使例子更简单，我们也可以自己在spec下创建一个文件AdditionSpec.js:

  ```
  describe('JavaScript addition operator', function () {
      it('adds two numbers together', function () {
          expect(1 + 2).toEqual(3);
      });
  });
  ```

- 运行 `jasmine` 就可以看到测试结果了。


### 浏览器前端运行Jasmine测试

在上一步创建的测试用例的基础上，与spec的同目录创建specRunner.html:

  ```
  <!DOCTYPE html>
  <html>
  <head>
    <meta charset="utf-8">
    <title>Jasmine Spec Runner v2.5.2</title>
    <link rel="shortcut icon" type="image/png" href="https://cdnjs.cloudflare.com/ajax/libs/jasmine/2.5.2/jasmine_favicon.png">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/jasmine/2.5.2/jasmine.css">
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jasmine/2.5.2/jasmine.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jasmine/2.5.2/jasmine-html.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jasmine/2.5.2/boot.js"></script>

    <script src="spec/AdditionSpec.js"></script>
  </head>
  <body>
  </body>
  </html>
  ```

直接在浏览器打开specRunner.html就可以看到AdditionSpec.js的测试结果。

{% include image.html url="/static/img/js/jasmine-browser.png" description="图1 Jasmine浏览器测试结果" width="600px" inline="true" %}


### 源码分析

在jasmine源码的lib/spec/jasmine.js（4000+行代码)里，定义了describe，it等函数，并将这些函数其绑定到相应的环境(浏览器环境是window上，Node.js环境在global上)。

- 在jasmine.js中，其首先定义了一个函数getJasmineRequireObj，在这个函数调用中返回几乎所有jasmine的对象。

  ```
  var getJasmineRequireObj = (function (jasmineGlobal) {
    var jasmineRequire;

    if (typeof module !== 'undefined' && module.exports && typeof exports !== 'undefined') {
      if (typeof global !== 'undefined') {
        jasmineGlobal = global;
      } else {
        jasmineGlobal = {};
      }
      jasmineRequire = exports;
    } else {
      if (typeof window !== 'undefined' && typeof window.toString === 'function' && window.toString() === '[object GjsGlobal]') {
        jasmineGlobal = window;
      }
      jasmineRequire = jasmineGlobal.jasmineRequire = jasmineGlobal.jasmineRequire || {};
    }

    function getJasmineRequire() {
      return jasmineRequire;
    }

    getJasmineRequire().core = function(jRequire) {
      var j$ = {};
        .....
      return j$;
    };

    return getJasmineRequire;
  })(this);
  ```

- Jasmine将describe, it, expect 等方法绑定到了window或者global上， describe的实现如：

  ```
  this.describe = function(description, specDefinitions) {
        ensureIsFunction(specDefinitions, 'describe');
        var suite = suiteFactory(description);
        if (specDefinitions.length > 0) {
          throw new Error('describe does not expect any arguments');
        }
        if (currentDeclarationSuite.markedPending) {
          suite.pend();
        }
        addSpecsToSuite(suite, specDefinitions);
        return suite;
      };
  ```

- Jasmin中定义QueueRunner来执行所有测试用例并统计测试结果，其定义了execute和run方法。

- Jasmine 2.0版本后在需要使用boot.js來初始化以及运行测试。

  ```
  window.jasmine = jasmineRequire.core(jasmineRequire);
  jasmineRequire.html(jasmine);
  var env = jasmine.getEnv();
  ....
  window.onload = function() {
      if (currentWindowOnload) {
        currentWindowOnload();
      }
      htmlReporter.initialize();
      env.execute();
    };
  ```

### Jasmine常用函数

- describe里可以定义一系列测试用例； xdescribe定义的测试用例不会被执行；如果在测试文件中有定义fdescribe，则表示只执行fdescribe下定义的测试用例。

- it, xit, fit

- 如果在Spec的函数体中调用 pending() 函数，那么该Spec也会被标记为Pending。 pending() 函数接受一个字符串参数

- beforeEach, afterEach, beforeAll, afterAll

- expect, toBe, not.toBe, toEqual, toBeDefined, toBeNull, toContain, toThrow, toThrowError, toMatch

- jasmine.clock()用来模拟操纵时间,  要想使用jasmine.clock()，先调用jasmine.clock().install告诉Jasmine你想要在spec或者suite操作时间，当你不需要使用时，务必调用jasmine.clock().uninstall来恢复时间状态。jasmine.clock().tick(milliseconds)来控制时间前进。

- Spy用来追踪函数的调用历史信息（是否被调用、调用参数列表、被请求次数等）。当在一个对象上使用spyOn方法后即可模拟调用对象上的函数，此时对所有函数的调用是不会执行实际代码的。

  - spyOn(foo, "setBar")将在foo对象上添加spy。

  - spyOn().and.returnValue()，由于Spy是模拟函数的调用，因此我们也可以强制指定函数的返回值。

  - spyOn().and.callThrough()，告诉Jasmine我们除了要完成对函数调用的跟踪，同时也需要执行实际的代码。

  - spyOn().and.callFake()，与returnValue相似，callFake则更进一步，直接通过指定一个假的自定义函数来执行。

  - spyOn().and.throwError()，模拟异常的抛出

- Jasmine支持测试需要执行异步操作的specs，调用beforeEach , it , 和afterEach 的时候，可以带一个可选的参数done ，当spec执行完成之后需要调用done 来告诉Jasmine异步操作已经完成。默认Jasmine的超时时间是5s，可以通过全局的jasmine.DEFAULT_TIMEOUT_INTERVAL设置。

### 参考资料

- [Testing Your JavaScript With Jasmine](https://code.tutsplus.com/tutorials/testing-your-javascript-with-jasmine--net-21229)

- [Jasmine 代码地址](https://github.com/jasmine/jasmine)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
