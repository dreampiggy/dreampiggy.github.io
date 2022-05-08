---
title: React Native之Redux架构入门
categories: iOS
tags:
  - iOS
  - React Native
  - Redux
updated: '2016-10-26 17:39:01'
date: 2016-10-26 17:23:28
---

> 如今也是入了移动iOS开发的坑，最近不仅学习了Core Animation的部分知识，还接触到React Native，这一Facebook出品的React for Native Platform框架，其中React本身入门是相对简单的，而Redux入门就相对困难了，因此在这里总结一下，最后有自己写的Slides可以参考

# 背景

> 为什么需要Redux

1. RN的state（可变，子组件不可见）和props（不可变，子组件可见）的设计，在面对大型项目时候，容易因为不经意修改state造成状态混乱，组件渲染错误
2. RN使用了Virtual DOM，不需要Target绑定->Action修改UI属性，只要当状态变化，render新状态下的组件，数据单向传递，而MVC的设计模式存在双向数据流
3. RN不易进行测试，Redux提供了非常方便的mock测试方式

# 准备

```bash
安装Redux：
	npm install --save redux
安装React Native和Redux绑定库：
	npm install --save react-redux
安装Redux Thunk异步Action中间件：
	npm install --save redux-thunk
```

# Redux 原则

1. 单一数据源	
	**描述：** 整个应用的 state 被储存在一个对象树中，对象树存在于唯一的 store 中	
	**说明：** Redux把所有state集中管理，避免了各处临时的状态造成的不可控
2. State 是只读的	
	**描述：** 惟一改变 state 的方法就是触发 action	
	**说明：** action 是一个含有 type 属性的普通JS对象，type 属性可以用具体的string常量，来表示事件
3. 使用纯函数来执行修改	
	**描述：** 编写 reducers 来描述对应action如何修改 state	
	**说明：** reducer函数是纯函数，无副作用，定义为reducer(state, action) => newState，一般可以用 switch(action.type) 来判断不同的action，同时return一个新的state
	
