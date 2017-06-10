---
layout: post
comments: false
categories: "React"
date:   2017-5-28 00:30:54
title: React中使用Redux
---

<div id="toc"></div>
<br />

在编写本文时，笔者已经有了一定的React和Redux的经验，此文只是为了加深对Redux的理解而写，所以会忽略掉很多基础的知识细节。如果想了解React组件，可以查看笔者的另一篇文章[React组件(Components)](/2017/05/28/reactjs-component/)。
<br />
## 问题提出及解决方案

### 问题的提出

React通过属性(props)与状态(state)来控制组件的render输出，每个组件都需要独立管理自己的状态(state)，父节点通过props向子孙节点传递数据，而子孙组件间状态变化需要向上传递则通过props的回调函数。这样存在的问题是项目中状态管理是分散的，且项目变大时状态管理也比较混乱。

### 思考问题的解决方案

在谈Redux之前，我们可以试着抽象的想一种解决方案来解决这种问题：

- 将所有组件状态的管理统一到一个地方，称之为状态管理者

- 状态管理者提供数据/状态增删改查的接口

- 当状态发生变化时将，状态管理者通知给相应的组件重新render

### 解决方案细节

再讨论下解决方案的细节：

- 状态管理者StateManager应维护了一个状态对象，如:

    ```
    {
        'comments': [],
        'tags': []
    }
    ```

- 状态的管理相对来说要复杂，如果维护的状态是如关系型数据库的表格形式，我们可以定义一套SQL语句类似的DSL来操作这个状态，但是因为状态每个key值对应的对象可能是字符串、数字、数组、对象，更新的方式的方式有多种不同，状态管理者需要提供统一的接口或着DSL来统一解决这多种操作情况。

    有一种解决方案是将状态的操作交由状态强相关的自定义操作函数。状态的管理者将所有的自定义操作对象存储在一个列表中，组件可以发送命令给状态管理者，命令管理者会找到对应的自定义操作函数。

    命令的格式如:

    ```
    {
        commandType: ADD_COMMENT
        comment: 'whatever comments'
    }
    ```

    而自定义的状态操作函数可能如

    ```
    const commentHandler = (command, state) => {
        switch(command.commandType) {
            case 'ADD_COMMENT':
                return state.comments.push(command.comment);
            default:
                return state;
        }
        ....
    }

    ```

    而状态管理者提供自定义操作函数的注册，如

    ```
    this.commandHandler = [];
    const registerCommandHandler = (handler) => {
        this.commandHandler.push(handler);
    }

    ```

    状态管理者也需要开发接口来接收command，并随之改变state, 如

    ```
    const dispatch = (command) => {
        this.state = this.commandHandler.forEach(handler => handler(this.state, command));
    }
    ```

- 状态发生变化后，需要通知到更新组件，我们也许会想到观察者模式，假定store中存在一个观察者列表，以及有一个subscribe函数可以增加观察者，如

    ```
    this.listeners = [];
    const subscribe = (listener) => {
        this.listeners.push(listener);
    }
    ```

    这时，我们还需要修改dispatch函数，当接受到一个命令时，不仅仅改变状态，也需要通知观察者:

    ```
    const dispatch = (command) => {
        this.state = this.commandHandler.forEach(handler => handler(this.state, command));
        this.listeners.forEach(listener => listener());
    }
    ```

上面的实现都是以Javascript伪代码的方式，笔者并没有实际验证代码正确性，但是思考的思路即如此，不过这个思路没有解决很多细节问题，如

- 大部分状态不需要React组件管理了，但是怎么将状态的变化传递给不同状态相关的组件

- React中不同的组件与这种listener机制结合到一起

- 每个React组件如何从状态管理者中获取状态信息

- command发出后需要异步从API获取数据，异步请求的逻辑要放到哪里

<br />
## Redux介绍

### 解开面纱

React+Redux中Redux就是状态管理者的角色。但是Redux并不是依附于React存在的，我们来看一个Redux作为状态管理的例子：

