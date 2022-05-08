---
title: React-Native -- 下一代UI编程思维
categories: iOS
tags:
  - JavaScript
  - Swift
  - ReactJS
updated: '2016-03-30 02:37:28'
date: 2016-03-30 02:35:42
---

# React-Native VS Cocoa Touch -- 下一代UI编程思维

## React 与状态

`React.js`自从Facebook一推出，就受到Web前端工程师的强烈推崇。虽说曾经火过一时的`Angular.js`颠覆了前端的工程，但是`React`更多颠覆的，是下一代UI编程的思维。

传统UI编程，基本很多地方都需要将数据来源，绑定到对应的UI对象，比如用户点击了一个操作，更改了名称，那么你需要更新执行一个回调函数来处理点击操作，并且把新的数据更新原有的UI对象的属性，比如大概就是这样的东西

```swift
func onClick(sender) {
    var data = getData(sender);
    self.button.title = data.name;
    self.button.color = data.color;
}
```

这样虽然说直观，但是有很大的问题。试想，假如有很多种的回调函数，每个回调函数监听不同的操作，比如`onMounseDown`,`onMouseUp`,`onKeyDown`,`onScroll`……甚至根据不同的sender，我们会有不同的操作，我们就必须得手写很多机械的

```swift
self.xx = data.xx
self.xy = data.xy
```swift
    

这类的代码，这就给日后维护和扩展带来了灾难，假如我要换一个UI组件，又得一个个检查是否赋值成功；假如我要把这个UI组件在新的UIView里面重用，我又得改动所有的赋值代码，对于大型项目这种UI对象成百上千，UI属性上万，这样是非常可怕的。

而`React`，就将所有的赋值，数据绑定，抽象成为一个个状态，不同的事件监听，就是不同的状态而已，而这些状态之间相互独立，不会受到某些全局变量更改而造成UI混乱的情况，更好的是，开发者不需要考虑到底这个属性什么时候赋值，是在数据更新之前还是之后，需不需要定时刷新这种无意义的苦力活上。

## React-Native VS Cocoa Touch

> 完整代码：[Tutorial][1]

看看`React-Native`的sample，需求就是实现一个电影列表显示的View。类似这样：

![iOS][2] ![Android][3]

这段代码中，只有View和ViewModel（Model就是临时的JSON），React用状态把UI属性和数据绑定起来，从而避免了事件监听手动判断时机来赋值

```jsx
var AwesomeProject = React.createClass({
  //初始状态，不渲染
  getInitialState: function() {
    return {
      dataSource: new ListView.DataSource({
        rowHasChanged: (row1, row2) => row1 !== row2,
      }),
      loaded: false,
    };
  },

  //模型层变动导致状态改变，渲染

  fetchData: function() {
    fetch(REQUEST_URL)
      .then((response) => response.json())
      .then((responseData) => {
        this.setState({
          dataSource: this.state.dataSource.cloneWithRows(responseData.movies),
          loaded: true,
        });
      })
      .done();
  },

  //渲染层，绑定状态和UI属性

  render: function() {
    return (
      <ListView //这里是JSX语法，在JS里面返回标签
        dataSource={this.state.dataSource}
        renderRow={this.renderMovie}
        style={styles.listView}
      />
    );
  }
 }
```

> 完整代码：[iOS Demo][4]

相比来说，原生Cocoa Touch的实现，就要丑陋的多了，尤其是渲染部分绑定UI对象的属性和数据来源，假如你有多处数据来源，多种UI属性，你就得写很多判断来保证你的UI对象的属性符合预期的赋值顺序。

```swift
//初始化UI对象，调用模型去获取数据
override func viewDidLoad() {
   super.viewDidLoad()
   fetchData()
}

//获取数据，需要手动维护状态，比如开启indicator(旋转等待条)，数据获取成功后渲染
func fetchData() {
   indicator.startAnimating()
   request(.GET, API_URL, parameters: ["apikey": API_KEY, "page_limit": PAGE_SIZE])
       .responseJSON{ _, _, data, _ in
           self.render(JSON(data!))
   }
}

//渲染UI对象，需要手动维护状态，比如关闭indicator(旋转等待条)
func render(result:JSON) {
   indicator.stopAnimating()
   movieJSON = result
   self.tableView.reloadData()
}

//这都算好的了，TableView会自动回调你的代码，返回这个TableView里面的Cell个数
func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
   var movieNum = movieJSON["movies"].arrayValue.count
   return movieNum
}

//可怕的地方，需要手动赋值给UI对象的属性
func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
   let cell = self.tableView.dequeueReusableCellWithIdentifier("movieIdentifier", forIndexPath: indexPath)
        as! MovieTableViewCell

   let row = indexPath.row

   //手动维护状态，JSON解析再赋值给对应的UI对象，这里省略了UI对象的代码和样式
   //（一般可以通过Interface Builder做或者专门一个类写UI代码）
   cell.movieTitle.text = movieJSON["movies"][row]["title"].stringValue
   cell.movieYear.text = movieJSON["movies"][row]["year"].stringValue
   var movieImageUrl = movieJSON["movies"][row]["posters"]["thumbnail"].stringValue

   //最可怕是渲染中还需要新的数据，再回调模型，就很难维护了
   if cell.movieImage.image == nil {
       request(.GET, movieImageUrl).response{ _, _, data, _ in
           let movieImage = UIImage(data: data as! NSData)
           cell.movieImage.image = movieImage
       }
   }

   return cell
}
```swift 