# Redux 数据流
![](http://dreampiggy-image.test.upcdn.net/image/6/ac/b29d443e4577ad15dd62c800c5fc4.png)


1. Component触发一个Action Creator（用来dispatch某个具体的Action的函数），dispatch到Store中	
2. Reducer进行判断，通过(state, action) => newState得到新的状态，写回到Store中	
3. Store绑定到Component，把state赋回Component的props，然后Component就可以render了，整个流程也比较清晰	

# Action

Action是一个普通JS对象，至少包括一个`type`属性代表事件，其他属性可以用来传递数据。实践上对一个流程定义一个函数，流程可以包括网络请求，最后返回Action，这个函数叫Action Creator

Type:

```js
const ADD_TODO = 'ADD_TODO';
```

Action：

```js
let action = { type: ADD_TODO, text: 'Build my first Redux app' }
```

Action Creator：

```js
function addTodo(input) {
	return { type: ADD_TODO, text: input }
}
```

举例：

![](http://dreampiggy-image.test.upcdn.net/image/a/b8/1e6f0b3d3f531817fb9bf7aed7ad3.png)
![](http://dreampiggy-image.test.upcdn.net/image/2/5c/d3050cfb96161668957d6b51fe00a.png)

# Reducer

Reducer是一个函数，根据Store中的前一次的state，和Store中被dispatch过来的Action，返回一个新的state并写入Store，即： `(state, action) => newState`

Reducer：

```js
function toDoReducer(state, action) {
	switch(action.type) {
		case ADD_TODO:
			return {text: action.text,completed: false}
		default:
			return state;
	}
}
```

注意：

1. 返回的是新state，如果需要保留部分旧state值，使用…state（ES7的对象展开语法，对对象会浅拷贝对应属性，这里等价于Object.assign({}, state, newState)），对应合并state的话只会合并一层，而子属性会被覆盖掉，对复杂state需要手动合并
2. 为了遵循Reducer的纯函数性，不应该直接修改state的值，然后return state，除非使用Immutable.js（见后）
3. 实际中，应当把不同模块的代码，构造不同的Reducer，最后再合并（因为纯函数性，无副作用）

```js
import { combineReducers } from 'redux’;
import todos from './todos’;
import counter from './counter’;
export default combineReducers({ todos, counter })
``` 

举例：

![](http://dreampiggy-image.test.upcdn.net/image/4/2d/bbcfa2df98958ec1a4c027b7a883e.png)

# Store

Store存储了一个Provider的`所有的state`，同时接收dispatch过来Action。Redux提供的每个Provider根组件有唯一对应的Store。实践中可以拆分多个业务模块到多个Provider组件，每个Provider组件利用Redux架构，绑定自己的Store，模块内的业务流程写到对应的Action Creator中


Store：

```js
import { createStore } from 'redux';
import reducer from './reducers/BusListReducer';

const store = createStore(reducer);
```

1. Store的读写，可以调用getState()来获取当前state，用dispatch()来分发Action改变state
2. Store创建的时候，除了需要绑定Reducer，还可以应用中间件，比如日志中间件，React Thunk中间件（针对异步Action Creator）	

```js
const logger = store => next => action => {
	console.log('dispatching', action);
	let result = next(action);
	console.log('next state', store.getState());
	return result;
}

let createStoreWithMiddleware = applyMiddleware([logger, thunk])(createStore);
```

举例：

![](http://dreampiggy-image.test.upcdn.net/image/e/c6/3c631d1a7bd31016e1fdf45e63428.png)

# Component绑定

整个控制流，就差最后Store中的state绑定到组件了，我们通过Redux提供的`connect()`，绑定需要的state以及Action Creator到你的组件的props上，这样组件就可以通过props来调用Action Creator，或者根据不同props来render()不同的组件


```js
function mapStateToProps(state) {
  return {
    name: state.name
    age: state.age
  }
}

function mapDispatchToProps(dispatch) {
  return {
    actions: bindActionCreators(actions, dispatch)
  }
}

export default connect(mapStateToProps, mapDispatchToProps)(BusList);
```

1. 注意connect返回的就是绑定后的React Component，用的时候记得export default

举例：

![](http://dreampiggy-image.test.upcdn.net/image/9/19/10bde796c214db134bb166bbac540.png)

# State合并和Immutable.js

> 为什么State合并很困难

1. Reducer中，通过...state对象扩展符，再合并action的state，这种合并是一层合并，不会合并子属性而会直接覆盖掉，意味着对于复杂的state，如果想要修改子层级的属性，就得手动创建一个新的对象，并合并部分state的属性，而不是用…state直接扩展
2. Reducer为了保持纯函数性，应当禁止对state直接修改部分属性，但直接…state的拷贝在大量项目state树庞大的情形下会有一定的效率影响
3. 在1，2两条的情形下，如何在不修改原始state的前提上，又能返回一个原state属性部分变化后的newState就成了一个问题

```js
let a = { b: { c: 4 } };

// 等价写法
// let g = Object.assign({}, a, {
// 	b: { d: 5 }
// });
let g = {
	...a,
	b: { d: 5 }
}

console.log(g); // { b: { d: 5 } }
```

> Immutable.js解决了什么

1. Immutable.js的实现复杂不可变对象的高效copy和部分修改，内部通过Persistent Data Structure（持久化数据结构），即一个不可变的对象树。在使用旧数据创建新数据时，通过部分结构共享，如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享
2. 这样既避免了DeepCopy造成的性能开销，又能达到所有返回的数据仍然是不可变的，符合了reducer(state,action) => newState的纯函数性

> 具体如何实践配合Redux

1. 对state中`部分复杂的属性`，可以使用Immutable的数据单独包裹，在组件render()处单独解包所有Immutable对象
2. 如果state本身就含有各种复杂属性，可以直接`把state本身包裹`，用Immutable.Map表示state本身，然后通过Immutable的API操作修改部分属性，返回一个新的Immutable对象作为newState，在组件的mapStateToProps()方法中，把Immutable对象包裹的state解包为JavaScript的对象
3. 对于其他组件，可以在`shouldComponentUpdate()`方法中，通过Immutable的比较，来避免无用的re-render

举例：ListView简单应用（避免re-render)

![](http://dreampiggy-image.test.upcdn.net/image/e/8c/42c765d2eea6cb90e2eef9dfae949.png)
![](http://dreampiggy-image.test.upcdn.net/image/b/71/a6d9a50d4ff1c8eebbd462279c073.png)
![](http://dreampiggy-image.test.upcdn.net/image/7/db/e8fcbe8a12f6d2796e64a37e5c4ae.png)
![](http://dreampiggy-image.test.upcdn.net/image/9/7f/7b039ac17e7828a2fe4f3d033ad73.png)

# 更多思考

1. Action的定义规范？（对于不需要rerender的所有Action定义一个单独type？）
Reducer中state复杂多级结构，如何规范合并得到newState？（Immutable.js必要性）
2. 大型组件的render方法解耦合方式（switch？）
3. 移动端是否有服务端渲染的必要？（dispatch过去的Action如果需要复杂计算，把整个store交给服务端来计算）
4. 网络请求使用fetch API封装是否足够？
5. Native Module的JS端引入（callback？还是Promise + async await）
6. Mock测试的具体实践？（Action，Reducer，Component三个方面）

# Slides

+ [React Native进阶-Redux架构和状态管理.pptx](https://raw.githubusercontent.com/dreampiggy/iOS-Resume/master/React%20Native%E8%BF%9B%E9%98%B6-Redux%E6%9E%B6%E6%9E%84%E5%92%8C%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86.pptx)


# 参考资料
1. [React Native官方文档](http://facebook.github.io/react-native/docs)
2. [Redux官方文档](http://redux.js.org)
3. [Redux官方文档（中文）](http://cn.redux.js.org)
4. [Immutable.js解析](https://github.com/camsong/blog/issues/3)
5. [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
6. [Object Spread Syntax（对象展开符）](http://redux.js.org/docs/recipes/UsingObjectSpreadOperator.html)
7. [React Native Redux架构参考](http://www.jianshu.com/p/14933fd9c312)