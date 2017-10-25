# DropDown 下拉菜单

下拉菜单组件是一个可以将页面上比较冗杂的操作收纳在一个点，以便节省页面空间，达到整洁美观的目的。

Antd的下拉菜单组件中有一个点，就是他的内部元素必须是Antd的Menu组件，感觉有点捆绑的意思。

## DropDownProps

这里留一个小问题，为什么触发方式是一个数组，而不是单个的

```js
  export interface DropDownProps {
    trigger?: ('click' | 'hover')[]; // 触发方式
    overlay: React.ReactNode; // 下拉菜单所承载的内容元素，要求为Antd的Menu组件
    style?: React.CSSProperties; // 行内样式
    onVisibleChange?: (visible?: boolean) => void; // 监听下拉菜单出现/消失
    visible?: boolean; // 菜单是否显示
    disabled?: boolean; // 菜单是否可以用
    align?: Object; // 这个参数目前没有被使用
    getPopupContainer?: (triggerNode: Element) => HTMLElement; // 渲染的挂载点，默认为body
    prefixCls?: string; // 样式类的命名前缀
    className?: string; // 样式
    placement?: 'topLeft' | 'topCenter' | 'topRight' | 'bottomLeft' | 'bottomCenter' | 'bottomRight';
    // 弹出框与触发点的对齐方式
  }
```

## 直接看代码

因为这个组件主要使用的是[rc-dropdown组件](https://github.com/react-component/dropdown)
所以这里只是对其参数做了一些封装，比较简单。

```js
  export default class Dropdown extends React.Component<DropDownProps, any> {
    static Button: typeof DropdownButton;
    static defaultProps = {
      prefixCls: 'ant-dropdown',
      mouseEnterDelay: 0.15,
      mouseLeaveDelay: 0.1,
      placement: 'bottomLeft',
    };

    // 设定一个动画效果名称
    getTransitionName() {
      const { placement = '' } = this.props;
      // js的indexOf()可以使用在Array上也可以使用在String上
      // 使用方法一样，第一个参数是匹配的对象，第二个参数是从哪里开始匹配
      if (placement.indexOf('top') >= 0) {
        return 'slide-down';
      }
      return 'slide-up';
    }

    componentDidMount() {
      // 这里就在检测菜单内容是否是antd的menu组件，并且检测menu组件的样式
      const { overlay } = this.props;
      const overlayProps = (overlay as any).props as any;
      // warning函数还是和之前学习的一样的用法
      warning(
        !overlayProps.mode || overlayProps.mode === 'vertical',
        `mode="${overlayProps.mode}" is not supported for Dropdown\'s Menu.`,
      );
    }

    render() {
      const { children, prefixCls, overlay, trigger, disabled } = this.props;
      // 将dropdown包裹的触发器加以封装，再渲染
      const dropdownTrigger = cloneElement(children as any, {
        className: classNames((children as any).props.className, `${prefixCls}-trigger`),
        disabled,
      });
      // menu cannot be selectable in dropdown defaultly
      const overlayProps = overlay && (overlay as any).props;
      const selectable = (overlayProps && 'selectable' in overlayProps)
        ? overlayProps.selectable : false;
      // 同样的将dropdown包裹的内容加以封装，再渲染
      const fixedModeOverlay = cloneElement(overlay as any, {
        mode: 'vertical',
        selectable,
      });
      return (
        // 最后将所有参数传入rc-dropdown组件
        <RcDropdown
          {...this.props}
          transitionName={this.getTransitionName()}
          trigger={disabled ? [] : trigger}
          overlay={fixedModeOverlay}
        >
          {dropdownTrigger}
        </RcDropdown>
      );
    }
  }
```

# DropdownButton

这个组件是一个带按钮的下拉菜单组件，其实其原理就是使用的之前所讲的ButtonGroup来进行组合的一个组件。

想要了解ButtonGroup组件的可以点击[这里](./button_group.md)

## DropdownButtonProps

这个组件的props继承了两个其他组件的props，这是typescript的interface的一个特性，可以继承多个，来形成一个新的。

```js
  export interface DropdownButtonProps extends ButtonGroupProps, DropDownProps {
    type?: 'primary' | 'ghost' | 'dashed'; // 按钮类型
    disabled?: boolean; // 是否禁用
    onClick?: React.MouseEventHandler<any>; // 点击事件
    children?: any; // 子节点
  }
```

## Render()

从render函数就可以看出这个组件就是antd为了方便，为大家封装好了的一个带下拉菜单的按钮组件

```js
  render() {
    const {
      type, disabled, onClick, children,
      prefixCls, className, overlay, trigger, align,
      visible, onVisibleChange, placement, getPopupContainer,
      ...restProps,
    } = this.props;

    const dropdownProps = {
      align,
      overlay,
      trigger: disabled ? [] : trigger,
      onVisibleChange,
      placement,
      getPopupContainer,
    };
    if ('visible' in this.props) {
      (dropdownProps as any).visible = visible;
    }

    return (
      <ButtonGroup
        {...restProps}
        className={classNames(prefixCls, className)}
      >
        <Button
          type={type}
          disabled={disabled}
          onClick={onClick}
        >
          {children}
        </Button>
        <Dropdown {...dropdownProps}>
          <Button type={type} disabled={disabled}>
            <Icon type="down" />
          </Button>
        </Dropdown>
      </ButtonGroup>
    );
  }
```

## 这就完了？怎么可能，还不过瘾

写到这里就完了么？是滴，这一节就完了，因为查看了一下rc-dropdown的实现，然后根据平常看代码的习惯

从render()函数入口发现这一段代码

```js
  render() {
    const {
      prefixCls, children,
      transitionName, animation,
      align, placement, getPopupContainer,
      showAction, hideAction,
      overlayClassName, overlayStyle,
      trigger, ...otherProps,
    } = this.props;
    return (
      <Trigger
        {...otherProps}
        prefixCls={prefixCls}
        ref={this.saveTrigger}
        popupClassName={overlayClassName}
        popupStyle={overlayStyle}
        builtinPlacements={placements}
        action={trigger}
        showAction={showAction}
        hideAction={hideAction}
        popupPlacement={placement}
        popupAlign={align}
        popupTransitionName={transitionName}
        popupAnimation={animation}
        popupVisible={this.state.visible}
        afterPopupVisibleChange={this.afterVisibleChange}
        popup={this.getMenuElement()}
        onPopupVisibleChange={this.onVisibleChange}
        getPopupContainer={getPopupContainer}
      >
        {children}
      </Trigger>
    );
  }
```

然后再看到`Trigger`这个组件，居然是另外一个库[rc-trigger](https://github.com/react-component/trigger)
然后在看了`rc-trigger`的实现之后，才知道原来tigger组件才是下拉菜单的核心。

所以还没完呢，只是需要讲解的太多，不适合在一篇文章中讲全面，所以敬请期待下一篇文章，

我们将会学习rc-trigger组件的实现，之后会再写一篇rc-dropdown的组件实现解读，

最后看完这三篇文章，再倒过来重温一遍，你将会学到怎么样一层层的包装组件，将一个基础组件包装成为一个高级组件的过程。