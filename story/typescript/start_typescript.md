# 开始Typescript

## 前言

由于看Antd源码，都是使用的typescript编写的，在有的时候看代码确实看不懂，需要一步一步的去验证求证一些东西，所以索性也就看了看怎么样开始一个typescript项目。

## 需要准备什么？

1. [Typescript基础语法知识](https://www.tslang.cn/docs/handbook/basic-types.html)
2. [create-react-app with typescript](https://github.com/Microsoft/TypeScript-React-Starter)

## 开始

打开终端输入以下命令（前提是你已经安装好了`create-react-app`）

```
create-react-app my-app --scripts-version=react-scripts-ts && cd my-app
```

这样我们就构建好一个react-typescript项目了，按照提示，在命令行输入`yarn start || npm start`就能够运行项目了

## 编写Hello Typescript组件

了解了typescript的基本语法之后，就能够编写一个简单的组件了

```js
  import * as React from 'react';
  import * as PropTypes from 'prop-types';
  import { MyDecorator } from './Decorator';

  // 这里是定义传入的参数类型
  export interface DemoProps {
    helloString?: string;
  }

  // 这里写一个类 对其传入参数类型也是有定义的第一个参数是props，第二个是state
  // props就是用刚才上面我们定义的那样，state如果不传就写一个any就好了
  export default class DecoratorTest extends React.Component<DemoProps, any> {
    static propTypes = {
      helloString: PropTypes.string,
    };

    constructor(props) {
      super(props);
    }

    // 这是一个装饰器，下面会附上装饰器代码，装饰的作用就是给callDecorator
    // 这个属性添加一个cancel属性
    // 如果不是很懂装饰器的同学可以去typescript官网上查看详细文档
    @MyDecorator()
    callDecorator() {
      console.log('I am in callDecorator');
    }

    componentDidMount() {
      this.callDecorator();
      (this.callDecorator as any).cancel();
    }

    render() {
      return (
        <div>
          {this.props.helloString}
        </div>
      );
    }
  }

  // 装饰器代码
  // 在写装饰器代码的时候需要在tsconfig.json文件中的compilerOptions属性添加一下代码
  // "experimentalDecorators": true 
  export default function decoratorTest(fn) {
    console.log('in definingProperty');
    const throttled = () => {
      fn();
    };

    (throttled as any).cancel = () => console.log('cancel');

    return throttled;
  }

  export function MyDecorator() {
    return function(target, key, descriptor) {
      let fn = descriptor.value;
      let definingProperty = false;
      console.log('before definingProperty');
      // 这个get函数会在类被实例化的时候就进行调用，所以就能够将这些属性赋给外部的target
      // 也就是在this.callDecorator的时候
      // 顺带说一下set函数 会在 this.callDecorator = something 的时候调用
      return {
        configurable: true,
        // get: function()这样的写法也是可以执行
        get() {
          if (definingProperty || this === target.prototype || this.hasOwnProperty(key)) {
            return fn;
          }
          let boundFn = decoratorTest(fn.bind(this));
          definingProperty = true;
          Object.defineProperty(this, key, {
            value: boundFn,
            configurable: true,
            writable: true,
          });
          definingProperty = false;
          return boundFn;
        },
      };
    };
  }
```

组件写完之后就可以在外部的`App.tsx`文件中引用了

```js
  import * as React from 'react';
  import Demo from './components/Demo';
  import './App.css';

  const logo = require('./logo.svg');

  class App extends React.Component {
    render() {
      return (
        <div className="App">
          <div className="App-header">
            <img src={logo} className="App-logo" alt="logo" />
            <h2>Welcome to React</h2>
          </div>
          <div className="App-intro">
            <Demo helloString="Hello, react with typescript" />
          </div>
        </div>
      );
    }
  }

  export default App;
```

然后转向浏览器 查看我们写的组件是否展示出来了,如果没有展示出来，可以去终端查看编译错误。

![hello](../images/hello_tp.png)