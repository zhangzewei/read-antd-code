# Form 表单

这个组件貌似比较独特，在官网上面的每一个例子都是使用了`Form.create()`这个HOC的方法去进行的创建相应的Form组件
所以对于表单组件我们主要讲的应该就会是create这个高阶函数

## Form.create

这是一个高阶函数，传入的是react组件，返回一个新的react组件，在函数内部会对传入组件进行改造，添加上一定的方法用于进行一些秘密操作
如果有对高阶组件有想要深入的请移步[这里](http://www.jianshu.com/p/0aae7d4d9bc1)，我们这里不做过多的深究。接下来我们直接看这个函数的代码

```js
  static create = function<TOwnProps>(options: FormCreateOption<TOwnProps> = {}): ComponentDecorator<TOwnProps> {
    const formWrapper = createDOMForm({
      fieldNameProp: 'id',
      ...options,
      fieldMetaProp: FIELD_META_PROP,
    });

    /* eslint-disable react/prefer-es6-class */
    return (Component) => formWrapper(createReactClass({
      propTypes: {
        form: PropTypes.object.isRequired,
      },
      childContextTypes: {
        form: PropTypes.object.isRequired,
      },
      getChildContext() {
        return {
          form: this.props.form,
        };
      },
      componentWillMount() {
        this.__getFieldProps = this.props.form.getFieldProps;
      },
      deprecatedGetFieldProps(name, option) {
        warning(
          false,
          '`getFieldProps` is not recommended, please use `getFieldDecorator` instead, ' +
          'see: https://u.ant.design/get-field-decorator',
        );
        return this.__getFieldProps(name, option);
      },
      render() {
        this.props.form.getFieldProps = this.deprecatedGetFieldProps;

        const withRef: any = {};
        if (options.withRef) {
          withRef.ref = 'formWrappedComponent';
        } else if (this.props.wrappedComponentRef) {
          withRef.ref = this.props.wrappedComponentRef;
        }
        return <Component {...this.props} {...withRef} />;
      },
    }));
  };
```

从代码看出这个函数返回的是一个函数，接受一个组件作为参数，但是返回什么不是很清楚，所以需要再看看`createDOMForm`创建的是一个什么

`createDOMForm`是`rc-form`库中引用的，从代码一层层的查找下去发现创建一个form组件的主要代码是在`createBaseForm.js`这个文件中

```js
function createBaseForm(option = {}, mixins = []) {
  const { ... } = option;

  return function decorate(WrappedComponent) {
    const Form = createReactClass({ ... });

    return argumentContainer(Form, WrappedComponent);
  };
}
```

这又是一个高阶函数，在这个函数中先创建了一个Form组件，然后使用`argumentContainer`函数进行包装在传出，传出的是一个新的组件

这个新的组件将会拥有传入组件以及高阶组件中的所有属性

```js
import hoistStatics from 'hoist-non-react-statics';

export function argumentContainer(Container, WrappedComponent) {
  /* eslint no-param-reassign:0 */
  Container.displayName = `Form(${getDisplayName(WrappedComponent)})`;
  Container.WrappedComponent = WrappedComponent;
  return hoistStatics(Container, WrappedComponent);
}
```
`argumentContainer`函数使用了一个库`hoist-non-react-statics`，这个库是用于解决高阶组件不能够使用传入的组件的静态方法这个问题的

具体在react官网上面也有相应的[解释](https://reactjs.org/docs/higher-order-components.html#static-methods-must-be-copied-over)，使用了这个方法就能够将传入组件的静态方法也完全拷贝到高阶函数返回的组件中。

从现在看来之前代码中的`formWrapper`就是一个接受传入组件，然后再将组件进行转化成为一个添加了antd自己的Form高阶组件。

## 总结

通过这个组件的这个函数，加深了我对HOC的使用和认识，也对装饰器有了更深认识，技能点+1。