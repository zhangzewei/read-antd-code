# ButtonGroup

这个组件没有重点可以说，毕竟就只是一个将`Button`组件包裹起来的一个容器，但是这里还是有一个点可以值得一提
```js
  // 这里的React.SFC是 typescript 的对于 react 的StatelessComponent的一个interface的一个别称
  // 那么对于Stateless Functional Component，是一种不需要管理state的组件，也就是说这个组件中不会
  // 对state进行操作的组件，是一个纯函数组件，大家有兴趣可以去了解
  // 详情请看 https://medium.com/@iktakahiro/react-stateless-functional-component-with-typescript-ce5043466011
  const ButtonGroup: React.SFC<ButtonGroupProps> = (props) => {}
```

```tsx
  import React from 'react';
  import classNames from 'classnames';

  export type ButtonSize = 'small' | 'large';

  export interface ButtonGroupProps {
    size?: ButtonSize;
    style?: React.CSSProperties;
    className?: string;
    prefixCls?: string;
  }

  const ButtonGroup: React.SFC<ButtonGroupProps> = (props) => {
    const { prefixCls = 'ant-btn-group', size = '', className, ...others } = props;

    // large => lg
    // small => sm
    let sizeCls = '';
    switch (size) {
      case 'large':
        sizeCls = 'lg';
        break;
      case 'small':
        sizeCls = 'sm';
      default:
        break;
    }

    const classes = classNames(prefixCls, {
      [`${prefixCls}-${sizeCls}`]: sizeCls,
    }, className);

    return <div {...others} className={classes} />;
  };

  export default ButtonGroup;
```