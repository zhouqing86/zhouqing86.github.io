---
layout: post
comments: false
categories: "React"
date:   2017-9-17 00:30:54
title: [WIP]Redux Form(6.0)
---

<div id="toc"></div>
<br />

项目中用到了Redux Form，在使用的过程中遇到很多问题，希望通过这篇文章来总结所获。

## 需求的引入
在介绍Redux Form之前，我们先了解在做一个表单（拿修改个人信息表单来说）时我们的一些具体需求：

- 从后台API获取个人信息，回填到表单中

- 修改某字段如电子邮箱，可以即时验证电子邮箱的格式是否正常

- 提交表单前端验证表单的字段，如果有验证不通过字段，提示错误

- 提交表单前判断表单没有任何字段修改，则不需要提交

- 表单不可以重复提交，即第一次提交后提交按钮不可再点击

- 表单后后台验证失败，在页面显示验证失败信息到相应的字段

- 后端API请求未知错误会超时，在页面上显示错误提示

- 后端更新成功后，前端页面跳转到个人信息展示页

另外，关于表单的操作，我们还有一些需求：

- 表单联动，如地址选择中，选择省后，能够列出这个省的所有市

- 文件上传

面对这么多的需求，我们试图在Redux的基础上庖丁解牛。

## Redux Form介绍

Redux Form是基于Redux的方便表单管理的库。其为了更方便的管理Form相关的状态而定义了一些基础组件以及接口。我们先从如何使用Redux form(版本6.0)入手。

### 将Redux From引入项目

项目中如果已使用了Redux，可以很方便的引入Redux Form。

- 使用Redux Form提供的Reducer来管理Form的状态

  ```
  import { createStore, combineReducers } from 'redux';
  import { reducer as formReducer } from 'redux-form';

  const rootReducer = combineReducers({
    // 你的其他的Reducer
    form: formReducer
  });

  const store = createStore(rootReducer);
  ```

> 注意的是，这里的key值默认必须是form，当然也可以使用自定义的key值，但比较麻烦，需要在定义reduxForm时实现getFormState方法。


## 使用中遇到的问题


## 参考资料

- [Redux Form](https://github.com/erikras/redux-form)


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
