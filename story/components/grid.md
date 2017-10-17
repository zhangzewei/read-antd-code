# Grid 
这个组件完全使用的是`flex`布局，如果还对`flex`布局不熟悉的同学可以看[这里](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)。

该组件有两个部分，一个是`Row`，一个是`Col`，采用`Row`包裹`Col`的方法来实现栅格布局，并且栅格布局是遵从[Bootstrap 3](https://getbootstrap.com/docs/3.3/css/#responsive-utilities-classes)的标准。

## Row

Row组件有一个比较特别的参数，就是`gutter`，这个参数是指的每个元素之间的间距，这个东西是在flex布局中没有存在的一个设定，所以对于比较熟悉的同学可以直接看他的间距的实现。

这里面还有两个需要注意的点，分别是:[React.Children](http://lib.csdn.net/article/react/12197)，[React.cloneElement()](https://segmentfault.com/a/1190000010062928)，这两个玩意儿在这react的对于子元素操作中非常常用，熟悉的同学可以跳过。
```js
import React from 'react';
import { Children, cloneElement } from 'react';
import classNames from 'classnames';
import PropTypes from 'prop-types';

export interface RowProps {
  className?: string;
  gutter?: number;
  type?: 'flex';
  align?: 'top' | 'middle' | 'bottom';
  justify?: 'start' | 'end' | 'center' | 'space-around' | 'space-between';
  style?: React.CSSProperties;
  prefixCls?: string;
}

export default class Row extends React.Component<RowProps, {}> {
  static defaultProps = {
    gutter: 0,
  };

  static propTypes = {
    type: PropTypes.string,
    align: PropTypes.string,
    justify: PropTypes.string,
    className: PropTypes.string,
    children: PropTypes.node,
    gutter: PropTypes.number,
    prefixCls: PropTypes.string,
  };
  render() {
    const { type, justify, align, className, gutter, style, children,
      prefixCls = 'ant-row', ...others } = this.props;
    const classes = classNames({
      [prefixCls]: !type,
      [`${prefixCls}-${type}`]: type,
      [`${prefixCls}-${type}-${justify}`]: type && justify,
      [`${prefixCls}-${type}-${align}`]: type && align,
    }, className);
    // 如果有gutter这个参数，就是需要添加间距，他的实现方法是给每一个Item添加左右的pading，
    // 但是又不想让第一个和最后一个Item也有这个内边距，所以在父级元素上面设置左右相同负值
    // 的margin，就能够抵消两端的padding。
    const rowStyle = (gutter as number) > 0 ? {
      marginLeft: (gutter as number) / -2,
      marginRight: (gutter as number) / -2,
      ...style,
    } : style;
    // 这里就用到了上面所说的Children 和 cloneElement
    // React.Children使用的时候不必担心内部子元素是否有嵌套关系
    const cols = Children.map(children, (col: React.ReactElement<HTMLDivElement>) => {
      if (!col) {
        return null;
      }
      // 判断一下col是否有props
      if (col.props && (gutter as number) > 0) {
        // 返回一个新的组件，会新增添加的其余的props或者修改过后的props
        return cloneElement(col, {
          style: {
            paddingLeft: (gutter as number) / 2,
            paddingRight: (gutter as number) / 2,
            ...col.props.style,
          },
        });
      }
      return col;
    });
    return <div {...others} className={classes} style={rowStyle}>{cols}</div>;
  }
}
```

## Col

`Col`这个组件全部都是用的CSS以及flex实现的，唯一需要讲解的大概应该是`'xs', 'sm', 'md', 'lg', 'xl'`的使用，为什么会有传入对象的参数，因为这样子可以自定义自己的一个栅格布局，实现更加灵活的栅格布局，这样也使得`Col`这个组件更加灵活
```js
import React from 'react';
import PropTypes from 'prop-types';
import classNames from 'classnames';

// 这里使用了PropTypys.oneOfType()，这个函数的意思就是使用其数组值周的任意的一种累心都可以
const stringOrNumber = PropTypes.oneOfType([PropTypes.string, PropTypes.number]);
const objectOrNumber = PropTypes.oneOfType([PropTypes.object, PropTypes.number]);

export interface ColSize {
  span?: number;
  order?: number;
  offset?: number;
  push?: number;
  pull?: number;
}

export interface ColProps {
  className?: string;
  span?: number;
  order?: number;
  offset?: number;
  push?: number;
  pull?: number;
  xs?: number | ColSize;
  sm?: number | ColSize;
  md?: number | ColSize;
  lg?: number | ColSize;
  xl?: number | ColSize;
  prefixCls?: string;
  style?: React.CSSProperties;
}

export default class Col extends React.Component<ColProps, any> {
  static propTypes = {
    span: stringOrNumber,
    order: stringOrNumber,
    offset: stringOrNumber,
    push: stringOrNumber,
    pull: stringOrNumber,
    className: PropTypes.string,
    children: PropTypes.node,
    xs: objectOrNumber,
    sm: objectOrNumber,
    md: objectOrNumber,
    lg: objectOrNumber,
    xl: objectOrNumber,
  };

  render() {
    const props = this.props;
    const { span, order, offset, push, pull, className, children, prefixCls = 'ant-col', ...others } = props;
    let sizeClassObj = {};
    ['xs', 'sm', 'md', 'lg', 'xl'].forEach(size => {
      let sizeProps: ColSize = {};
      // 当传入参数为对象的时候，就不仅可以定义span，还可以定义其他的参数,push, pull, older, offset 
      if (typeof props[size] === 'number') {
        sizeProps.span = props[size];
      } else if (typeof props[size] === 'object') {
        sizeProps = props[size] || {};
      }

      delete others[size];

      sizeClassObj = {
        ...sizeClassObj,
        [`${prefixCls}-${size}-${sizeProps.span}`]: sizeProps.span !== undefined,
        [`${prefixCls}-${size}-order-${sizeProps.order}`]: sizeProps.order || sizeProps.order === 0,
        [`${prefixCls}-${size}-offset-${sizeProps.offset}`]: sizeProps.offset || sizeProps.offset === 0,
        [`${prefixCls}-${size}-push-${sizeProps.push}`]: sizeProps.push || sizeProps.push === 0,
        [`${prefixCls}-${size}-pull-${sizeProps.pull}`]: sizeProps.pull || sizeProps.pull === 0,
      };
    });
    // 利用classnames这个库可以高效的合并并且覆盖元素样式
    const classes = classNames({
      [`${prefixCls}-${span}`]: span !== undefined,
      [`${prefixCls}-order-${order}`]: order,
      [`${prefixCls}-offset-${offset}`]: offset,
      [`${prefixCls}-push-${push}`]: push,
      [`${prefixCls}-pull-${pull}`]: pull,
    }, className, sizeClassObj);

    return <div {...others} className={classes}>{children}</div>;
  }
}

```