# 微信小程序开发踩坑
经历了一次小程序开发，遇到了一些开发时遇到的坑，首先说明一下技术框架是使用的京东的 taro 进行的开发，并且使用的 hooks 的开发方式

## Taro 的坑
首先来看看使用 taro 框架会有哪些坑是需要避免的呢

1. ### taro hooks缺少
    我们在使用的时候，发现 taro 的生命周期当中有一个生命周期没有对应的 hook，这个生命周期就是 componentWillUnmount，并且源码中也没有能够去增加新关于生命周期的 hook 的功能，所以如果要用到这个生命周期的话，请使用 class 的写法；

2. ### 自定义组件在 taro 的组件上不能直接使用外部传入的 classname
    需要使用到 custom-class 或者 css module 解决这个问题

3. ### 自定义组件的 props 不能传入复杂的渲染函数
    ```tsx
    // 比如一个带有头像的电话列表
    
    const tels = [
      {
        FIRST_CHAR: 'A'
        list: [{
          avatar: 'imgsrc',
          tel: '12345678900'
        },{
          avatar: 'imgsrc',
          tel: '12345678900'
        },{
          avatar: 'imgsrc',
          tel: '12345678900'
        },{
          avatar: 'imgsrc',
          tel: '12345678900'
        }]
      }
    ];

    // 这样的一个数据结构，因为想做一个通用性更大的组件，所以在设计的时候想要把第二层渲染list的渲染函数写成一个从外部传入的函数

    const render = () => {
      return (<Indexes
        renderItem={
          item => (<View><Image src={item.avatar} /><Text>{item.tel}</Text></View>)
        }
      />);
    }

    // 但是并不能这样子写，taro不会进行渲染，所以只能够将列表的渲染方法写在组件内部
    list.map((listItem) => (
      <View
        key={`index-${listItem.key}`}
        className="index-list-item"
        id={`index-${listItem.key}`}
      >
        <View className="item-title">{listItem.title}</View>
        // {renderItem()} 如果是使用上面那种方式 直接在这里写成这样就可以了
        {listItem.items.map((i) => {
          return (
            <View
              key={i.name}
              className="item-container"
              hoverClass="item-container-hover"
              onClick={() => onClick(i)}
            >
              {needIcon && ( // 因为不确定 icon 的确切的值，所以就交给 CustomImage 组件内部处理，外面只需要管需不需要icon的属性
                <CustomImage
                  src={i.icon}
                  mode="aspectFit"
                  custom-class="icon"
                  defaultImage={needIcon && defaultIcon}
                />
              )}
              <Text>{i.name}</Text>
            </View>
          );
        })}
      </View>

      // 所以复杂的渲染还是需要写在组件内部

      // 另外如果需要传入 tsx 作为 props 的话，需要使用一个 View包裹起来

      <Component tsx={<View>something</View>} />
    ```

## 在开发中需要保持的一些思维

1. ### 在使用 useEffect 的时候需要有函数的思维
    使用一个 useEffect 时会在后面加上其依赖，当依赖变化，就会触发一次 useEffect 的回调，所以在使用 setState 改变依赖的时候，需要有一个函数式的思维，改变变量进而产生不一样的结果
    
    ```js
      // 比如翻页请求下一页时，需要的是改变页码就能够触发函数，而不是不断的调用函数
      const [page, setPage] = useState(1);

      useEffect(() => {
        // ... 获取相应页码的数据
      }, [page]);

      const changePage = () => setPage(prePage => prePage + 1)
    ```

2. ### 当从一个页面进入到另一个页面做了某些操作之后，回到当前页面需要更新当前页的一些数据
    这是一个很常见的场景，比如我在个人信息页面，我进入修改页面修改了某些数据，当我返回时期望的是看到我已经修改的数据展示在页面上，这种场景在实用 hook 并且没有实用 redux 技术下，只能够实用 useDidShow 这个 hook 去进行页面数据的刷新。
    ```js
      useDidShow(() => {
        // ... 再次获取数据
      });
    ```

