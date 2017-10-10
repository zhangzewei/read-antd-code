> github: [地址](https://github.com/zhangzewei/read-antd-code)
> gitbook: [地址](https://zhangzewei.gitbooks.io/read-antd-code/content/)

# Index.js
看一个代码的时候首先当然是从他的入口文件开始看起，所以第一份代码我们看的是`/index.js`文件

# 开始
打开`index.js`文件，代码只有28行，其中包含了一个`camelCase`函数（看函数名就知道这是个给名称进行驼峰命名法的函数），一个`req`变量，以及这个的变量操作和`export`操作

在这个文件里面我首先查了`require.context()`这个函数的使用，可以参考[这里](https://juejin.im/entry/590c2777128fe10058392598)，以及`exports`和`module.exports`的区别，可以参考[这里](https://cnodejs.org/topic/5231a630101e574521e45ef8)，这里是一些铺垫，下面进入正题

通过上面两个铺垫，我们知道了`req`这个变量是用来循环抛出组件的一个对象，并且还抛出了每一个组件的样式文件

```js 
  // index.js
  function camelCase(name) {
    return name.charAt(0).toUpperCase() +
      name.slice(1).replace(/-(\w)/g, (m, n) => {
        return n.toUpperCase();
      });
  }

  // 抛出样式 这个正则是匹配当前目录下的所有的/style/index.tsx文件
  const req = require.context('./components', true, /^\.\/[^_][\w-]+\/style\/index\.tsx?$/);

  req.keys().forEach((mod) => {
    let v = req(mod);
    if (v && v.default) {
      v = v.default;
    }
    // 抛出组件 这个正则是匹配当前目录下的素有index.tsx文件
    const match = mod.match(/^\.\/([^_][\w-]+)\/index\.tsx?$/);
    if (match && match[1]) {
      if (match[1] === 'message' || match[1] === 'notification') {
        // message & notification should not be capitalized
        exports[match[1]] = v;
      } else {
        exports[camelCase(match[1])] = v;
      }
    }
  });

  module.exports = require('./components');
```

但是最后不知道为甚还需要加上对吼那一句`module.exports = require('./components');`
既然上面都已经抛出，为什么这里还需要再次抛出，不过好像是跟什么环境和打包之后的一些操作有关，所以这里一两次抛出。这个地方还需要向大家请教。

