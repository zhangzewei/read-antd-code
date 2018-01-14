# Notification
这是一个全局变量的组件，可以在任意地方调用其函数就能够生成一个，我们就来看看这个组件又是用了什么奇巧淫技来实现的

-- 注意：解读的源码版本为2.13.4 rc-notification版本为2.0.0 不要下载错了

## 本节讲点

1. 查看notification组件源码的文件顺序和入口点
2. rc-utils组件中的createChainedFunction函数
3. 缓存机制
4. ReactDOM.unmountComponentAtNode

## 快速阅读代码

我将带大家使用`略览`代码的方法来进行一个组件的快速通读，这就跟高中英语阅读时使用的一种阅读方法一样，快速阅读，略过细节，抓主线路，理清整个组件工作原理之后再去查看细节

1. antd-design-master/components/index.tsx
    
    因为使用方法是直接使用的notification.api(config)，所以想到先去看看是怎么抛出的
    `export { default as notification } from './notification'`

2. antd-design-master/components/notification/index.tsx

    再看看引用的文件是怎么抛出的
    `export default api as NotificationApi;`

3. antd-design-master/components/notification/index.tsx

    由下往上看代码，看到`api`的构成，再看到`api.notice`->`function notice`->`function getNotificationInstance`->`(Notification as any).newInstance`->`import Notification from 'rc-notification';`

    ```js
        getNotificationInstance(
          outerPrefixCls,
          args.placement || defaultPlacement
        ).notice({
          content: (
            <div className={iconNode ? `${prefixCls}-with-icon` : ''}>
              {iconNode}
              <div className={`${prefixCls}-message`}>
                {autoMarginTag}
                {args.message}
              </div>
              <div className={`${prefixCls}-description`}>{args.description}</div>
              {args.btn ? <span className={`${prefixCls}-btn`}>{args.btn}</span> : null}
            </div>
          ),
          duration,
          closable: true,
          onClose: args.onClose,
          key: args.key,
          style: args.style || {},
          className: args.className,
        })
    ```

    在这个文件中比较重要的一条代码线就是上面展示的这一条，剩下的代码可以一眼带过，比较特殊的就是他将生成的notification实例都存在一个全局常量中，方便第二次使用只要这个实例没有被destroy

4. rc-notification/src/index.js

    找到入口文件`import Notification from './Notification';`

5. rc-notification/src/Notification.jsx

    在上面第3条我们看到有的一个方法`newInstance`是用来创建新实例，所以我们在这个文件中也可以看到相应的代码`Notification.newInstance = function newNotificationInstance`，在这个函数中我们继续略览代码，看到`ReactDOM.render(<Notification {...props} ref={ref} />, div);`我们知道这是将一个组件渲染在一个dom节点，所以下一个查看点就应该是`Notification`这个组件类

6. rc-notification/src/Notification.jsx

    看到文件上面`class Notification extends Component`，可以看到整个组件的实现，我们可以在`render`函数中看到一个循环输出，那就是在循环输出`state`中存的`notice`，`state`中的`notice`是通过上面第3点展示的代码，获取实例之后使用`notice`函数调用的实例的`add`函数进行添加的

    ```js
      const onClose = createChainedFunction(this.remove.bind(this, notice.key), notice.onClose);
      return (<Notice
        prefixCls={props.prefixCls}
        {...notice}
        onClose={onClose}
      >
        {notice.content}
      </Notice>);
    ```
