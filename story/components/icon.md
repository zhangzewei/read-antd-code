# Icon 

`icon`作为开发当中使用相对频繁的一个组件，其实现也很简单，但是其中比较麻烦的一部分是icon字体的制作，可以参看这篇[文章](https://segmentfault.com/a/1190000008374352)。

Antd的Icon组件使用了很简单的css来实现交互与动效

```js
import React from 'react';
import classNames from 'classnames';  
import omit from 'omit.js';
//classNames是条件判断输出className的值
//omit是移出对象的指定属性，而实现浅拷贝

//定义IconProps接口
export interface IconProps {
  type: string;
  className?: string;
  title?: string;
  onClick?: React.MouseEventHandler<any>;
  spin?: boolean;
  style?: React.CSSProperties;
}

const Icon = (props: IconProps) => {
  const { type, className = '', spin } = props;
  const classString = classNames({
    anticon: true,
    'anticon-spin': !!spin || type === 'loading', // 是否需要旋转动画，loading这个icon是默认加上旋转动效的
    [`anticon-${type}`]: true,
  }, className);

  // 这里说一下为什么要用omit()：html的<i>标签，其标准标签属性只有六种：id、class、title、style、dir、lang。
  // IconProps接口中的6种属性（方法），type、spin不属于上述六种。onClick为事件属性，可以；
  return <i {...omit(props, ['type', 'spin'])} className={classString} />;
};

export default Icon;
```

大家也可以根据上面所提供的制作icon的方法和这样的方式来实现自己的Icon组件
