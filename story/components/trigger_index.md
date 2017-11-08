# Trigger 

这个组件的index文件就有很多代码，590行代码，而且在头部引入的额外文件特别的多，所以我们这一个组件就先从这些额外的组件中开始吧，先看看这些外部方法能够做些什么。

> 强烈建议把tigger的代码下载下来自行查看，因为实在是太长了

```js
// index.js 头部
  import PropTypes from 'prop-types';
  import { findDOMNode, createPortal } from 'react-dom';
  import createReactClass from 'create-react-class';
  import contains from 'rc-util/lib/Dom/contains';
  import addEventListener from 'rc-util/lib/Dom/addEventListener';
  import Popup from './Popup';
  import { getAlignFromPlacement, getPopupClassNameFromAlign } from './utils';
  import getContainerRenderMixin from 'rc-util/lib/getContainerRenderMixin';
  import Portal from 'rc-util/lib/Portal';
```

## createPortal 

在官网[这里](https://reactjs.org/docs/react-dom.html#createportal)有这么一个解释

```js
ReactDOM.createPortal(child, container)
```

`Creates a portal. Portals provide a way to render children into a DOM node that exists outside the hierarchy of the DOM component.`

这个函数是用来创建一个`portal`，而这个`Portal`是提供一个方法来在指定的dom元素渲染一些组件的方法。

## createReactClass

这个函数也是能够在官网[这里](https://reactjs.org/docs/react-without-es6.html)上找到的，是用来创建一个raect类而不是用es6语法的方法，在里面可以使用`getDefaultProps()`方法

创建当前组件的默认props，可以使用`getInitialState()`创建当前组件的初始state，并且在里面写的方法都会自动的绑定上`this`,

也就是他所说的`Autobinding`，还有一个最有用的属性`Mixins`，这个是能够在编写很多的能够使用的外部方法传入组件的属性。

## contains && addEventListener

这两个函数都是`rc-util/lib/Dom/`里面的工具函数，接下来我们分辨看看这两个函数能够做啥

```js
// contains.js

// 这个函数是用来判断传入根节点root是否包含传入节点n，
// 如果包含则返回true，否者返回false
  export default function contains(root, n) {
    let node = n;
    while (node) {
      if (node === root) {
        return true;
      }
      node = node.parentNode;
    }

    return false;
  }
```

```js
// addEventListener.js
// 这个函数主要的聚焦点是ReactDOM.unstable_batchedUpdates
// 这个api是没有公开的一个api，但是可以使用，为了是想要将当前的组件状态强制性的
// 更新到组件内部去并且，但是这样做的目的可能有点粗暴。。
// 想要了解的可以看这篇文章，或许你有新的想法
// https://zhuanlan.zhihu.com/p/20328570
  import addDOMEventListener from 'add-dom-event-listener';
  import ReactDOM from 'react-dom';

  export default function addEventListenerWrap(target, eventType, cb) {
    /* eslint camelcase: 2 */
    const callback = ReactDOM.unstable_batchedUpdates ? function run(e) {
      ReactDOM.unstable_batchedUpdates(cb, e);
    } : cb;
    return addDOMEventListener(target, eventType, callback);
  }
```

## getContainerRenderMixin && Portal

接下来是这两个函数，都是来自于`rc-util/lib/`

```js
// getContainerRenderMixin.js

  import ReactDOM from 'react-dom';

  function defaultGetContainer() {
    const container = document.createElement('div');
    document.body.appendChild(container);
    return container;
  }

  export default function getContainerRenderMixin(config) {
    const {
      autoMount = true,
      autoDestroy = true,
      isVisible,
      getComponent,
      getContainer = defaultGetContainer,
    } = config;

    let mixin;

    function renderComponent(instance, componentArg, ready) {
      if (!isVisible || instance._component || isVisible(instance)) {
        // 如果有isVisible，并且传入的实例有_component，并且isVisible返回真则进行一下代码
        if (!instance._container) {
          // 如果传入实例没有_container，则为其添加一个默认的
          instance._container = getContainer(instance);
        }
        let component;
        if (instance.getComponent) {
          // 如果传入实例有getComponent，则将传入的参数传入实例的getComponent函数
          component = instance.getComponent(componentArg);
        } else {
          // 否则就进行就是用传入参数中的getComponent方法构造一个Component
          component = getComponent(instance, componentArg);
        }
        // unstable_renderSubtreeIntoContainer是更新组件到传入的DOM节点上
        // 可以使用它完成在组件内部实现跨组件的DOM操作
        // ReactComponent unstable_renderSubtreeIntoContainer(
        //    parentComponent component,
        //    ReactElement element,
        //    DOMElement container,
        //    [function callback]
        //  )
        ReactDOM.unstable_renderSubtreeIntoContainer(instance,
          component, instance._container,
          function callback() {
            instance._component = this;
            if (ready) {
              ready.call(this);
            }
          });
      }
    }

    if (autoMount) {
      mixin = {
        ...mixin,
        // 如果是自动渲染组件，那就在DidMount和DidUpdate渲染组件
        componentDidMount() {
          renderComponent(this);
        },
        componentDidUpdate() {
          renderComponent(this);
        },
      };
    }

    if (!autoMount || !autoDestroy) {
      mixin = {
        // 如果不是自动渲染的，那就在mixin中添加一个渲染函数
        ...mixin,
        renderComponent(componentArg, ready) {
          renderComponent(this, componentArg, ready);
        },
      };
    }

    function removeContainer(instance) {
      // 用于在挂载节点remove掉添加的组件
      if (instance._container) {
        const container = instance._container;
        // 先将组件unmount
        ReactDOM.unmountComponentAtNode(container);
        // 然后在删除挂载点
        container.parentNode.removeChild(container);
        instance._container = null;
      }
    }

    if (autoDestroy) {
      // 如果是自动销毁的，那就在WillUnmount的时候销毁
      mixin = {
        ...mixin,
        componentWillUnmount() {
          removeContainer(this);
        },
      };
    } else {
      mixin = {
        // 如果不是自动销毁，那就只是在mixin中添加一个销毁的函数
        ...mixin,
        removeContainer() {
          removeContainer(this);
        },
      };
    }
    // 最后返回构建好的mixin
    return mixin;
  }
```

```js
// Portal.js
// 这个函数就像我们刚才上面所提到的Potal组件的一个编写，这样的组件非常有用
// 我们可以利用这个组件创建在一些我们所需要创建组件的地方，比如在body节点创建
// 模态框，或者在窗口节点创建fixed的定位的弹出框之类的。
// 还有就是在用完这个组件也就是在componentWillUnmount的时候一定要将节点移除
  import React from 'react';
  import PropTypes from 'prop-types';
  import { createPortal } from 'react-dom';

  export default class Portal extends React.Component {
    static propTypes = {
      getContainer: PropTypes.func.isRequired,
      children: PropTypes.node.isRequired,
    }

    componentDidMount() {
      this.createContainer();
    }

    componentWillUnmount() {
      this.removeContainer();
    }

    createContainer() {
      this._container = this.props.getContainer();
      this.forceUpdate();
    }

    removeContainer() {
      if (this._container) {
        this._container.parentNode.removeChild(this._container);
      }
    }

    render() {
      if (this._container) {
        return createPortal(this.props.children, this._container);
      }
      return null;
    }
  }
```

## 在组件开始之前

在组件开始之前还有一些辅助的东西需要了解到

```js
  // 函数体的默认值
  function noop() {
  }

  function returnEmptyString() {
    return '';
  }

  function returnDocument() {
    return window.document;
  }
  // 设置允许的事件，onContextMenu是右键菜单事件
  const ALL_HANDLERS = ['onClick', 'onMouseDown', 'onTouchStart', 'onMouseEnter',
    'onMouseLeave', 'onFocus', 'onBlur', 'onContextMenu'];
  // 判断一下react的版本是不是react16
  const IS_REACT_16 = !!createPortal;
  // 判断是否是手机查看，
  // Navigator 对象包含有关浏览器的信息。 详情可以看这里http://www.w3school.com.cn/jsref/dom_obj_navigator.asp
  // 这里判断一下浏览器代理是不是移动端的代理。
  const isMobile = typeof navigator !== 'undefined' && !!navigator.userAgent.match(
    /(Android|iPhone|iPad|iPod|iOS|UCWEB)/i
  );

  const mixins = [];
  // 判断一下，如果不是react16，就在mixin中自己添加一个类似于createPortal的函数
  if (!IS_REACT_16) {
    mixins.push(
      getContainerRenderMixin({
        autoMount: false,

        isVisible(instance) {
          return instance.state.popupVisible;
        },

        getContainer(instance) {
          return instance.getContainer();
        },
      })
    );
  }
```
## Props

这个组件的传入参数非常的多，为了做兼容或者适应更多的使用者。
```js
  propTypes: {
    children: PropTypes.any,
    // 还记得我在dropdown里面留下的问题么，当时我问的是为什
    // 么触发可以试试一个数组，这里这个参数将会告诉你为什么，
    // 是可以让写在数组中的事件都成为其触发的事件。
    action: PropTypes.oneOfType([PropTypes.string, PropTypes.arrayOf(PropTypes.string)]),
    showAction: PropTypes.any,
    hideAction: PropTypes.any,
    getPopupClassNameFromAlign: PropTypes.any,
    onPopupVisibleChange: PropTypes.func,
    afterPopupVisibleChange: PropTypes.func,
    popup: PropTypes.oneOfType([
      PropTypes.node,
      PropTypes.func,
    ]).isRequired,
    popupStyle: PropTypes.object,
    prefixCls: PropTypes.string,
    popupClassName: PropTypes.string,
    popupPlacement: PropTypes.string,
    builtinPlacements: PropTypes.object,
    popupTransitionName: PropTypes.oneOfType([
      PropTypes.string,
      PropTypes.object,
    ]),
    popupAnimation: PropTypes.any,
    mouseEnterDelay: PropTypes.number,
    mouseLeaveDelay: PropTypes.number,
    zIndex: PropTypes.number,
    focusDelay: PropTypes.number,
    blurDelay: PropTypes.number,
    getPopupContainer: PropTypes.func,
    getDocument: PropTypes.func,
    destroyPopupOnHide: PropTypes.bool,
    mask: PropTypes.bool,
    maskClosable: PropTypes.bool,
    onPopupAlign: PropTypes.func,
    popupAlign: PropTypes.object,
    popupVisible: PropTypes.bool,
    maskTransitionName: PropTypes.oneOfType([
      PropTypes.string,
      PropTypes.object,
    ]),
    maskAnimation: PropTypes.string,
  }
```

参数很多，我直接将其参数作用拷贝过来了。

<table class="table table-bordered table-striped">
    <thead>
    <tr>
        <th style="width: 100px;">name</th>
        <th style="width: 50px;">type</th>
        <th style="width: 50px;">default</th>
        <th>description</th>
    </tr>
    </thead>
    <tbody>
        <tr>
          <td>popupClassName</td>
          <td>string</td>
          <td></td>
          <td>additional className added to popup</td>
        </tr>
        <tr>
          <td>destroyPopupOnHide</td>
          <td>boolean</td>
          <td>false</td>
          <td>whether destroy popup when hide</td>
        </tr>
        <tr>
          <td>getPopupClassNameFromAlign</td>
          <td>getPopupClassNameFromAlign(align: Object):String</td>
          <td></td>
          <td>additional className added to popup according to align</td>
        </tr>
        <tr>
          <td>action</td>
          <td>string[]</td>
          <td>['hover']</td>
          <td>which actions cause popup shown. enum of 'hover','click','focus','contextMenu'</td>
        </tr>
        <tr>
          <td>mouseEnterDelay</td>
          <td>number</td>
          <td>0</td>
          <td>delay time to show when mouse enter. unit: s.</td>
        </tr>
        <tr>
          <td>mouseLeaveDelay</td>
          <td>number</td>
          <td>0.1</td>
          <td>delay time to hide when mouse leave. unit: s.</td>
        </tr>
        <tr>
          <td>popupStyle</td>
          <td>Object</td>
          <td></td>
          <td>additional style of popup</td>
        </tr>
        <tr>
          <td>prefixCls</td>
          <td>String</td>
          <td>rc-trigger-popup</td>
          <td>prefix class name</td>
        </tr>
        <tr>
          <td>popupTransitionName</td>
          <td>String|Object</td>
          <td></td>
          <td>https://github.com/react-component/animate</td>
        </tr>
        <tr>
          <td>maskTransitionName</td>
          <td>String|Object</td>
          <td></td>
          <td>https://github.com/react-component/animate</td>
        </tr>
        <tr>
          <td>onPopupVisibleChange</td>
          <td>Function</td>
          <td></td>
          <td>call when popup visible is changed</td>
        </tr>
        <tr>
          <td>mask</td>
          <td>boolean</td>
          <td>false</td>
          <td>whether to support mask</td>
        </tr>
        <tr>
          <td>maskClosable</td>
          <td>boolean</td>
          <td>true</td>
          <td>whether to support click mask to hide</td>
        </tr>
        <tr>
          <td>popupVisible</td>
          <td>boolean</td>
          <td></td>
          <td>whether popup is visible</td>
        </tr>
        <tr>
          <td>zIndex</td>
          <td>number</td>
          <td></td>
          <td>popup's zIndex</td>
        </tr>
        <tr>
          <td>defaultPopupVisible</td>
          <td>boolean</td>
          <td></td>
          <td>whether popup is visible initially</td>
        </tr>
        <tr>
          <td>popupAlign</td>
          <td>Object: alignConfig of [dom-align](https://github.com/yiminghe/dom-align)</td>
          <td></td>
          <td>popup 's align config</td>
        </tr>
        <tr>
          <td>onPopupAlign</td>
          <td>function(popupDomNode, align)</td>
          <td></td>
          <td>callback when popup node is aligned</td>
        </tr>
        <tr>
          <td>popup</td>
          <td>React.Element | function() => React.Element</td>
          <td></td>
          <td>popup content</td>
        </tr>
        <tr>
          <td>getPopupContainer</td>
          <td>getPopupContainer(): HTMLElement</td>
          <td></td>
          <td>function returning html node which will act as popup container</td>
        </tr>
        <tr>
          <td>getDocument</td>
          <td>getDocument(): HTMLElement</td>
          <td></td>
          <td>function returning document node which will be attached click event to close trigger</td>
        </tr>
        <tr>
          <td>popupPlacement</td>
          <td>string</td>
          <td></td>
          <td>use preset popup align config from builtinPlacements, can be merged by popupAlign prop</td>
        </tr>
        <tr>
          <td>builtinPlacements</td>
          <td>object</td>
          <td></td>
          <td>builtin placement align map. used by placement prop</td>
        </tr>
    </tbody>
</table>


## Render() 

我们依然还是从他的render函数作为突破点

```js
  render() {
    const { popupVisible } = this.state;
    const props = this.props;
    const children = props.children;
    // react.children.only 是检查是否只包含一个孩子节点，否则这个函数抛出错误
    const child = React.Children.only(children);
    // 这里添加key这个属性为了在后面返回数组的时候能够有一个key
    const newChildProps = { key: 'trigger' };

    // 下面的所有的操作是给传出的trigger绑定事件

    if (this.isContextMenuToShow()) {
      newChildProps.onContextMenu = this.onContextMenu;
    } else {
      newChildProps.onContextMenu = this.createTwoChains('onContextMenu');
    }

    if (this.isClickToHide() || this.isClickToShow()) {
      newChildProps.onClick = this.onClick;
      newChildProps.onMouseDown = this.onMouseDown;
      newChildProps.onTouchStart = this.onTouchStart;
    } else {
      newChildProps.onClick = this.createTwoChains('onClick');
      newChildProps.onMouseDown = this.createTwoChains('onMouseDown');
      newChildProps.onTouchStart = this.createTwoChains('onTouchStart');
    }
    if (this.isMouseEnterToShow()) {
      newChildProps.onMouseEnter = this.onMouseEnter;
    } else {
      newChildProps.onMouseEnter = this.createTwoChains('onMouseEnter');
    }
    if (this.isMouseLeaveToHide()) {
      newChildProps.onMouseLeave = this.onMouseLeave;
    } else {
      newChildProps.onMouseLeave = this.createTwoChains('onMouseLeave');
    }
    if (this.isFocusToShow() || this.isBlurToHide()) {
      newChildProps.onFocus = this.onFocus;
      newChildProps.onBlur = this.onBlur;
    } else {
      newChildProps.onFocus = this.createTwoChains('onFocus');
      newChildProps.onBlur = this.createTwoChains('onBlur');
    }
    // 利用新的props构建一个新的trigger
    const trigger = React.cloneElement(child, newChildProps);

    // 判断是否是react16版本 不是就直接返回trigger
    if (!IS_REACT_16) {
      return trigger;
    }

    let portal;
    // prevent unmounting after it's rendered
    if (popupVisible || this._component) {
      portal = (
        <Portal
          key="portal"
          getContainer={this.getContainer}
        >
          {this.getComponent()}
        </Portal>
      );
    }

    return [
      trigger,
      portal,
    ];
  },
```

在上面的代码中我们看这些函数

```js
  this.createTwoChains();

  this.isContextMenuToShow();

  this.isClickToHide();

  this.isClickToShow();

  this.isMouseEnterToShow();

  this.isMouseLeaveToHide();

  this.isFocusToShow();

  this.isBlurToHide();
```

那么我们将来了解这些函数都干了什么

+ ### this.createTwoChains()
  ```js
    // 这个函数是给trigger组件绑定对应事件
    createTwoChains(event) {
      // 获取包裹元素的props
      const childPros = this.props.children.props;
      // 获取当前组件的props
      const props = this.props;
      // 如果子元素有这个事件类型并且trigger组件有这个事件类型
      // 就返回trigger组件中的对应的事件触发函数
      // 如果两者中有一方没有的话，就返回有的那一方的事件
      if (childPros[event] && props[event]) {
        return this[`fire${event}`];
      }
      return childPros[event] || props[event];
    }
  ```
+ ### 判断事件是否需要添加
  ```js
    // 这几个函数的结构都是一样的
    this.isContextMenuToShow();

    this.isClickToHide();

    this.isClickToShow();

    this.isMouseEnterToShow();

    this.isMouseLeaveToHide();

    this.isFocusToShow();

    this.isBlurToHide();

    // 这个函数是通过事件触发action来判断是否需要给组件绑定对应事件类型
    // 下面是伪代码
    isSomeEventToShowOrHide() {
      // 从传入props中的action和showAction中查询是否有这个事件类型
      // 有就返回true，否则返回false
      const { action, showActionOrHideAction } = this.props;
      return action.indexOf(event) !== -1 || showActionOrHideAction.indexOf(event) !== -1;
    }
  ```

## 生命周期

在createTwoChains函数中我们又看见了一个新的函数`'this[`fire${event}`]'`,

这些函数都是在componentDidMount的时候构建成的，那么记下来顺理成章的我们应该转接到组件的生命周期


```js
  // 首先设置一个popupVisible作为state中的一个变量，方便下面使用
  getInitialState() {
    const props = this.props;
    let popupVisible;
    if ('popupVisible' in props) {
      popupVisible = !!props.popupVisible;
    } else {
      popupVisible = !!props.defaultPopupVisible;
    }
    return {
      popupVisible,
    };
  },

  componentWillMount() {
    // 给每一个事件都写上默认事件
    ALL_HANDLERS.forEach((h) => {
      this[`fire${h}`] = (e) => {
        this.fireEvents(h, e);
      };
    });
  },

  componentDidMount() {
    // 在第一次渲染的时候强制性调用一下更新状态
    this.componentDidUpdate({}, {
      popupVisible: this.state.popupVisible,
    });
  },

  componentWillReceiveProps({ popupVisible }) {
    if (popupVisible !== undefined) {
      this.setState({
        popupVisible,
      });
    }
  },

  componentDidUpdate(_, prevState) {
    const props = this.props;
    const state = this.state;
    const triggerAfterPopupVisibleChange = () => {
      if (prevState.popupVisible !== state.popupVisible) {
        props.afterPopupVisibleChange(state.popupVisible);
      }
    };
    if (!IS_REACT_16) {
      // 如果不是react16版本就使用mixin中的函数渲染组件，并且能够执行外部afterPopupVisibleChange函数的回调
      this.renderComponent(null, triggerAfterPopupVisibleChange);
    } else {
      // 否则直接执行回调
      triggerAfterPopupVisibleChange();
    }

    // We must listen to `mousedown`, edge case:
    // https://github.com/ant-design/ant-design/issues/5804
    // https://github.com/react-component/calendar/issues/250
    // https://github.com/react-component/trigger/issues/50
    if (state.popupVisible) {
      let currentDocument;
      if (!this.clickOutsideHandler && (this.isClickToHide() || this.isContextMenuToShow())) {
        currentDocument = props.getDocument();
        this.clickOutsideHandler = addEventListener(currentDocument,
          'mousedown', this.onDocumentClick);
      }
      // always hide on mobile
      // `isMobile` fix: mask clicked will cause below element events triggered
      // https://github.com/ant-design/ant-design-mobile/issues/1909
      // https://github.com/ant-design/ant-design-mobile/issues/1928
      if (!this.touchOutsideHandler && isMobile) {
        currentDocument = currentDocument || props.getDocument();
        this.touchOutsideHandler = addEventListener(currentDocument,
          'click', this.onDocumentClick);
      }
      // close popup when trigger type contains 'onContextMenu' and document is scrolling.
      if (!this.contextMenuOutsideHandler1 && this.isContextMenuToShow()) {
        currentDocument = currentDocument || props.getDocument();
        this.contextMenuOutsideHandler1 = addEventListener(currentDocument,
          'scroll', this.onContextMenuClose);
      }
      // close popup when trigger type contains 'onContextMenu' and window is blur.
      if (!this.contextMenuOutsideHandler2 && this.isContextMenuToShow()) {
        this.contextMenuOutsideHandler2 = addEventListener(window,
          'blur', this.onContextMenuClose);
      }
      return;
    }
    // 清除所有外部的事件，因为上面为了解决一些issue而添加的事件
    this.clearOutsideHandler();
  },

  componentWillUnmount() {
    this.clearDelayTimer();
    this.clearOutsideHandler();
  },
```

可是看到现在也还是没有设么头绪，别忙这里先讲清楚一件事情，就是trigger这个组件的实现

trigger组件由于其中的展示内容需要绝对定位，但是这些定位如果放在已经存在的dom结构中会很复杂很难实现统一，于是这里就将所有的需要定位的元素全部渲染在body的最后，这样子计算定位就很方便了，所以trigger组件的目的就是需要将呈现的东西给渲染在body之后，但是大家都知道，react的render只要一个入口，也就是最初的id为root的div，然后就是在这个div里面进行react开发，所以react为大家提供了一个函数，我们在上面的renderComponent()这个函数中也讲到`unstable_renderSubtreeIntoContainer()`，可以使用这个函数就能够将组件中的内容渲染在创造出的节点上并且追加在任何地方，一般是追加在body，也可以追加在指定的dom节点后面

接下来就是另一个分析的思路，因为我在看这些代码的时候开始也是混乱的，在经过查资料的过程中我也在思考，发现到一个点那就是这个组件在判断当前react版本是不是react16，并且根据上面所讲的trigger组件的实现原理，我恍然大悟，因为在react16之前没有`createPortal`这个API的，这个API其实就是trigger的原理实现，所以我就知道了，判断如果是react16版本的就使用react自己的API来创建挂载点，如果不是就利用mixin中的renderComponent()函数中的老的react的方法`unstable_renderSubtreeIntoContainer()`来创建挂载点以及挂载组件，那么接下来我们就来分析一下他的思路。

## IS__REACT__16?

上面既然说到了要从当前版本来进行操作，那么我就按照是与不是分别看看这个组件都做了哪些处理

### IS

首先就是从当前react版本是16开始，从render函数开始，在render函数中我们就谈到有一个判断

```js
  if (!IS_REACT_16) {
    return trigger;
  }

  let portal;
  // prevent unmounting after it's rendered
  if (popupVisible || this._component) {
    portal = (
      <Portal
        key="portal"
        getContainer={this.getContainer}
      >
        {this.getComponent()}
      </Portal>
    );
  }

  return [
    trigger,
    portal,
  ];
```

也就是在使用cloneElement生成完trigger组件之后，如果不是react16版本就直接返回了trigger，然后如果是react16版本就使用Protal组件将需要挂载的dom元素渲染出来，使用getContainer进行dom节点的创建，使用getComponent将弹出层渲染，最终挂载在getContainer创建的dom节点，然后append在body，这就是使用了react16版本的一个创建过程，其中Protal组件中就是用了react16中的createPortal，剩下的就是Popup，又是antd的另一个基层组件，需要去了解。

### NOT IS

如果大家和我一样看了源码之后也许会纳闷，如果不是react16版本的时候，就直接返回了trigger组件，那么他是在什么时刻去渲染弹出层以及弹出层的挂载节点的呢？接下来就是揭秘时间：

```js
  componentDidUpdate(_, prevState) {
    const props = this.props;
    const state = this.state;
    const triggerAfterPopupVisibleChange = () => {
      if (prevState.popupVisible !== state.popupVisible) {
        props.afterPopupVisibleChange(state.popupVisible);
      }
    };
    if (!IS_REACT_16) {
      // 如果不是react16版本就使用mixin中的函数渲染组件，并且能够执行外部afterPopupVisibleChange函数的回调
      this.renderComponent(null, triggerAfterPopupVisibleChange);
    } else {
      // 否则直接执行回调
      triggerAfterPopupVisibleChange();
    }

    // 一些无关紧要的code ...
  }
```

在componentDidUpdate中在不是react16的时候使用了一个renderComponent函数，那么这个函数又是哪里来的呢，我们继续往上追溯，我们发现在上面讲到的getContainerRenderMixin中有这样的一断代码

```js
 if (!autoMount || !autoDestroy) {
    mixin = {
      // 如果不是自动渲染的，那就在mixin中添加一个渲染函数
      ...mixin,
      renderComponent(componentArg, ready) {
        renderComponent(this, componentArg, ready);
      },
    };
  }
```

那么知道mixin作用的同学就应该知道了，上面的componentDidUpdate中使用的renderComponent函数是在哪里定义的了，接下来就直接分析这个mixin中干了什么

首先是我们在使用的时候传入了这些参数；

```js
getContainerRenderMixin({
  autoMount: false,

  isVisible(instance) {
    return instance.state.popupVisible;
  },

  getContainer(instance) {
    return instance.getContainer();
  },
})
```

这里不得不再讲一遍这个getContainerRenderMixin

```js
  import ReactDOM from 'react-dom';

  function defaultGetContainer() {
    const container = document.createElement('div');
    document.body.appendChild(container);
    return container;
  }

  export default function getContainerRenderMixin(config) {
    // 首先传了三个参数进来，autoMount = false, isVisible(func), getContainer(func)
    const {
      autoMount = true,
      autoDestroy = true,
      isVisible,
      getComponent,
      getContainer = defaultGetContainer,
    } = config;

    let mixin;

    function renderComponent(instance, componentArg, ready) {
      // 当外部传入的状态为显示，并且外部的实例有_component（这个_component是在传入外部的Popup组件的ref所指向的节点）
      if (!isVisible || instance._component || isVisible(instance)) {
        if (!instance._container) {
          // trigger组件没有_container，默认创建一个
          instance._container = getContainer(instance);
        }
        let component;
        if (instance.getComponent) {
          // 如果传入实例有getComponent，则将传入的参数传入实例的getComponent函数
          component = instance.getComponent(componentArg);
        } else {
          // 否则就进行就是用传入参数中的getComponent方法构造一个Component
          component = getComponent(instance, componentArg);
        }
        // unstable_renderSubtreeIntoContainer是更新组件到传入的DOM节点上
        // 可以使用它完成在组件内部实现跨组件的DOM操作
        // ReactComponent unstable_renderSubtreeIntoContainer(
        //    parentComponent component,
        //    ReactElement element,
        //    DOMElement container,
        //    [function callback]
        //  )
        // 最终使用这个方法将弹出层挂载点以及弹出层进行渲染，然后还能够触发一个弹出层弹出之后的回调，感觉这个回调走得好绕。。。
        ReactDOM.unstable_renderSubtreeIntoContainer(instance,
          component, instance._container,
          function callback() {
            instance._component = this;
            if (ready) {
              ready.call(this);
            }
          });
      }
    }

    // trigger组件传入的autoMount为false所以这一段我们不需要再看
    if (autoMount) {
      mixin = {
        ...mixin,
        // 如果是自动渲染组件，那就在DidMount和DidUpdate渲染组件
        componentDidMount() {
          renderComponent(this);
        },
        componentDidUpdate() {
          renderComponent(this);
        },
      };
    }

    // 这里是入口，
    if (!autoMount || !autoDestroy) {
      mixin = {
        // 如果不是自动渲染的，那就在mixin中添加一个渲染函数
        ...mixin,
        renderComponent(componentArg, ready) {
          // 这里的this也就是当前mixin插入的类，componentArg是外部传入的null，raedy是外部传入的callback
          // 再次回到上面的renderComponent函数
          renderComponent(this, componentArg, ready);
        },
      };
    }

    function removeContainer(instance) {
      // 用于在挂载节点remove掉添加的组件
      if (instance._container) {
        const container = instance._container;
        // 先将组件unmount
        ReactDOM.unmountComponentAtNode(container);
        // 然后在删除挂载点
        container.parentNode.removeChild(container);
        instance._container = null;
      }
    }

    if (autoDestroy) {
      // 如果是自动销毁的，那就在WillUnmount的时候销毁
      mixin = {
        ...mixin,
        componentWillUnmount() {
          removeContainer(this);
        },
      };
    } else {
      mixin = {
        // 如果不是自动销毁，那就只是在mixin中添加一个销毁的函数
        ...mixin,
        removeContainer() {
          removeContainer(this);
        },
      };
    }
    // 最后返回构建好的mixin
    return mixin;
  }
```

这样trigger组件的一个大致构造思路以及大部分代码就已经进行了解读，剩余的部分都是进行的状态控制，antd为了适应手机还所以在状态控制上面写了很多函数，不多都是简单的函数，而且有的函数仅仅只是为了出一些出现的issue，感觉有点hotfix的意味，反正希望看完这一节对于大家制作react弹出层有一定的了解，这里我就提出两点

1. react16版本之前的需要自己写一个弹出层挂载点
2. react16版本之后的可以使用react提供的createPortal进行挂载点的处理

当然这个弹出层不仅仅是小的弹出层，可以制作很多东西，模态框，提醒框，下拉菜单，tooltip等等只要是需要绝对定位在某一个元素的某一个位置的场景，尽量发挥想象吧。