7. rc-notification/src/Notice.jsx

    ```js
      componentDidMount() {
        if (this.props.duration) {
          this.closeTimer = setTimeout(() => {
            this.close();
          }, this.props.duration * 1000);
        }
      }

      componentWillUnmount() {
        this.clearCloseTimer();
      }

      clearCloseTimer = () => {
        if (this.closeTimer) {
          clearTimeout(this.closeTimer);
          this.closeTimer = null;
        }
      }

      close = () => {
        this.clearCloseTimer();
        this.props.onClose();
      }
    ```

    这个文件中玄妙之处其实在于以上三个函数，在`componentDidMount`之时，添加了一个定时器，将在规定时间之后删除掉当前的这个提示窗，并且这个删除动作是交由给外层文件去删除当前这个提示框的实例进行的也就是第6点文件中的`remove`函数，在最新的（3.0.0）rc-notification中添加了以下代码，为了能够在鼠标移上去之后不让消息框消失，增加了用户体验度

    ```js
      componentDidMount() {
        this.startCloseTimer();
      }

      componentWillUnmount() {
        this.clearCloseTimer();
      }

      close = () => {
        this.clearCloseTimer();
        this.props.onClose();
      }

      startCloseTimer = () => {
        if (this.props.duration) {
          this.closeTimer = setTimeout(() => {
            this.close();
          }, this.props.duration * 1000);
        }
      }

      clearCloseTimer = () => {
        if (this.closeTimer) {
          clearTimeout(this.closeTimer);
          this.closeTimer = null;
        }
      }

      render() {
        const props = this.props;
        const componentClass = `${props.prefixCls}-notice`;
        const className = {
          [`${componentClass}`]: 1,
          [`${componentClass}-closable`]: props.closable,
          [props.className]: !!props.className,
        };
        return (
          <div className={classNames(className)} style={props.style} onMouseEnter={this.clearCloseTimer}
            onMouseLeave={this.startCloseTimer}
          >
            <div className={`${componentClass}-content`}>{props.children}</div>
              {props.closable ?
                <a tabIndex="0" onClick={this.close} className={`${componentClass}-close`}>
                  <span className={`${componentClass}-close-x`}></span>
                </a> : null
              }
          </div>
        );
      }
    ```

## CreateChainedFunction

这个函数是使用在上面第6点，目的是为了能够删除当前的notification的缓存值，然后再执行外部传入的关闭回调函数，这个函数的实现在`rc-util`包中，这个包中有很多的方法是值得学习的，但是他在github上面的star数量却只有73个，这里软推一下吧。

```js
  export default function createChainedFunction() {
    const args = [].slice.call(arguments, 0);
    if (args.length === 1) {
      return args[0];
    }

    return function chainedFunction() {
      for (let i = 0; i < args.length; i++) {
        if (args[i] && args[i].apply) {
          args[i].apply(this, arguments);
        }
      }
    };
  }
```

这个函数中使用了`call`来将传入的参数变成一个数组，然后使用`apply`将传入的函数一一执行，这样子就能够实现一个函数接受多个函数，然后按照顺序执行，并且在第6点的代码中`this.remove.bind(this, notice.key)`使用了`bind`函数制定了this和传入参数，方法很精妙也很经典。

## 缓存机制

`notification`组件在`ant-design-master`中使用了

```js
const notificationInstance = {};

destroy() {
  Object.keys(notificationInstance).forEach(cacheKey => {
    notificationInstance[cacheKey].destroy();
    delete notificationInstance[cacheKey];
  });
}
```
来进行对创建实例的缓存，然后在销毁时将缓存的实例删除

在`notification 2.0.0`中也使用了缓存机制

```js
add = (notice) => {
  const key = notice.key = notice.key || getUuid();
  this.setState(previousState => {
    const notices = previousState.notices;
    if (!notices.filter(v => v.key === key).length) {
      return {
        notices: notices.concat(notice),
      };
    }
  });
}

remove = (key) => {
  this.setState(previousState => {
    return {
      notices: previousState.notices.filter(notice => notice.key !== key),
    };
  });
}
```
在这个代码中看到这个缓存机制是使用的数组的方式实现的，但是在外层封装却是用的是是对象的方式实现，我猜想这两个代码不是一个人写的。。。代码风格不同意呢。

## ReactDOM.unmountComponentAtNode

```js
Notification.newInstance = function newNotificationInstance(properties) {
  const { getContainer, ...props } = properties || {};
  let div;
  if (getContainer) {
    div = getContainer();
  } else {
    div = document.createElement('div');
    document.body.appendChild(div);
  }
  const notification = ReactDOM.render(<Notification {...props} />, div);
  return {
    notice(noticeProps) {
      notification.add(noticeProps);
    },
    removeNotice(key) {
      notification.remove(key);
    },
    component: notification,
    destroy() {
      ReactDOM.unmountComponentAtNode(div);
      document.body.removeChild(div);
    },
  };
};
```

从上面的代码中看出，`notification`组件使用`unmountComponentAtNode`函数将其进行销毁，这个方法适用于某些不能在当前组件中进行组件销毁的情况。