# React 与函数式

`React`更多的，就是一种类似函数式的想法，把UI对象属性，数据来源，当作一个Monad包裹起来，传统意义上的不同数据来源进行UI属性赋值，相当于这个Monad经过不同的函数作用，达到状态的切换，好处就是大大减少了开发者手动维护UI属性的工作，而且可以达到更高的开发效率。而且再也不怕扩展了，因为这时候可以把多个组件分配给不同的人，每个人完全不需要管别人内部的变量名是什么，UI属性是什么，只要把自己的状态管理好，Model层接口统一，剩下的合并即可。

# React 与效率

既然提到了状态，因为`React`采取`VirtualDOM`来diff需要进行状态更新的UI对象，每次确保了只更新属性发生改变的部分。实现也很高效，使用了一个普通的二叉树`vtree`。每个结点`vnode`就是对应的Tag，比如<Image>之类，结点还存储了一个struct用来保存这些Tag的属性（比如Image的size）

其中的diff算法可以一看……代码地址在：[GitHub-vtree][5]

```javascript
//主要逻辑，就是对b树进行reorder，找出a树与b树的同结点不同属性的diff
//剩下可以无痛patch的部分只需要下面的for循环就可以处理
var aChildren = a.children
var orderedSet = reorder(aChildren, b.children)
var bChildren = orderedSet.children

var len = aLen > bLen ? aLen : bLen

for (var i = 0; i < len; i++) {
   var leftNode = aChildren[i]
   var rightNode = bChildren[i]
   index += 1

   if (!leftNode) {
       if (rightNode) {
           //这里处理的多余的结点，直接加到b树上即可
           apply = appendPatch(apply,
               new VPatch(VPatch.INSERT, null, rightNode))
       }
   } else {
       walk(leftNode, rightNode, patch, index)
   }
}
```

这是核心reorder代码，目标找到是同一结点不同属性的diff，多出来的不需要管

```javascript
//遍历a树，如果b树中结点集合含有a的key，标记下b树中这些key，更新旧的标记
for (var i = 0 ; i < aChildren.length; i++) {
   var aItem = aChildren[i]
   var itemIndex

   if (aItem.key) {
       if (bKeys.hasOwnProperty(aItem.key)) {
           // Match up the old keys
           itemIndex = bKeys[aItem.key]
           newChildren.push(bChildren[itemIndex])

       } else {
           // Remove old keyed items
           itemIndex = i - deletedItems++
           newChildren.push(null)
       }
   }
   //......
    }


//遍历b树，添加上面标记的所有key到一个集合，暂时未排序
for (var j = 0; j < bChildren.length; j++) {
   var newItem = bChildren[j]
   if (newItem.key) {
       if (!aKeys.hasOwnProperty(newItem.key)) {
           newChildren.push(newItem)
       }
       //......
 }

//对上述集合，真实的b树删除多余结点，直到上述集合为空为止，复杂度O(n)
var simulate = newChildren.slice()

for (var k = 0; k < bChildren.length;) {
var wantedItem = bChildren[k]
simulateItem = simulate[simulateIndex]

//最后删除剩下虚拟的结点
while(simulateIndex < simulate.length) {
   simulateItem = simulate[simulateIndex]
   removes.push(remove(simulate, simulateIndex, simulateItem && simulateItem.key))
}
```

整体的复杂度，达到了O(M+N)，已经是理论下界了。比起手动管理状态来说，效率可以说是直接持平，甚至对部分滥用事件监听的写法效率会更高。

# 总结

虽然我并不喜欢UI编程，但是自从图形化出现之后，UI已经成为了继数据结构、算法外，面向终端用户的应用又一个大工程。

从最早的指令式跳转赋值UI，函数指针响应处理，到中期的面向对象，消息发送回调事件，手动管理属性，在到如今的React以状态和VirualDOM来绑定数据和组件。

UI编程其实也是在不断进化的，也许今后会有更好的开发方式让我们这群不会写UI的人也能够轻松写起来UI。

 [1]: https://facebook.github.io/react-native/docs/tutorial.html#content
 [2]: https://facebook.github.io/react-native/img/TutorialFinal.png
 [3]: https://facebook.github.io/react-native/img/TutorialFinal2.png
 [4]: https://github.com/lizhuoli1126/iOS-Demo-Project
 [5]: https://github.com/Matt-Esch/virtual-dom/blob/master/vtree/diff.js