```
import { createStore } from 'redux';

function counter(state = 0, action) {
  switch (action.type) {
  case 'INCREMENT':
    return state + 1
  case 'DECREMENT':
    return state - 1
  default:
    return state
  }
}

let store = createStore(counter);
store.subscribe(() =>
  console.log(store.getState())
);
store.dispatch({ type: 'INCREMENT' });
store.dispatch({ type: 'DECREMENT' });
```

上面的例子可以看出：

- redux通过`createStore`来创建一个状态管理者`store`

- `createStore`需要传入状态操作函数`counter`，这里成为`Reducer`，状态的变化都是状态操作函数引起的

- 状态管理者开放了`subscribe`接口来注册监听者，当状态发送变化时，监听者就会被调用

- 状态管理者开放了`dispatch`接口来发送命令，命令的处理逻辑在状态操作函数中

- 状态管理者开放了`getState`接口来读取所有状态

Redux库中有三个需要谨记的原则：

- Single source of truth，意味着整个应用的状态都存储在一个对象中

- State is read-only，意味着不能直接修改状态(除了Reducers)，修改状态的唯一方式是触发一个命令(dispatch an action)

- Changes are made with pure functions, 意味着状态的操作函数应该是纯函数，即给定输入一定是给定输出，不会有读写文件，网络请求等副作用操作在函数中

### react-redux

把react与redux联系起来的原因可以了解这篇文章[Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)。

在React+Redux项目中，理想情况下只是最上层的组件（如route handler）是需要知道Redux的存在，成为Container组件或Smart组件；下层的组件只是从props来接收所有的数据，调用从props传入的回调函数，下层的组件被称为Presentational组件或Dumb组件。

Smart组件长得样子如下：

```
import { connect } from 'react-redux';
import Counter from '../components/Counter';
import { increment } from '../actionsCreators';

function mapStateToProps(state) {
  return {
    value: state.counter
  };
}

function mapDispatchToProps(dispatch) {
  return {
    onIncrement: () => dispatch(increment())
  };
}

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(Counter);
```

可以看出，`Smart`组件是将状态和状态操作接口绑定在了`Dumb`组件上。执行绑定的是react-redux提供的`connect`函数。

`connect`函数还有更多的高级用法，可参考

react-router还需要解决一个问题，就是怎么样把状态管理者对象与React关联起来呢。只是有了connect函数显然不够。react-router提供了一个`Provider`组件来引入状态管理者，引入的方式如:

```
import { Provider } from 'react-redux';
React.render((
  <Provider store={store}>
    {() => <App />}
  </Provider>
), targetEl);
```

> Provider组件的child必须是一个函数，所以这里是{() => <App />}

在于react-router配合使用时，可以参考：

```
const Root = ({ store }) => (
  <Provider store={store}>
    <Router>
      <Route path="/" component={App} />
    </Router>
  </Provider>
);
```

### 遗留问题

- 关于引入Redux后如何做异步API调用，笔者后续会专门写一篇文章来详细介绍异步调用的问题解决

<br />
## 你可能不需要Redux

许多开发人员往往在项目的前期就在自己的React项目中引入Redux，但是却不清楚是否真的需要Redux。

在使用的过程中往往也会存在这样的疑问： 为什么增加一个简单的功能需要修改多个文件。

那么，其实很多时候你的项目是不需要Redux的：

- 如果刚开始学习ReactJS， 不要把Redux同时也当做你的第一选择

- 如果你的项目的状态管理并不复杂，并不需要一个状态管理者专门去帮你管理状态时

- 如果你使用RxJS，那么也许不需要专门引入Redux了，也为在RxJS实现一个合适自己的Redux并不难。


## 参考资料

- [Usage with React](http://redux.js.org/docs/basics/UsageWithReact.html)

- [Redux](https://github.com/reactjs/redux)

- [Three Principles of Redux](http://redux.js.org/docs/introduction/ThreePrinciples.html)

- [Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)

- [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
