# Trigger > Popup

在之前的trigger的index文件中，其渲染的弹出层就是这个Popup组件，这篇文章将会为你揭晓Popup。

## 提纲

这个组件中我们主要路线是这样的

Popup组件使用了两个另外的rc-component，分别是`rc-align`，`rc-animate`，这两个组件我们目前只需要知道他们能够做什么就好了。

然后是这些重点

```js
  // 方法
  saveRef();
  React.findDOMNode();
  getPopupElement();
  getMaskElement();

  // 额外组件
  PopupInner
  LazyRenderBox
```

下面的文章将围绕这些重点展开，解决了这些疑点之后，就可以对Popup组件有所了解

## 外部插件了解

这里又有两个外部的插件，也是antd写的基层组件，一个是rc-align组件，这个是用于定位的组件，另外一个rc-animate组件，是用于动画特效的组件，我们之后一定会讲到这两个组件的，因为这是基层组件，是整个antd都会使用到的。

### rc-align

对于组件，我们一般要用的话只需要了解他的api就好

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
          <td>align</td>
          <td>Object</td>
          <td></td>
          <td>same with alignConfig from https://github.com/yiminghe/dom-align</td>
        </tr>
        <tr>
          <td>onAlign</td>
          <td>function(source:HTMLElement, align:Object)</td>
          <td></td>
          <td>called when align</td>
        </tr>
        <tr>
          <td>target</td>
          <td>function():HTMLElement</td>
          <td>function(){return window;}</td>
          <td>a function which returned value is used for target from https://github.com/yiminghe/dom-align</td>
        </tr>
        <tr>
          <td>monitorWindowResize</td>
          <td>Boolean</td>
          <td>false</td>
          <td>whether realign when window is resized</td>
        </tr>
    </tbody>
</table>

打开了他所用的定位的[alignConfig](https://github.com/yiminghe/dom-align)，不由得在想是不是他们自己写的一套定位的方案。。。

### rc-animate

这是一个动画组件，嵌套的组件可以被添加上动画特效

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
          <td>component</td>
          <td>React.Element/String</td>
          <td>'span'</td>
          <td>wrap dom node or component for children. set to '' if you do not wrap for only one child</td>
        </tr>
        <tr>
          <td>showProp</td>
          <td>String</td>
          <td></td>
          <td>using prop for show and hide. [demo](http://react-component.github.io/animate/examples/hide-todo.html) </td>
        </tr>
        <tr>
          <td>exclusive</td>
          <td>Boolean</td>
          <td></td>
          <td>whether allow only one set of animations(enter and leave) at the same time. </td>
        </tr>
        <tr>
          <td>transitionName</td>
          <td>String</td>
          <td></td>
          <td>transitionName, need to specify corresponding css</td>
        </tr>
       <tr>
         <td>transitionAppear</td>
         <td>Boolean</td>
         <td>false</td>
         <td>whether support transition appear anim</td>
       </tr>
        <tr>
          <td>transitionEnter</td>
          <td>Boolean</td>
          <td>true</td>
          <td>whether support transition enter anim</td>
        </tr>
       <tr>
         <td>transitionLeave</td>
         <td>Boolean</td>
         <td>true</td>
         <td>whether support transition leave anim</td>
       </tr>
       <tr>
         <td>onEnd</td>
         <td>function(key:String, exists:Boolean)</td>
         <td>true</td>
         <td>animation end callback</td>
       </tr>
        <tr>
          <td>animation</td>
          <td>Object</td>
          <td>{}</td>
          <td>
            to animate with js. see animation format below.
          </td>
        </tr>
    </tbody>
</table>

这里有一个参数`showProp`我开始没明白是啥意思，后面才看了他的实例才知道是使用哪一个prop作为显示还是隐藏的开关判断