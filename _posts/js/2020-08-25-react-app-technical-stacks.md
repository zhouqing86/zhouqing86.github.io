---
layout: post
comments: false
categories: reactjs
date:   2020-08-25 00:30:54
title: React前端应用的技术栈选择
---

<div id="toc"></div>

这段时间，从上一个项目下来的空歇期，在给公司内部做一些工具应用，前端用的是ReactJS，初始做前端技术选型时考虑的问题，值得在此记录。

模板项目的代码可以查看[https://github.com/zhouqing86/react-template-project](https://github.com/zhouqing86/react-template-project)

## 创建项目

在安装了node.js的机器上，使用 `npx create-react-app some-app` 来初始化一个应用。

使用`create-react-app`的好处是其引入了很多开发友好的特性，使得我们只需要关注代码的编写，而其他的开发sever的启动, 文件修改时`server`重新加载文件，`bundle`等等。

具体`create-react-app`包括的功能可查看[What’s Included?](https://github.com/facebook/create-react-app#whats-included)

### npx

`npx`命令是在`npm@5.2.0`中引入的一个工具，`npx`使得可以很方便的获取和使用`npm registry`中的客户端工具/可执行工具。

在没有`npx`这个工具之前，我们要使用`npx registry`里的客户端工具，往往需要调用`npm install -g mocha`从`npm registry`中下载`mocha`并全局安装，再调用`mocha`命令。

而又了`npx`后，不需要先全局安装，直接使用`npx mocha`即可执行`mocha`命令，使用者不需要关注`mocha`下载安装的过程。

### react-scripts

`react-scripts`中包含了`create-react-app`用到的脚本和配置。

譬如`react-scripts start`其实调用的是[start.js](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/scripts/start.js)，可以看到其使用了`webpack-dev-server`和`webpack`。

而`react-scripts test`其调用的是[test.js](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/scripts/test.js)，其用到的Jest相关配置由[createJestConfig.js](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/scripts/utils/createJestConfig.js)创建。

注意在create-react-app创建的React应用中直接运行`yarn jest`会失败，因为在项目中并没有存在Jest相关的配置文件，会出现的问题：

- 没有`babel`相关配置，会导致Jest解析不了一些`ES6`的语法，如`import`关键字等

- 没有`svg`文件相关的Jest`transform`配置，因而Jest处理不了`logo.svg`这类文件

如果实在希望在默认配置的基础上添加或者覆盖一些配置，建议使用[react-app-rewired](https://github.com/timarney/react-app-rewired)

### React.Component还是Functional Component

`React.Component`意味着类组件的方式：

```
import React from 'react';

class SomeComponent extends React.Component {
  constructor(props) {
    super(props);
    this.onClick = this.onClick.bind(this);
    this.state = {
      count: 0
    };
  }

  onClick() {
    this.setState(prevState => {
        count: prevState.count + 1
    });
  }

  render() {
    return (
      <div>
        <p>You clicked {this.state.count} times</p>
        <button onClick={onClick}>
          Click me
        </button>
      </div>
    );
  }
}
```

React版本16.8后引入了各种钩子(`Hooks`)，使得开发者可以很方便的使用`Functional Component`完成之前只有在React的`Class Component`才能完成的事情。

```
import React, { useState } from 'react';

function Example() {
  // Declare a new state variable, which we'll call "count"
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

`Hooks`使得前端的代码更简洁，也更容易抽取重复代码，因此，建议选择在项目中使用`Hooks`。

> 关于`Hooks`介绍的视频可以参考[Introducing Hooks](https://reactjs.org/docs/hooks-intro.html)。


## 静态代码检查

### prettier

项目中使用`yarn add -D prettier`在项目中引入`prettier`。在项目中创建`.prettierrc`:

```
{
  "bracketSpacing": true,
  "printWidth": 150,
  "singleQuote": true,
  "trailingComma": "es5",
  "tabWidth": 2,
  "useTabs": false,
  "semi": true
}
```

使用`yarn prettier --check src/**/*.js`就可以检查`src`目录下所有JS文件的格式是否符合`prettier`的设置。

而使用`yarn prettier --write src/**/*.js`就会修改`src`目录下所欲JS文件使其符合`prettier`的设置。

也可以在`package.json`中添加相关任务：

```
"prettier:check": "prettier --check src/**/*.js",
"prettier:write": "prettier --write src/**/*.js"
```

> 关于`prettier`的配置[https://prettier.io/docs/en/configuration.html](https://prettier.io/docs/en/configuration.html)，而prettierrc的schema可以查看[http://json.schemastore.org/prettierrc](http://json.schemastore.org/prettierrc)

### eslint

项目中使用`yarn add -D eslint`在项目中引入`eslint`，引入eslist相关的插件：

```
yarn add -D eslint-config-airbnb eslint-plugin-import eslint-plugin-jsx-a11y eslint-plugin-prettier eslint-plugin-react eslint-plugin-react-hooks
```

ESlint可以对代码进行静态检查，其配置文件`.eslintrc`如:

```
{
  "parser": "babel-eslint",
  "env": {
    "es6": true,
    "browser": true
  },
  "extends": [
    "airbnb",
    "airbnb/hooks",
    "eslint:recommended"
  ],
  "settings": {
    "import/resolver": {
      "node": {
        "paths": [
          "src"
        ],
        "extensions": [
          ".js",
          ".jsx"
        ]
      }
    }
  },
  "rules": {
    "max-len": ["error", { "code": 150 }],
    "arrow-body-style": "off",
    "comma-dangle": "off",
    "import/no-unresolved": "off",
    "jsx-a11y/anchor-is-valid": "off",
    "jsx-a11y/click-events-have-key-events": "off",
    "jsx-a11y/control-has-associated-label": "off",
    "jsx-a11y/no-static-element-interactions": "off",
    "no-console": "off",
    "no-param-reassign": "off",
    "no-plusplus": "off",
    "react-hooks/exhaustive-deps": "off",
    "react/forbid-prop-types": "off",
    "react/jsx-filename-extension": "off",
    "react/jsx-props-no-spreading": "off",
    "react/require-default-props": "off",
    "linebreak-style": "off",
    "arrow-parens": "off",
    "import/prefer-default-export": "off",
    "react/no-array-index-key": "off",
    "import/no-dynamic-require": "off"
  }
}
```

可以使用`yarn eslint src`来检查`src`目录下的JS文件。

也可以在`package.json`中添加任务：

```
"lint": "eslint src"
```

> 可以使用`yarn eslint --init`来初始化`.eslintrc`文件.

> 另外需要注意的是eslint和prettier可能会有冲突，可以选择修改prettier的配置或者eslint的配置来解决冲突。

### editorconfig

其实如果使用了`prettier`，不需要再配置`editorconfig`，因为`prettier`优先拿去`.prettierrc`的配置。不过给出一个基本的`.editorconfig`的例子:

```
root = true

[*.js]
end_of_line = lf
insert_final_newline = true
charset = utf-8

[*.{js,md,json}]
indent_style = space
indent_size = 2
```

### VSCode

使用[VSCode](https://code.visualstudio.com/download)来进行代码的开发。建议安装的插件：

- Prettier - Code formatter

  安装完这个插件后，在VSCode的`settings.json`中：

  ```
  {
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "[javascript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
    },
    // Set the default
    "editor.formatOnSave": false,
    // Enable per-language
    "[javascript]": {
        "editor.formatOnSave": true
    }
  }
  ```

  这样，VSCode就会默认会用`prettier`来格式化文件，同时对于`javascript`语言的文件，默认在保存时会自动格式化文件。

  > Windows下使用ctrl+shift+P来搜索`Open Settings`打开`settings.json`，在VSCode中使用alt+shift+F来格式化某个特定文件

- [ESlint](https://github.com/Microsoft/vscode-eslint#settings-options)


## Test

前端应用的单元测试编写并不容易，但是也建议对于核心模块/函数，能够尽量编写单元测试。

### jest

`react-scripts test`默认就是使用`jest`来进行测试。

相关经常需要使用的文档：

- [Jest Expect](https://jestjs.io/docs/en/expect)

- [Jest Globals](https://jestjs.io/docs/en/api)

一个简单的Jest的测试用例`StringUtils.test.js`：

```
/* eslint-disable */
import { truncateStringWithSuffix } from './StringUtils';
describe('StringUtils', () => {
  describe('#truncateStringWithSuffix', () => {
    it("should return origin string when it's length lower or equal than maxLength ", () => {
      expect(truncateStringWithSuffix('hello', 5)).toEqual('hello');
    });

    it("should return truncated string when it's length greater than maxLength", () => {
      expect(truncateStringWithSuffix('hello world', 5)).toEqual('he...');
    });
  });
});
```

这里的`describe`，`it`和`expect`都是Jest全局定义的函数，可以不需要显示的`import`，当然，也可以显示的引入如`import {describe, it, expect} from '@jest/globals`.

这里的`truncateStringWithSuffix`使我们在`StringUtils.js`中实现的一个函数：

```
const truncateStringWithSuffix = (str, maxlen, suffix = '...') => {
  if (str.length <= maxlen || suffix.length >= maxlen) {
    return str;
  }
  return `${str.slice(0, maxlen - suffix.length)}${suffix}`;
};

export { truncateStringWithSuffix };
```

在项目中`yarn test`就可以使用`create-scripts test`来运行单元测试检查实现代码的正确性。

### enzyme

`Enzyme`支持`Shallow Render`，`Shallow Render`在写单元测试时非常有用，其不依赖DOM，且其只`render`第一层的组件，不需要担心子组件的行为。

参考[https://create-react-app.dev/docs/running-tests/#option-1-shallow-rendering](https://create-react-app.dev/docs/running-tests/#option-1-shallow-rendering)

- 安装`Enzyme`相关依赖

  ```
  yarn add enzyme enzyme-adapter-react-16 react-test-renderer
  ```

- 创建`src/setupTests.js`

  ```
  import { configure } from 'enzyme';
  import Adapter from 'enzyme-adapter-react-16';
  configure({ adapter: new Adapter() });
  ```

- 创建对于`App`的测试

  ```
  import React from 'react';
  import { render } from '@testing-library/react';
  import App from './App';
  import { shallow } from 'enzyme';

  describe('App', () => {
    it('renders learn react link', () => {
      const { getByText } = render(<App />);
      const linkElement = getByText(/learn react/i);
      expect(linkElement).toBeInTheDocument();
    });

    it('renders learn react link', () => {
      const wrapper = shallow(<App />);
      console.log(wrapper.text());
      expect(wrapper.text()).toEqual(expect.stringContaining('Learn React'));
    });
  });
  ```

  这里面有两个测试，第一个测试使用了`@testing-library/react`中的`render`方法，第二个测试中使用了`Enzyme`的`shallow`方法。

  关于`shallow`的api，可以查看文档[ShallowWrapper API](https://github.com/enzymejs/enzyme/blob/master/docs/api/shallow.md#shallownode-options--shallowwrapper)

- 执行`yarn test`来执行测试。

### 测试覆盖率

参考 [Coverage Reporting](https://create-react-app.dev/docs/running-tests/#coverage-reporting)，可以运行`npm test -- --coverage`来生成测试报告。

其底层是使用了[Istanbuljs nyc](https://github.com/istanbuljs/nyc#installation--usage)，参考[nyc Installation & Usage](https://github.com/istanbuljs/nyc#installation--usage)中的描述：

```
Note: If you use jest or tap, you do not need to install nyc. Those runners already have the IstanbulJS libraries to provide coverage for you. Follow their documentation to enable and configure coverage reporting.
```

我们在`package.json`中定义测试覆盖率的任务：

```
"coverage": "npm test -- --coverage --watchAll=false --reporter=html",
```

运行`yarn coverage`后，测试报告将生成在`coverage/lcov-report`目录下。

> 如果需要将某些文件不需要在测试覆盖率中统计，则可以在相应的js文件前添加一行`/* istanbul ignore file */`

### VSCode

- [jest-runner](https://marketplace.visualstudio.com/items?itemName=firsttris.vscode-jest-runner)

  `jest-runner`支持在VSCode中运行某个测试，使用`react-scripts test`需要在VScode的`settings.json`中添加配置：

  ```
  "jestrunner.jestCommand": "npm run test --"
  ```

  配置好`jest-runner`后，进入某个测试文件，VSCode中在单元测试用例上显示`Debug|Run`的执行菜单，点击`Run`就可以直接在VSCode里单独运行某一个测试用例了。

## UI

对于后端开发背景的全栈开发人员来说，UI可能是很让人头疼的问题，不过还好的是，现在有很多开源的UI框架可用，提供了很多UI组件。市面上常用的有，如

- [React Bootstrap](https://react-bootstrap.github.io/)

- [React Elment UI](https://github.com/ElemeFE/element-react)

- [Material UI](https://material-ui.com/)

- [React Ant Design](https://ant.design/docs/react/getting-started)

### Material UI

在`Material-UI`的主站上，其介绍自己为：

```
React components for faster and easier web development. Build your own design system, or start with Material Design.
```

项目中我们选用了`Material UI`来搭建我们的UI框架。

```
yarn add @material-ui/core
```

如何使用`Material-UI`的组件呢，可以修改`App.js`为:

```
import React from 'react';
import { Button } from '@material-ui/core';
import './App.css';

function App() {
  return (
    <div className="App">
      <Button variant="contained" color="primary">
        Learn React
      </Button>
    </div>
  );
}

export default App;
```

`yarn start`打开页面[http://localhost:3000](http://localhost:3000)可以查看到一个只有一个按钮的页面

`Material-UI`提供的组件可参考[Material UI Components](https://material-ui.com/components/buttons/)

#### flex

`Material-UI`中大量使用flex的概念来进行布局，如[Grid](https://material-ui.com/components/grid/).

关于什么是flex，简单易懂的课程是[A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)

#### theme

为了让页面的配色比较统一，我们可以实现自己的一套主题颜色。`Material-UI`定义的默认主题的颜色可以参考[Material UI Theming](https://material-ui.com/customization/theming/)

创建`src/theme/typography.js`:

```
export default {
  h1: {
    fontWeight: 500,
    fontSize: 35,
    letterSpacing: '-0.24px',
  },
  h2: {
    fontWeight: 500,
    fontSize: 29,
    letterSpacing: '-0.24px',
  },
  h3: {
    fontWeight: 500,
    fontSize: 24,
    letterSpacing: '-0.06px',
  },
  h4: {
    fontWeight: 500,
    fontSize: 20,
    letterSpacing: '-0.06px',
  },
  h5: {
    fontWeight: 500,
    fontSize: 16,
    letterSpacing: '-0.05px',
  },
  h6: {
    fontWeight: 500,
    fontSize: 14,
    letterSpacing: '-0.05px',
  },
  overline: {
    fontWeight: 500,
  },
};
```

创建`src/theme/shadows.js`:

```
export default [
  'none',
  '0 0 0 1px rgba(63,63,68,0.05), 0 1px 2px 0 rgba(63,63,68,0.15)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 2px 2px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 3px 4px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 3px 4px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 4px 6px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 4px 6px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 4px 8px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 5px 8px -2px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 6px 12px -4px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 7px 12px -4px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 6px 16px -4px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 7px 16px -4px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 8px 18px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 9px 18px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 10px 20px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 11px 20px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 12px 22px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 13px 22px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 14px 24px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 16px 28px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 18px 30px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 20px 32px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 22px 34px -8px rgba(0,0,0,0.25)',
  '0 0 1px 0 rgba(0,0,0,0.31), 0 24px 36px -8px rgba(0,0,0,0.25)',
];
```

创建`src/theme/index.js`:

```
import { createMuiTheme, colors } from '@material-ui/core';
import shadows from './shadows';
import typography from './typography';

const theme = createMuiTheme({
  palette: {
    background: {
      dark: '#F4F6F8',
      default: colors.common.white,
      paper: colors.common.white,
    },
    primary: {
      main: colors.green[500],
    },
    secondary: {
      main: colors.green[500],
    },
    text: {
      primary: colors.blueGrey[900],
      secondary: colors.blueGrey[600],
    },
  },
  shadows,
  typography,
});

export default theme;
```

而后在项目的`src/App.js`中使用主题：

```
import React from 'react';
import PropTypes from 'prop-types';
import { Button, ThemeProvider, makeStyles } from '@material-ui/core';
import clsx from 'clsx';
import theme from './theme';

const useStyles = makeStyles(() => ({
  app: {
    textAlign: 'center',
  },
}));

const App = ({ className, ...rest }) => {
  const classes = useStyles();

  return (
    <ThemeProvider theme={theme}>
      <div className={clsx(className, classes.app)} {...rest}>
        <Button variant="contained" color="primary">
          Learn React
        </Button>
      </div>
    </ThemeProvider>
  );
};

App.propTypes = {
  className: PropTypes.string,
};

export default App;
```

可以看到，这里使用了`Material UI`提供的`ThemeProvider`来将引入我们自定义的主题。同时，我们可以通过`Material UI`提供的`makeStyles`函数来定义钩子`useStyles`。
而在`App`这个`Functional Component`内部，我们就可以通过`useStyles`拿到定义好的样式，进而把样式设置到相应的HTML元素上。


也修改`App.test.js`为:

```
import React from 'react';
import App from './App';
import { shallow } from 'enzyme';
import { ThemeProvider } from '@material-ui/core';

describe('App', () => {
  it('renders learn react link', () => {
    const wrapper = shallow(<App />);
    expect(wrapper.find(ThemeProvider).length).toEqual(1);
  });
});
```

> clsx和prop-types是通过`yarn add clsx prop-types`安装的

### Icons

`Material UI`本身有提供一套Icon库，通过`yarn add @material-ui/icons`就可以将icon添加进来。

不过本项目选用的是[react-feature](https://github.com/feathericons/react-feather)这个ICON库，通过[Simple feather icons](https://feathericons.com/)可以直观的搜索和查看图标。

`yarn add react-feather`添加依赖。

如在`src/App.js`的`Button`上添加图标：

```
import { Book as BookIcon } from 'react-feature';

<Button variant="contained" color="primary" startIcon={<BookIcon />}>
  Learn React
</Button>
```

## 支持不同环境

前端项目的打包方式可能与`Java`语言等打包方式略有不同。如`Java`程序所有的环境都可以使用一个Jar包，只需要在运行时将环境信息传入则不同环境可以使用不同的配置信息。
前端程序的所谓打包的主要目的，是将浏览器不识别的语法翻译成浏览器识别的语法的静态JS文件，这些JS文件生成后是无法将环境信息在运行时传入的。
这意味着对于JS前端程序来说，由于每个环境的配置是不同的，需要不同的静态JS文件。

### env文件

react-scripts本身支持在项目中定义`.env.*`文件，但是环境的类型是固定的，如`.env`，`.env.lcoal`，`.env.development`，`.env.test`，`.env.production`.

其认为只要是调用`build`相关命令，其就都是使用`.env.production`的配置。

具体的类容参考[Custom environment variables](https://create-react-app.dev/docs/adding-custom-environment-variables/)

### 环境变量

很显然，react-scripts的`.env.*`的方式不能满足我们的需求，现实中项目需要部署到`dev`,`staging`,`production`环境，意味着每个环境都要生成各自的JS静态文件包。

我们这里先`yarn add -D cross-env`引入`cross-env`使得在`package.json`里的命令中我们可以传入环境变量，其屏蔽了`Linux`操作系统下和`Windows`操作系统下传入环境变量方式的不同。

在`package.json`中我们可以为不同的`build`命令传入不同的自定义环境变量`REACT_APP_ENV`:

```
"build:prod": "cross-env REACT_APP_ENV=prod react-scripts build",
"build:dev": "cross-env REACT_APP_ENV=dev react-scripts build",
```

### 配置文件

- 创建`src/config/index.js`文件：

  ```
  const appEnv = process.env.REACT_APP_ENV || 'default';

  const defaultConfig = require('./config.default');

  const config = require(`./config.${appEnv}`);

  export default { ...defaultConfig, ...config };
  ```

- 创建`src/config/config.default.js`文件来定义默认配置：

  ```
  module.exports = {
    API_BASE_URL: 'http://localhost:4000',
  };
  ```

- 创建`src/config/config.dev.js`来定义`dev`环境的配置：

  ```
  module.exports = {
    API_BASE_URL: 'dev_url'
  };
  ```

- 创建`src/config/config.prod.js`来定义`prod`环境的配置。

- 在`src/App.js`中添加：

  ```
  import config from './config';

  <div>
  {config.API_BASE_URL}
  </div>
  ```

- 使用`yarn start`时候将使用`config.default.js`中的配置。


- 运行`yarn build:dev`将在`build`目录生成静态文件，而后`npx serve build`将启动一个静态文件的server, 访问[http://localhost:5000](http://localhost:5000)将看到其使用的是`config.dev.js`的配置。


- 运行`yarn build:prod`将在`build`目录生成静态文件，而后`npx serve build`将启动一个静态文件的server, 访问[http://localhost:5000](http://localhost:5000)将看到其使用的是`config.dev.js`的配置。

也可以将通过yarn命令来启动一个静态文件的sever，其比`npx serve build`命令会更快的启动一个server:

- `yarn add -D server`

- `package.json`中添加一个命令

  ```
  "serve": "serve build -l 5010"
  ```

- 调用`yarn serve`就可以访问静态文件server [http://localhost:5010](http://localhost:5010)


### 打包

上面的`build`命令会将`dev`环境和`prod`环境的静态文件都放到`build`目录下，如果我们需要在运行`build`相关命令时都打包成`zip`包该如何做呢。

- `yarn add -D mkdirp npm-build-zip` 添加相关依赖

- `package.json`中添加命令如

  ```
  "postbuild:prod": "mkdirp packages/prod && npm-build-zip --destination=packages/prod",
  "postbuild:dev": "mkdirp packages/dev && npm-build-zip --destination=packages/dev",
  ```

- `yarn build:dev`和`yarn build:prod`将分别将生成的静态文件打成zip包并放置在不同的目录下

### 生成版本文件

- `yarn add -D genversion`添加依赖

- `package.json`中添加命令

  ```
  "postversion": "genversion --semi --es6 src/lib/version.js",
  ```

- 创建`.yarnrc`，修改yarn的配置使得其在运行`yarn version`时不会自动创建git tags和commit

  ```
  version-git-tag false
  version-commit-hooks false
  ```

- 运行`yarn version`，输入新的版本后，将自动生成`src/lib/version.js`文件。


## 前端路由

React默认是单页面的，如果我们希望前端也支持多路由，需要引入`React Router`。

### react-router-dom

- `yarn add react-router@6.0.0-beta.0 react-router-dom@6.0.0-beta.0 history` 引入依赖，react-router-dom在V6上做了很多改进，具体可以参考[React Router v6 Preview](https://reacttraining.com/blog/react-router-v6-pre/)

- 创建`src/views/HomeView.js`

  ```
  import React from 'react';
  import PropTypes from 'prop-types';
  import { Button, makeStyles } from '@material-ui/core';
  import { Book as BookIcon } from 'react-feather';
  import clsx from 'clsx';
  import config from 'src/config';

  const useStyles = makeStyles(() => ({
    app: {
      textAlign: 'center',
    },
  }));

  const HomeView = ({ className, ...rest }) => {
    const classes = useStyles();

    return (
      <div className={clsx(className, classes.app)} {...rest}>
        <Button variant="contained" color="primary" startIcon={<BookIcon />}>
          Learn React
        </Button>
        {config.API_BASE_URL}
      </div>
    );
  };

  HomeView.propTypes = {
    className: PropTypes.string,
  };

  export default HomeView;
  ```

- 创建`src/views/VersionView.js`

  ```
  import React from 'react';
  import { version } from 'src/lib/version';

  const VersionView = () => {
    return <>{version}</>;
  };

  export default VersionView;
  ```

- 创建`src/views/NotFoundView.js`

- 创建`routes.js`

  ```
  import React from 'react';
  import { Navigate } from 'react-router-dom';
  import HomeView from 'src/views/HomeView';
  import VersionView from 'src/views/VersionView';
  import NotFoundView from 'src/views/NotFoundView';

  const routes = [
    { path: '/', element: <HomeView /> },
    { path: '/version', element: <VersionView /> },
    { path: '/404', element: <NotFoundView /> },
    { path: '*', element: <Navigate to="/404" /> },
  ];

  export default routes;
  ```

- 修改`src/App.js`

  ```
  import React from 'react';
  import { useRoutes } from 'react-router-dom';
  import { ThemeProvider } from '@material-ui/core';
  import theme from './theme';
  import routes from './routes';

  const App = () => {
    const routing = useRoutes(routes);

    return <ThemeProvider theme={theme}>{routing}</ThemeProvider>;
  };

  export default App;

  ```

- 创建`jsconfig.json`

  ```
  {
    "compilerOptions": {
      "baseUrl": "."
    },
    "include": [
      "src"
    ]
  }
  ```

- 修改`index.js`

  ```
  import React from 'react';
  import ReactDOM from 'react-dom';
  import { BrowserRouter } from 'react-router-dom';
  import App from './App';
  import * as serviceWorker from './serviceWorker';

  ReactDOM.render(
    <React.StrictMode>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </React.StrictMode>,
    document.getElementById('root')
  );

  serviceWorker.unregister();
  ```

- `yarn start` 后可以尝试访问不同的url: [http://localhost:3000/](http://localhost:3000/)，[http://localhost:3000/version](http://localhost:3000/version)，[http://localhost:3000/404](http://localhost:3000/404)， [http://localhost:3000/NOT_EXIST](http://localhost:3000/NOT_EXIST)

- 修改`package.json`中的`serve`命令，使得静态server也能处理不同的URL请求:

  ```
  "serve": "serve -s build -l 5010"
  ```

### Code Splitting

- 创建`src/components/LazyView.js`

  ```
  import React, { Suspense } from 'react';
  import PropTypes from 'prop-types';

  const LazyView = ({ children }) => {
    return <Suspense fallback={<div>loading</div>}>{children}</Suspense>;
  };

  LazyView.propTypes = {
    children: PropTypes.any,
  };

  export default LazyView;
  ```

- 修改`routes.js`:

  ```
  import React from 'react';
  import { Navigate } from 'react-router-dom';

  import LazyView from 'src/components/LazyView';

  const HomeView = React.lazy(() => import('src/views/HomeView'));
  const VersionView = React.lazy(() => import('src/views/VersionView'));
  const NotFoundView = React.lazy(() => import('src/views/NotFoundView'));

  const routes = [
    {
      path: '/',
      element: (
        <LazyView>
          <HomeView />
        </LazyView>
      ),
    },
    {
      path: '/version',
      element: (
        <LazyView>
          <VersionView />
        </LazyView>
      ),
    },
    {
      path: '/404',
      element: (
        <LazyView>
          <NotFoundView />
        </LazyView>
      ),
    },
    { path: '*', element: <Navigate to="/404" /> },
  ];

  export default routes;
  ```

运行`yarn build:dev`会发现有很多的小的静态`js`文件被创建出来。

关于React Lazy的介绍参考 [React.lazy](https://zh-hant.reactjs.org/docs/code-splitting.html#reactlazy)

### Layout

项目中往往多个页面共用同一个导航栏或者，这时我们引入Layout的概念。

- 创建`src/components/Logo.js`

  ```
  import React from 'react';
  import PropTypes from 'prop-types';

  const Logo = ({ className, ...rest }) => {
    return <img alt="Logo" src="/logo192.png" className={className} {...rest} />;
  };

  Logo.propTypes = {
    className: PropTypes.string,
  };

  export default Logo;
  ```

- 创建`src/layout/MainLayout/Header.js`

  ```
  import React from 'react';
  import clsx from 'clsx';
  import PropTypes from 'prop-types';
  import { Box, makeStyles } from '@material-ui/core';
  import Logo from 'src/components/Logo';

  const useStyles = makeStyles((theme) => ({
    root: {
      marginTop: theme.spacing(1),
      borderBottom: `1px solid ${theme.palette.background.dark}`,
    },
    logo: {
      width: '30px',
      marginLeft: theme.spacing(1),
    },
  }));

  const Header = ({ className, ...rest }) => {
    const classes = useStyles();

    return (
      <Box className={clsx(classes.root, className)} {...rest}>
        <Logo className={classes.logo} />
      </Box>
    );
  };

  Header.propTypes = {
    className: PropTypes.string,
  };

  export default Header;
  ```

- 创建 `src/layout/MainLayout/index.js`文件

  ```
  import React from 'react';
  import { Outlet } from 'react-router-dom';
  import { makeStyles } from '@material-ui/core';
  import Header from './Header';

  const useStyles = makeStyles(theme => ({
    root: {
      backgroundColor: theme.palette.background.default,
      display: 'flex',
      flexDirection: 'column',
      height: '100%',
      width: '100%',
    },
    header: {
      width: '100%',
    },
    context: {
      width: '100%',
    },
  }));

  const MainLayout = () => {
    const classes = useStyles();

    return (
      <div className={classes.root}>
        <Header className={classes.header} />
        <div className={classes.context}>
          <Outlet />
        </div>
      </div>
    );
  };

  export default MainLayout;  
  ```

- 创建`src/components/GlobalStyles.js`

  ```
  import { createStyles, makeStyles } from '@material-ui/core';

  const useStyles = makeStyles(() =>
    createStyles({
      '@global': {
        '*': {
          boxSizing: 'border-box',
          margin: 0,
          padding: 0,
        },
        html: {
          '-webkit-font-smoothing': 'antialiased',
          '-moz-osx-font-smoothing': 'grayscale',
          height: '100%',
          width: '100%',
        },
        body: {
          backgroundColor: '#f4f6f8',
          height: '100%',
          width: '100%',
        },
        a: {
          textDecoration: 'none',
        },
        '#root': {
          height: '100%',
          width: '100%',
        },
      },
    })
  );

  const GlobalStyles = () => {
    useStyles();

    return null;
  };

  export default GlobalStyles;
  ```

- 修改`routes.js`

  ```
  import React from 'react';
  import { Navigate } from 'react-router-dom';
  import MainLayout from 'src/layouts/MainLayout';

  ...

  const routes = [
    {
      path: '/',
      element: <MainLayout />,
      children: [
        {
          path: '/',
          element: (
            <LazyView>
              <HomeView />
            </LazyView>
          ),
        },
        ....
      ],
    },
  ];

  export default routes;  
  ```

### Page Header

不同的页面应该使用不同的`title`，这里需要引入`react-helmet`来解决这个问题：

- `yarn add react-helmet`引入依赖

- 创建`src/components/Page.js`

  ```
  import React, { forwardRef } from 'react';
  import { Helmet } from 'react-helmet';
  import PropTypes from 'prop-types';

  const Page = forwardRef(({ children, title = '', ...rest }, ref) => {
    return (
      <div ref={ref} {...rest}>
        <Helmet>
          <title>{title}</title>
        </Helmet>
        {children}
      </div>
    );
  });

  Page.propTypes = {
    children: PropTypes.node.isRequired,
    title: PropTypes.string,
  };

  export default Page;
  ```

- 修改`src/views/HomeView`

  ```
  import Page from 'src/components/Page';

  <Page title="Home">

  </Page>
  ```

  类似的方式修改`src/views/VersionView`和`src/views/NotFoundView`.

## 状态管理

### redux

#### react-redux


#### redux toolkit

## 异步请求

### axios


## 表单

## 工具库

### lodash



## 总结





## 参考文章

- [Introducing npx: an npm package runner](https://blog.npmjs.org/post/162869356040/introducing-npx-an-npm-package-runner)

- [React Testing Library](https://github.com/testing-library/react-testing-library)

- [Introducing Hooks](https://reactjs.org/docs/hooks-intro.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
