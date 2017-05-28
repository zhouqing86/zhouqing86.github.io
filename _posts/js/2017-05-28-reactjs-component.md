---
layout: post
comments: false
categories: "React"
date:   2017-5-27 18:30:54
title: React组件(Components)
---

<div id="toc"></div>
<br />

组件(Components)是React进行模块化的重要武器，通过定义不同的组件，React将UI定义成独立的可重用的块。

ES6支持定义类，而也React定义了React.Component，自定义的组件类只需要继承React.Component就获取了React给组件定义的默认方法的流程。

```
class Greeting extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

而在不支持ES6语法的JavaScript中（如ES5)如果要使用React的组件， 需要使用React.createClass方法来定义组件。

```
React.createClass({
 render() {
   return (
     <div>Hello, {this.props.name}</div>
   );
 }
});
```

<br />
## React.Component

### 组件的属性(props)与状态(state)

对于如下定义的Greeting对象，我们来对属性和状态做直观的区分：

```
class Greeting extends React.Component {
  constructor() {
    this.state = {
      data: [
        {
          id: 1,
          name: 'Tom'
        }
      ]
    };
  }

  render() {
    return (
        <div>
          <h1>Hello, {this.props.name}</h1>
          <table>
            <tbody>
              {this.state.data.map((person, i) => <TableRow key = {i} data = {person} />)}
            </tbody>
          </table>
        </div>
    );
  }
}
```

Greeting组件内的constructor方法中使用初始化了组件的状态对象，在render方法中可以直接通过`this.state`来获取组件的状态对象。
render方法中还使用了Greeting组件的属性name，name的值需要通过`<Greeting name='tom' />`的方式传递进组件。

组件的属性(props)与状态(state)都是组件的数据来源，它们的共同点有：

- 两者都是纯JS对象

- 两者发生变化时都会触发更新组件的事件

- 两者的值对组件的render起着决定性的作用，属性与状态决定了唯一的render结果

它们之间的不同点是:

- 如果组件在某个时间点需要修改某些数据，难么这些数据应该定义在state上

- 父组件与子组件通过属性传递数据，父组件可以将自己状态里的数据以属性的形式传递给子组件，也可以传递回调函数给子组件，子组件通过调用这个句柄影响到父组件状态的变化

- 组件的状态是可选的，所以有Stateless Component与Stateful Component的区分

- 初始化的时候可以使用 `this.state=`，其他的地方必须使用 `this.setState`，但是也不代表使用了`this.setState`就是对的，考虑setState的异步特性，不推荐使用

  ```
  this.setState({
    count: this.state.count + 1
  });
  ```
  这种调用也是异步的，状态不会立即被改变，如接近在这个setState后面再进行setState的话，其将覆盖前面的语句，即如果你期望的是两次的count + 1，但实际上count只加1。所以推荐使用：

  ```
  this.setState((prevState, props) => {
    return { count: prevState.count + 1 }
  });
  ```

- 组件不能修改自己的属性

### 组件的生命周期相关方法

组件中定义了一套执行模板，规定了一系列方法的执行的先后顺序，方法链中的有些方法`will`与`did`成对出现。成对出现的方法中`will`是先于`did`执行的。

`创建`一个组件并将其插入到DOM中将依次执行的方法是constructor(), componentWillMount(), render(), componentDidMount()。

`更新`一个组件发生在组件的props以及states发生变化时，将依次执行的方法是componentWillReceiveProps(), shouldComponentUpdate(), componentWillUpdate(), render(), componentDidUpdate()。

`移除`一个组件时将调用componentWillUnmount()。

需要注意的是:

- constructor方法中别忘了在第一句是super(props)。

- 如果需要定义与浏览器交互的内容，不要在render中定义，建议在componentDidMount()或者其他方法中定义。

- componentWillMount在render之前执行，在这里面修改状态的值并不会触发组件的重新render， 这个方法也是唯一的一个Server Rendering的钩子。

- componentDidMount在组件mount后立即被调用，如果需要异步调用获取远程数据，建议在此调用，在此方法中设置状态将触发组件的重新加载。

- 当父组件的状态变化引起子组件的属性变化时，子组件重新render时会调用componentWillReceiveProps，但是子组件在props没有发生变化时可能也会调用此函数，所以在此方法中需要对比`this.props`与`nextProps`然后调用`this.setState()`来传递变化。注意的是，在初始化组件或this.setState()都不会触发对componentWillReceiveProps的调用。如果需要属性变化触发状态变化，建议将逻辑写在这个方法里。

- shouldComponentUpdate用来指示组件的render输出会不会受当前状态和属性的影响，如果复写此方法并返回false, 其不会阻止`子组件`的重新render, 但是会阻止当前组件的componentWillUpdate(), render(), 和 componentDidUpdate()被调用。在组件被创建或者forceUpdate()时此方法将不会被调用。

- componentWillUpdate在组件重新render时被调用，但是其里面不能使用`this.setState()`。注意当shouldComponentUpdate()返回false时，其不会被调用。

- componentDidUpdate在组件重新render后被调用，其也是一个远程获取数据的合适的地方。注意当shouldComponentUpdate()返回false时，其不会被调用。

- componentWillUnmount当组件unmount或者destroy时会被调用，可以在此方法里做一些清理工作如撤销定时器，撤销网络请求。

### 其他方法

在开始此节之前，想问的一个问题是：组件重新render是否一定会导致更新DOM? 我们马上通过了解forceUpdate来得到答案。

forceUpdate方法强制组件重新render，但是组件的重新render并不代表要更新DOM: 只有在DOM结构或内容发生变化才会更新DOM。对于当前组件忽略对shouldComponentUpdate的调用，但子组件还是会走起正常的流程，会调用shouldComponentUpdate方法。

而对于类的默认属性值，React Component中定义了defaultProps方法来给出默认值：

```
class CustomButton extends React.Component {
  // ...
}