3. ### 多次不间断触发查询条件，应该以最后一次结果为准展示
    这个场景是在一个能够使用筛选的列表中遇到的，当我不断切换筛选条件并且在网络环境较差的时候，页面中的渲染会有多次渲染，渲染次数和切换的次数一样，原因是因为没有取最后一次结果作为渲染的依据。所以在有类似情况出现时，应该使用唯一标识去拿到最后一次请求的结果再进行页面的渲染，具体实施方法有很多，我使用的是使用一个数字代表请求次数，每次请求次数加一，请求完成次数减一，直到次数归零，才进行页面渲染。
    ```js
      useEffect(() => {
        setLoading(true);
        queryQueen++;
        searchStationByKeyword().then((res: StationCollection) => {
          queryQueen--;
          if (res.chcList && queryQueen === 0) {
            setLoading(false);
          }
        });
      }, []);
    ```

4. ### 对于 Image 组件需要有错误处理
    很多同学在使用图片的时候都没有对于图片进行错误处理，导致只要遇到图片链接不正确或者没有链接的时候，页面中的展示就会出现问题，所以不论在什么情况下开发，对于图片也要有错误处理。
    ```tsx
      interface IProps extends StandardProps, ImageProps {
        defaultImage?: string;
        lazyLoad?: boolean;
      }

      const CustomImage: Taro.FC<IProps> = (props) => {
        const { src, mode, defaultImage, lazyLoad = true } = props;
        const [rightSrc, setRightSrc] = useState(defaultImage || imagePlaceholder);

        useEffect(() => {
          if (src) {
            setRightSrc(src);
          }
        }, [src]);

        return (
          <Image
            lazyLoad={lazyLoad}
            src={rightSrc}
            mode={mode}
            className="custom-class"
            onError={() => setRightSrc(imagePlaceholder)}
          />
        );
      };
    ```

## 小程序中的一些注意事项

1. ### scroll-view 中的定位问题
   在 scroll-view 中不能使用 absolute 以及 fixed 布局，使用之后不会生效，解决方法就是不要再scroll-view 中使用这两种布局，可以放在与 scroll-view 同层的位置去使用这两种布局。

   scroll-view 能够很好的帮助我们构造一个 infinity-scroll 组件，可以很方便的实现上滑加载更多，下拉刷新的功能，并且能够使用微信 dom 节点上的 ID 进行类似于 scrollIntoView() 的滚动效果。

2. ### canvas 的使用注意事项
   a. 首先第一点就是 canvas 的 type="2d" 的模式不会进行绘制，原因尚未知晓，解决方法就是不设定 type，而是使用 `canvas.getContext('2d')`。
    
   b. canvas 在绘制时如果要设置一些样式，需要配合 `context.save() context.restore()`，因为 canvas 的样式是依据最近的一次，如果想要各个设置不互相干扰，就需要在设置前后使用上面两个 API 进行保存，实在是因为样式混杂不好处理，那么就在每次 draw 的时候进行整块画布清空 `context.clearRect()`。

   c. canvas 的所有计算完毕之后，一定要使用 `context.draw()` 进行绘制，否则画布上不会出现任何图像。

   d. 小程序的 canvas 的宽高就是其 style 的宽高，与 HTML 的不一样。

   e. 在调试 canvas 的时候需要在真机上面进行调试，因为小程序 canvas 使用的是原生组件，模拟器上的样子不一定是真机上的样子，并且如果有 canvas 尽量不要放在浮层中，可能会因为 canvas 是原生组件而层级并不会在浮层上展示，同样因为是原生组件，所以也不能在 scroll-view 中使用 canvas，canvas 不会随着 scroll-view 滑动而滑动。

   f. 如果需要在小程序中进行 canvas 的动画，需要调用 `canvas.requestAnimationFrame()`，如果不生效，可以使用 `setInterval()`，但是小程序的动画在真机上可能会很卡，原因尚未知晓，尽量不要设计过于复杂的动效在小程序上。

## 埋点小技巧
1. ### PV 埋点
    将埋点放在生命周期中最为合适，我们使用的是 hook 所以我们可以提取成为一个简单的自定义 hook，然后在每个页面调用一下即可，但是由于 taro 的 hook 中没有对应 componentWillUnmount，所以我们直接复写 page 的 onUnload 方法。

2. ### fromWhere 埋点
    需要记录页面之间的跳转关系，可以利用路由，将路由与页面名字对应，然后能在当前页面拿到上次页面的路由信息，就能够知道上个页面的名字，进而进行记录，在设定记录的名称的时候也考虑到是否记录来源就足够了，因为有的组件在多个地方使用，就不需要再在每个使用的地方进行点击事件的记录，直接在跳转到的页面记录是从哪个地方来的，就能够达到相同的效果。