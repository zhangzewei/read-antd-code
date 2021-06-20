# sticky 魔法

众所周知，css 有一个样式叫做 `sticky` 是一个能够将元素紧贴在浏览器窗口顶部的样式，但是在其[兼容性](https://caniuse.com/?search=sticky)目前并不是很理想，所以我们有时候需要自己手动去实现 sticky 的效果，当然已经存在了很多相关的组件或者包提供大家使用，但是对于了解其原理来说，自己动手写一遍，才能够深刻的记牢。

## 初版需求
首先需要理清楚需求是什么：

1. 在页面滚动过程中，元素划过浏览器视窗顶部之后，需要将其固定在浏览器视窗下方
2. 固定在浏览器视窗下方需要能够设定一定的偏移量

## 初版实现
从上面的需求看来知道，一个元素当它划过了浏览器顶部时就需要将其固定在浏览器视窗顶部，所以实现方案也很简单，即当它的顶部与浏览器顶部之间的距离小于0之后，就给其添加 `position: fixed;` 的样式, 对于偏移量来说就是 `top: offset` 即可；

```js
const stickyElement = document.querySelector('.sticky');
window.addEventListener('scroll', function() {
  const isScrollOutScreen = stickyElement.getBoundingClientRect().top < 0;
  if (isScrollOutScreen) {
    stickyElement.style.position = 'fixed';
    stickyElement.style.top = `${offset || 0}px`;
  } else {
    stickyElement.style.position = 'relative';
    stickyElement.style.top = null;
  }
});
```
### 解析
#### element.getBoundingClientRect [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)

使用 `getBoundingClientRect` 返回的是一个 [DOMRect](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMRect) 对象可以获得元素的 `{ top, left, bottom, right, x, y, width, height }` 值，除了 width 和 height 属性，其余的值都是根据浏览器视窗来进行计算的，所以这里的 top 值就是所需要的相对于浏览器顶部的距离。

#### 方案缺点
因为使用的是 `fixed` 的方式，就会让 `dom` 节点脱离文档流，如果 sticky 元素在之前是占有一定高度的情况，在脱离文档流和恢复的瞬间会造成某些不可预料的 `html` 结构塌陷或者抖动，要防止这样的情况发生需要使用一个占位元素去将 sticky 的元素原有位置占位。

## 进阶实现
其实在很多时候可以使用 css3 的 `transform` 去简化实现一些复杂的布局方案。
```js
const stickyElement = document.querySelector('.sticky');
window.addEventListener('scroll', function() {
  const scrollTop = window.scrollY; // 这里如果不是window是其他元素的应该使用 scrollTop；
  const isScrollOutScreen = stickyElement.getBoundingClientRect().top < 0;
  if (isScrollOutScreen) {
    stickyElement.style.transform = `translateY(${scrollTop - stickyElement.offsetTop}px)`;
  } else {
    stickyElement.style.transform = 'translateY(0px)';
  }
});
```
## 进阶需求

1. 在页面滚动过程中，元素需要在某个区域之间固定在浏览器顶部，超过或者未达到该区域，元素都不出现在是窗内。
2. 固定在浏览器视窗下方需要能够设定一定的偏移量

## 进阶需求解析
这个需求是需要在某块区域内部才进行 sticky 的操作，那么可以理解为，在某个大的元素块一直在浏览器视窗内的时候，对需要固定的元素进行 sticky 操作，其实这样子的需求现在看来是相对较多的，例如：比如在某个大区域块内的导航，只是跟随内容部分进行固定，内容部分以外就不会进行固定，下方内容继续正常展示。

## 进阶需求实现

```js
const isInWindowScreen = (ele, offset = 0) => {
  // if element's top is bigger than element's negative height and element's top is less than window's height
  // so that we can call this situation is element in the window screen now.
  if (!ele) return false;
  const windowHeight = window.innerHeight;
  const { top, height } = ele.getBoundingClientRect();
  return top <= offset && top > - windowHeight - height + offset;
};

const tinySticky = (stickyElement, stickyAreaElement, offset = 0) => {
  if (!stickyElement) {
    return console.error('please set a element to sticky');
  }
  const stickyElementHeight = stickyElement.getBoundingClientRect().height;
  window.addEventListener('scroll', () => window.requestAnimationFrame(() => {
    const { top, height } = stickyAreaElement.getBoundingClientRect();
    const isInWindow = isInWindowScreen(stickyAreaElement);
    const totalHeight = height - stickyElementHeight;
    if (isInWindow) {
      stickyElement.style.transform = `translateY(${(-top >= totalHeight) ? totalHeight + offset : offset - top}px)`;
    } else {
      stickyElement.style.transform = 'translateY(0px)';
    }
  }));
}
```

## 解析

这里判断元素是否在浏览器视窗内部的方法就不再是那么简单
1. 首先需要判定，元素距离浏览器视窗的顶部是不是小于 0，这时候就意味着已经进入了固定区域的判定了；
2. 然后是判定什么时候离开判定区域，即元素距离浏览器视窗顶部的距离的负值（因为超过了浏览器视窗顶部之后就是负数了，所以需要加个符号转正）是否小于浏览器视窗高度和元素本身高度，在这个区域内部，就代表元素还出现在浏览器视窗内。
3. 固定元素的偏移量计算也变化了，不再使用的是 `scrollY | scrollTop`，直接使用滚动区域的 `top` 的负值，这样子就不需要判断滚动区域是否是 `window` 了，并且增加了新的算法，需要判断判定区域的 `top` 是否超出了总高度（总高度的意思是判定区域的高度减去固定元素的高度），这样子才可以将固定元素固定在判定区域的底部，当页面滚动到视窗内展示的判定区域的高度正好等于固定元素高度的时候。
4. 最后使用 `window.requestAnimationFrame` 做一下性能优化，取代 `debounce`。

## 兼容性

1. [getBoundingClientRect](https://www.caniuse.com/?search=getBoundingClientRect) 因为要用到 `height` 这个属性，所以兼容到 IE9;
2. [transform.translateY](https://www.caniuse.com/?search=translateY) 兼容到 IE9;
3. [requestAnimationFrame](https://www.caniuse.com/?search=requestAnimationFrame) 兼容到 IE10;

综上：如果要兼容到 IE9，可以不使用 `requestAnimationFrame`，剩下的可以兼容到 IE10，算是不错的兼容性了。

## 总结

实现 sticky 的时候，最开始的需求其实就很简单，但是逐步逐步的，进行需求变化或者进行代码重构优化，这是一个必经之路，有时候并不是一开始写的代码就足够好或者能够完全 cover 住所有的场景，需要一步一步的思考还需要做什么以及怎么去做，同理做其他的需求也是如此。

## 广告

这里是本篇文章的 [github](https://github.com/zhangzewei/mini-sticky) 地址

我也发布到了 [npm](https://www.npmjs.com/package/tiny-sticky) 上，欢迎大家使用，包体积很小，只有8.19 kB