CustomButton.defaultProps = {
  color: 'blue'
};
```

注意的是当调用此组件并在color属性上传递null时，组件初始化时使用的null而并不会使用默认值blue。

对于属性所要接收的属性的值做类型检查需要使用propTypes方法：

```
CustomButton.propTypes = {
  color: React.PropTypes.string
};
```

如果传入的color属性不是字符串，程序将打印告警(warn)。

### 自定义方法

在组件类里，可以自定义方法，如

```
class App extends React.Component {
  onInputChangeHandler(e) {
    this.setState({ inputValue: e.target.value });class App extends React.Component {
  }
}  
```

但是render方法如何调用自定义的方法呢？其实比较简单如：

```
<input type="text" value={this.state.inputValue} onChange={this.onInputChangeHandler}/>
```

但是上面的调用会存在问题，报错`Uncaught TypeError: Cannot read property 'setState' of undefined`。原因是onInputChangeHandler不知道this是什么东西，所以我们需要使用如 `onChange={this.onInputChangeHandler.bind(this)}`，
或在constructor中bind this对象：

```
constructor() {
  this.onInputChangeHandler = this.onInputChangeHandler.bind(this);
}
```

<br />
## React.createClass

基于对React.Component的介绍，这里介绍一些createClass使用上不同的地方：

- 使用getInitialState方法来代理constructor方法:

  ```
  const Contacts = React.createClass({
    getInitialState () {
      return {

      };
    },
    render() {
      return (
        <div></div>
      );
    }
  });
  ```

- React.createClass将propTypes和defaultProps至于创建对象内，如:

  ```
  const Contacts = React.createClass({
    propTypes: {

    },
    getDefaultProps() {
      return {

      };
    }
  });
  ```

- React.createClass不需要手动的bind this对象

  ```
  const Contacts = React.createClass({
    handleClick() {
      console.log(this); // React Component instance
    },
    render() {
      return (
        <div onClick={this.handleClick}></div>
      );
    }
  });

  ```

- React.createClass可以使用mixin, React.Component中已不再支持

  ```
  var SomeMixin = {
  doSomething() {

    }
  };
  const Contacts = React.createClass({
    mixins: [SomeMixin],
    handleClick() {
      this.doSomething(); // use mixin
    }
  });
  ```

Facebook之前在v0.13版本时就建议使用React.Component，最终的目的是React.Component完全替换掉createClass。

<br />
## React.PureComponent

React 15.3.0 新增了一个 PureComponent类, 翻译过来就是纯组件类。

```
class MyComponent extends PureComponent {...}
```

那什么是纯组件呢？即给定的输入（属性与状态）将产生给定的render结果。这种假定下PureComponent类改变了生命周期方法shouldComponentUpdate，并且它会自动检查组件的状态(props)和状态(state)是否发生了变化（使用浅等于，即比较对象的属性和值，但不会嵌套判断子对象的属性和值)，如果没有发生变化就不会重新render。

举例说明：

```
handleClick() {
  let {items} = this.state

  items.push('new-item')
  this.setState({ items })
}

render() {
  return (
    <div>
      <button onClick={this.handleClick} />
      <ItemList items={this.state.items} />
    </div>
  )
}
```

如果ItemList是纯组件，点击定义的按钮是不会重新渲染，因为his.state.items 的值发生了改变，但是它仍然指向同一个对象的引用。要想使其渲染，必须按如下方式写handleClick:

```
handleClick() {
  this.setState(prevState => ({
    words: prevState.items.concat(['new-item'])
  }));
}
```


## 参考资料

> 本文内容都基于React 15.4.2版本。

- [React.Component](https://facebook.github.io/react/docs/react-component.html)

- [props vs state](https://github.com/uberVU/react-guide/blob/master/props-vs-state.md)

- [ReactJS: Props vs. State](http://lucybain.com/blog/2016/react-state-vs-pros/)

- [在React.js中使用PureComponent的重要性和使用方式](http://www.open-open.com/lib/view/1484466792204)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
