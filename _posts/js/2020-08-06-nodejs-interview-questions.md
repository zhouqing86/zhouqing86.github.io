---
layout: post
comments: false
categories: interview
date:   2020-08-06 00:30:54
title: NodeJS面试题汇总
---

<div id="toc"></div>

不仅仅包括Node.JS相关的面试题，还包括`React`, 单元测试相关等面试题。


## NodeJS

### require, import, export, exports

- [JavaScript module.exports, require, import, export, define ??](https://medium.com/@vishwa.efor/javascript-module-exports-require-import-export-define-cc04461f4d5e)

- [commonJs、AMD和ES6模块化的总结](https://juejin.im/post/6844903939763077127)

模块化使得源代码能够拆分成独立的可重复使用的代码逻辑单元。ES5版本以及ES5之前版本的Javascript本身是不支持模块化的，但是通过第三方库有一些变通方法使得其能够支持模块化。
ES6版本的Javascript是原生支持模块化的。

- 为了让ES5支持模块化，第三方库提出了各种方案，如AMD模块化格式和CommonJS模块化格式。

#### AMD模块化格式

AMD是一个在浏览器前端实现模块化的规范，RequireJS是对这个规范的实现。AMD有两个API，define用于定义模块，require用于调用模块。

```
//app.js中定义app模块，其依赖于playlist和song模块，分别是playlist.js和song.js
define(['playlist', 'song'], function(P, S) {
  //App code          
}
```

在网页中使用如：

```
<script data-main="js/app.js" src="js/require.js"></script>
```

这里的`js/require.js`是来自[https://github.com/requirejs/requirejs](https://github.com/requirejs/requirejs)，`js/app.js`可以看做是程序的入口。

>完整的代码可以查看[这里](https://github.com/vzztalks/javascrypt-get-started/tree/master/modularization/amd)

#### CommonJS模块化格式

CommonJS是用在服务器端的模块化格式，`Node.js`常用这种模块化格式，其引入了一种`import-export`机制，每一个模块将导出（`module.exports`）一些东西来给其他模块导入（`require`）来使用。

```
//app.js
var Playlist = require('./playlist.js');
var Song = require('./song.js');

//song.js
function Song() {}
module.exports = Song;

//playlist.js
function Playlist(){}
module.exports = Playlist;
```

CommonJS模块化格式也可以用于前端:

```
<script src="system.js"></script>
<script>
  System.import('/js/app.js');
</script>
```

> 完成的代码可以看[这里](https://github.com/vishwaefor/javascrypt-get-started/tree/master/modularization/common-js)

#### ES6中对模块化的支持

ES6的规范中引入两个关键字`import`和`export`。

```
//app.js
import Playlist from './playlist.js'; // defult import
import { Song } from './song.js'; // named import

//song.js
function Song() {}
export { Song }; // named export

//playlist.js
function Playlist(){}
export default Playlist; // default export
```

前端加载ES6模块化的代码：

```
<script src="js/app.js" type="module"></script>
```

> 完成的代码可以看[这里](https://github.com/vishwaefor/javascrypt-get-started/tree/master/modularization/es6)

但是需要注意的是，浏览器对于很多`ES6`版本规范中定义的语法并不支持。另`Node.js`也不支持`import-export`语法。
虽然如此，我们还是可以完全使用`ES6`规范语法来编写我们的代码，我们只需要通过`Babel`这样的第三方库来将ES6的代码转化成旧格式的代码。

### Promise vs Async/Await

- [Graceful asynchronous programming with Promises](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises)

- [JavaScript: Promises or async-await](https://medium.com/better-programming/should-i-use-promises-or-async-await-126ab5c98789)

#### 什么是Promise

本质上，`Promise`是一个对象，其代表着一个操作的中间状态。实际上，是对将来某一个时间点某种结果将被返回的承诺。
当然，其不会精确的保证具体返回什么，但是其能保证，当返回结果可用时，将会做给定逻辑处理，或者当返回结果不可用/失败时，将处理异常。

创建一个`Promise`对象以及如何来使用这个Promise对象：

```
var promise = new Promise(function(resolve, reject){
  setTimeout(function(){
    const num = Math.random() * 100;
    if (num > 50) {
      resolve(Math.floor(num));
    } else {
      reject('Fail to generate random number');
    }
  }, 1000)
});

function printValue(value) {
  console.log(value);
  return value;
}

promise.then(value => value * value )
      .then(printValue)
      .then(value => value + 2)
      .then(printValue)
      .catch(err => console.log(err));
```

这里的`setTimeout`用来模拟异步运算。promise的调用链中`then`处理的是`resovle`函数的输入参数，而`catch`处理的是`reject`函数的输入参数。

promise有三种状态：未完成，完成和失败。promise的状态只能由未完成转换成完成，或者未完成转换成失败。


#### 什么是Async/Await

- [async和await:让异步编程更简单](https://developer.mozilla.org/zh-CN/docs/learn/JavaScript/%E5%BC%82%E6%AD%A5/Async_await)

## ReactJS

### 组件生命周期

参考[组件的生命周期相关方法](/2017/05/28/reactjs-component/#组件的生命周期相关方法)

#### 对API的调用应该写在那个生命周期方法里？

- componentDidMount在组件mount后立即被调用，如果需要异步调用获取远程数据，建议在此调用，在此方法中设置状态将触发组件的重新加载。

- componentDidUpdate在组件重新render后被调用，其也是一个远程获取数据的合适的地方。注意当shouldComponentUpdate()返回false时，其不会被调用。

#### 如何给组件设置默认props值

React Component中定义了defaultProps方法来给出默认值。

### React里的bind

### PureComponent

### 如何理解setState的异步特性

不推荐使用：

```
this.setState({
  count: this.state.count + 1
});
```
这种调用也是异步的，状态不会立即被改变，如在这个setState后面再进行setState的话，其将覆盖前面的语句，即如果你期望的是两次的count + 1，但实际上count只加1。推荐使用：

```
this.setState((prevState, props) => {
  return { count: prevState.count + 1 }
});
```

### props vs state

### render函数的执行时机

#### 调用setState但是如果state没有发生改变，会触发render函数的调用吗？

只要 setState 方法执行，render 函数就会执行。

#### props发生改变就会调用render函数吗？

组件的 props 改变了，不一定触发 render 函数的执行，除非 props 的值来自于父组件或者祖先组件的 state，在这种情况下，组件的 props 改变，也就意味着父组件或者祖先组件的 state 发生了改变，也就是父组件或者祖先组件执行了 setState 方法；那么可以总结出，render 函数的执行时机就是 setState 方法的执行。

#### render方法被调用就意味着需要进行DOM操作吗？

Render 函数执行并不一定意味着发生 DOM 操作，render 函数执行只是返回虚拟 DOM，需要通过比较新旧虚拟 DOM 来决定是否发生 DOM 操作，新旧虚拟 DOM 的比较

### Diff算法


### React Redux

参考[React中使用Redux](/2017/05/28/reactjs-redux/)。


## TDD/BDD

### Spy vs Stub vs Mock




<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
