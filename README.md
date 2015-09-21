# 这段时间研究了下Redux，写写自己对它的感觉。

## 先说下传统的redux

个人感觉，redux简化了flux的流程。

### flux的流程是：

1. view触发action中的方法；
2. action发送dispatch；
3. store接收新的数据进行合并，触发View中绑定在store上的方法；
4. 通过修改局部state，改变局部view。

### redux是：

1. view直接触发dispatch；
2. 将action发送到reducer中后，根节点上会更新props，改变全局view。

redux将view和store的绑定从手动编码中提取出来，放在了统一规范放在了自己的体系中。

而在基本的redux流程中，action只是充当了一个类似于topic之类的角色，reducer会根据这个topic确定需要如何返回新的数据；数据的结构处理也从store中移到了reducer中。

### redux数据流如下图:

![redux](https://raw.githubusercontent.com/lawrencebla/redux-review/master/redux.jpg)


### redux的基本原理实际上就是围绕着store进行的。

这个store不是flux中的store，而是通过[`createStore`](http://rackt.github.io/redux/docs/api/createStore.html)方法创建的；
[`createStore`](http://rackt.github.io/redux/docs/api/createStore.html)方法接收**reducer函数**和**初始化的数据(currentState)**，并将这2个参数并保存在store中。


`createStore`时传入的reducer方法会在store的`dispatch`被调用的时候，被调用，接收store中的state和action，根据业务逻辑修改store中的state；

store中包含`subscribe`、`dispatch`、`getState`和`replaceReducer`这4个方法。[相关API](http://rackt.github.io/redux/docs/api/Store.html)

其中，`subscribe`和`dispatch`顾名思义就是订阅和发布的功能；
`subscribe`接收一个回调(listener)，当`dispatch`触发时，执行reducer函数去修改当前数据(currentState)，并执行`subscribe`传入的回调函数(listener)；

而`getState`是获取当前store的state(currentState)；

至于`replaceReducer`方法，就是动态替换reducer函数，这个我觉得应该算用的比较少吧。

### 说说Middleware

redux中的middleware还是比较简单的，它只是针对于dispatch方法做了middleware处理；也就是说，只接受一个action对象；使用也很简单，如文档中的例子:

	const createStoreWithMiddleware = applyMiddleware(
	  thunkMiddleware,
	  loggerMiddleware
	)(createStore);
	store = createStoreWithMiddleware(rootReducer, initialState);
redux的middleware用reduceRight方法，将[`applyMiddleware`](http://rackt.github.io/redux/docs/api/applyMiddleware.html)方法中的参数串起来，原始的dispatch方法会最后执行。
如下图：
![middleware](https://raw.githubusercontent.com/lawrencebla/redux-review/master/middleware.jpg)

** 编写middleware **
如果需要自定义middleware也很简单，这个middleware只接收一个action，执行后也需要返回一个action；如果需要执行下一步，调用`next(action)`即可。

## 再来说说react-redux

[react-redux](https://github.com/rackt/react-redux)，是对redux流程的一种封装，使其可以适配与react的代码结构。

react-redux首先提供了一个`Provider`，可以将从`createStore`返回的store放入context中，使子集可以获取到store并进行操作；

	<Provider store={store}>
		{() => <App />}
	</Provider>

其次react-redux提供了`connect`方法，将原始根节点包装在Connect下，在Connect中的state存储不可变对象，并将state对象中的props和store中的`dispatch`函数传递给原始根节点；

Connect在`componentDidMount`中，给store添加listener方法(`handleChange`)，每当store中的`dispatch`被调用时执行`handleChange`；`handleChange`会去修改state中的porps，使原始根节点重新render；并且Connect已经在`shouldComponentUpdate`实现了PureRender功能。

`handleChange`更新state中的props逻辑主要由3个函数构成，这3个函数都由connect方法传入;

	connect(mapStateToProps, mapDispatchToProps, mergeToProps)(App);
第一个函数接收store中state和props，使页面可以根据当前的store中state和props返回新的**stateProps**;
第二个函数接收store中的dispatch和props，使页面可以复写dispatch方法，返回新的**dispatchProps**；
第三个函数接收前2个函数生成的**stateProps**和**dispatchProps**，在加上**原始的props**，合并成新的props，并传给原始根节点的props。
[相关API](https://github.com/rackt/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options)

在react-redux中的流程如下图：
![react-redux](https://raw.githubusercontent.com/lawrencebla/redux-review/master/react-redux.jpg)

1.View触发`dispatch`
2.进入reducer，修改store中的state
3.将新的state和props传入`handleChange`中，生成更符合页面的props
4.传给原始根节点重新render

## 最后的再说两句

**redux的优点：**
	
1. redux把流程规范了，统一渲染根节点虽然对代码管理上规范了一些,只要有需要显示数据的组件，当相关数据更新时都会自动进行更新。

2. 减少手动编码量，提高编码效率。


**redux的缺点：**

1. 一个组件所需要的数据，必须由父组件传过来，而不能像flux中直接从store取。

2. 当一个组件相关数据更新时，即使父组件不需要用到这个组件，父组件还是会重新render，可能会有效率影响，或者需要写复杂的`shouldComponentUpdate`进行判断。


**redux的疑问：**

1. redux如何处理多个api同时请求，在都得到结果后，再进行渲染（每个api都可能会单独拿出来做其他的请求）？

2. 实际上在flux的流程中，在action中做api请求，然后返回数据，我并没有觉得违和。
但是放在了redux中，通过`redux-thunk`或`redux-promise`来阻止第一次`dispatch`，并将`dispatch`函数传入action中的回调方法再次调用。
我感觉不是很符合正常的语言逻辑，使action的功能好像并不单一，是不是有什么更改的实现方式呢？

3. 多个reducer文件怎样组合会更好些？是每种类型的数据放在每个文件里，与flux中的store类似，最后在一个总的reducer文件中做`combineReducers`？
