---
title: 阿里前端工程师（Node.js）向实习生面试经验
categories: Life
tags:
  - 面试
updated: '2016-03-30 02:44:56'
date: 2016-03-30 02:44:41
---

## 前提说明：
自己是在北邮论坛中找的学长内推，当时与学长问了关于阿里前端中是否有偏向JavaScript开发（中间件，工具库）的方向，学长说只有杭州有类似岗位，最后把我内推到了淘宝UED的团队去了。

简历说明中侧重讲了关于Node.js的经历，JavaScript轮子的介绍以及一些使用了Node.js技术栈的Web项目。这点对于前端实习也是一个加分项。

## 一面：
一面面试官问了大概有4年开发经验（包括了Java和JavaScript），上来除了标准的自我介绍以外，大概主要谈论了关于Node.js，JavaScript语法以及Web开发的领域，对简历中提到的项目稍微深入问了一下。

1. babel或者coffee Script的这些编译到JS的语言是如何工作的？
这个问题是最纠结的，因为不太清楚面试官想问什么，大概说了关于Parse的东西，后面补充了关于babel的递归引用JS文件处理的东西，估计有问题。

2. JavaScript的Async库基本原理是什么？
这个网上都有，就是Async的parrllel，waterfall的简单实现，讲出了把callback function替换this域，用一个list来遍历执行，把最终的error或者result参数填回到Async.parallel([functionList], callback(err,result))中，差不多

3. 你写的Functional.js中monad, curry, lazy的解释和意义？
基本面向简历的作品，讲解了一点关于monad的简单意义（包裹，传递流，防止外部更改），curry化对JS库函数的意义，还有lazy list或者range对于那种大量数据处理的好处什么的。

4. Node.js框架同其他语言框架的比较？
答出Node.js特色的非阻塞IO和异步性，和Python的Flask对比，再讲解一下express中间件这种模式的特点，差不多了

5. Node.js与Swift在Web领域的未来？（因为我简历写了iOS开发和Swift）
随便扯吧……就是强类型的问题，基于原型面向对象优劣，语法糖的问题，还有支持库什么的。

其他就是自由提问，知道了阿里淘宝对前端实习要求基本不高，主要是JS熟悉，会Node有加分，而且没有固定谁来切图谁写JS，一般都会一点跨栈的东西。

## 二面：
（二面充分暴露了自己的若菜本质）。面试官是Winter，就是那个知乎的温兆伦的Winter（P8）。二面是电话+网页coding的部分，需要在电脑，网络OK情况下进行，要自己提前准备好（不行就说明改个时间……）。开始电话问了一些关于JS的东西，什么闭包，Node.js的require依赖顺序什么的……之后就开始正式网页coding。

1. 第一题：设计一个简单的红绿灯策略，比如红灯亮分别为console.log("red")这种，要求按照红3s-黄1s-绿1s顺序不断循环展示。

这个本来很简单的问题开始愣住了，因为原生的setTimeout好久没用了，问了问可以使用第三方库，但原生其实有笨办法，就是直接硬编三个setTimeout，时间分别为0，3s，4s，然后最外层一个10s延迟的setInterval的来重复。虽然效果但是这肯定不对，因为这直接无视了事件发生相对顺序，靠着全局时间来实现，长期下去由于JS引擎的延时会最终乱掉。

第二种想法，我借用了Promise，大概就是用Promise里面resolve一个setTimeout的函数模拟事件结束，最后由Promsie.then控制流程，好处就是绝对不会出现事件先后顺序错乱，而且写起来简单。

```js
function button(color, time) {
	let p = new Promise(function (resolve, reject) {
		setTimeout(function() {
			resolve("Timestamp: " + getTimestamp() + " Color: " + color)
		}, time);
	});
	return p;
}


function flash() {
	button("red", 3000) // after last task end, which means the last task will need 3s
	.then( (v) => {
		console.log(v);
		return button("yellow", 3000); // last spend 3s
	}).then( (v) => {
		console.log(v);
		return button("green", 1000); // last spend 1s
	}).then( (v) => {
		console.log(v);
	});
}
```

2. 第二题：算法题，第一问是：给定一个整数金额的整钱n，还有2，3，5元三种货币，要你计算出所有能凑出整钱的组合个数。

（这里又暴露自己思维模式问题）应该先从最简单想，假如n=10，把5，3，2元的取的张数定为i,j,k，一定要按照由大到笑顺序，那么就相当于从i=0,j=0,k=0开始循环，一旦组合总金额超过n，那么就break停止（因为从大到小取金额，最小的都无法凑出，那么之后再也不能取了），能凑齐就加一个组合。

```js
function countMoney(total) {
	if (total < 2) {
		return 0;
	}
	var result = 0;
	var maxAmount = total / 2;
	for (var i = 0; i <= maxAmount; i++) { // 5
		for (var j = 0; j <= maxAmount; j++) { // 3
			for (var k = 0; k <= maxAmount; k++) { // 2
				var sum = i * 5 + j * 3 + k * 2;
				if (sum == total) {
					result++;
					break;
				} else if (sum > total) {
					break;
				}
			}
		}
	}
	
	return result;
}
console.log(countMoney(10));
```
笨方法，但是外层不会超过 n / max[i]，所以复杂度最差也没有到O(n^3)

第二问：假如这个能使用的货币列表是给定的，意思是输入一个整数list，比如[1,2,3,5]，还有金额n，求出所有组合数。
现在货币列表不是定的，那么就得想别的方法，当时答的时候说要用递归，但是最终没写出来（唉……），在之后才写出来。

前提思路用的是递归，function countMoney(amount, moneyArr)，amount为剩下的金额，moneyArr为可以选择的货币列表，返回的是产生的组合数，那么初始条件认为amount = n, moneyArr = list（排序，由高往低）。取出当前moneyArr（也就是当前最大的面值
）的货币first，剩下的货币可选面额叫做smallerMoneyArr，然后从0到first最大能取的个数开始（即 0 ~ amount / first)，不断递归调用countMoney(remainingAmount, smallerMoneyArr)，加起来所有组合数即可。
终止条件很简单，如果剩余余额不为0但可选货币为空，那么分割方法失败，返回0；如果余额是0，那么分割成功，返回1；搞定。
```js
//Recursive DP

var inputMoneyArray = [1,2,3,5];
inputMoneyArray.sort().reverse(); // must from higher to lower

function countMoney(amount, moneyArr) {
	if (amount != 0 && moneyArr.length == 0) {
		return 0; // no use
	} else if (amount == 0) {
	  return 1; // success one
	}
	
	var first = moneyArr[0];
	var smallerMoneyArr = [];
	for (var i = 1; i < moneyArr.length; i++) {
		smallerMoneyArr[i-1] = moneyArr[i]; 
	}
	
	var sum = 0;
	for (var i = 0; i <= amount / first; i++) {
		var remainingAmount = amount - (first * i);
		sum += countMoney(remainingAmount, smallerMoneyArr);
	}
	
	return sum;
}

var result = countMoney(10, inputMoneyArray);
console.log(result); // 20
```

然而……当时面试脑抽没想出来，后面就问了问一些前端开发要求还有工作环境什么的就没了……唉，还是没有准备的问题，如果真要面试，希望提前准备好一点常见题目，主要是思维方式要对，先最简单（从特例开始，变量假设为固定值），然后推广，复习一下递归，动态规划什么的就很简单。


## 三面
三面还是技术面，其实我也没有任何准备（以为二面挂了），面试官是淘宝一个P7级别的吧，也是上来自我介绍，然后开始问一些Node.js和JS（有前端JS）的问题，基本我全程都没遇到过HTML5，CSS，Web前端框架，构建工具等问题……可能是简历导致的吧。

1. 说一下关于Node.js的文件读写方式和实现？
基本解释一下fs.readFile，说明异步性，然后面试官开始追问：Node.js是单线程，如何实现多个同时的文件IO？
紧接着就开始解释Node.js的fs调用V8的libuv中的`uv_fs_open`，绑定JS的callback到一个c的函数指针上，然后推入事件列表队列（QueueUserWorkItem），再根据操作系统，Windows下使用IOCP来完成异步IO，*NIX上使用libev来实现。说明Node.js从上层到V8是单线程，从libuv到IOCP或者libev是多线程IO读写

2. 说一下JavaScript几种异步方法和原理？
基础问题……先说callback function，说明问题，再讲Promise，包括Promise的原理和实现。然后还有co，利用generator和yield来实现异步控制，顺便也提到了async和await（虽然ES7还是没有加……简直服了），介绍一下使用经验，差不多了。

3. 介绍一下Session和Cookie？
不用解释了吧……服务端随机分一个SessionID然后HTTP字段Set-Cookie，内存存一个Map -> 浏览器存Session，以后请求都带Cookie字段（值为SessionID） -> 服务端看到Cookie字段，读SessionID对应的Map，执行逻辑。

4. 前端方面，说明addEventListener使用？作用域？
其实问的是如何给一个链接上鼠标事件，先说addEventListener。再追问说，现在IE8和Chrome不一致，IE的addEventListener绑定是window，而标准是document，如何设计一个库来兼容避免对window的污染？
然后讲解首先库要做成一个全局匿名闭包，接受一个listener来处理作为执行函数，然后要把window.document保留为内部的变量，然后把listener的this域换位这个闭包的this，就是大概这样？由于不怎么写前端可能是错的……

```js
(function myAddEventListener(type, listener) {
	var document = window.document;
	var that = this;
	
	if (IE) {
		attachEvent.bind(that);
		attachEvent(type, listener);
	} else {
		addEventLisener(type, listener);	
	}
})();
```

5. 谈谈冒泡和捕获事件的区别，应用？
就是Event捕获顺序，前者从底向上，后者从顶向下，说明可以stopPropagation，大家都会……

然后就没了，最终谈了谈阿里对实习生要求不高，正式校招实习生只有免笔试的优惠，还得面试一轮，不过也觉得可以了。还知道内推是默认团队的，除非再人事申请，公开实习可以选择意向。HR面还没有，基本上没有意外一般不会又问题。

## 总结

前端工程师还是老老实实干前端吧，我这个面试经历算是特例。其实选择前端是因为阿里的后台开发只有Java岗位，而且都会考很多J2EE的东西（Servlet，SSH，设计模式，中间件，JavaBean什么的）感觉很不喜欢（毕竟是实习，想做一点感兴趣的东西），iOS开发也只要OC而且难度很高。其他公司倒有投后台和iOS的，基本上感觉以后还是很大可能搞iOS了。

其实如果真想毕业找工作，大三下之前就最好决定自己方向（或者一个可选），不要技能栈拉太长，导致深度不够，这是很重要的。面试经历，基本上不准备不太可能（我这是特例……），多刷点简单OJ题（实习面试coding一般很简单），对一些框架，技术原理性要掌握（不一定实践过），项目要写的话一定确保自己还有印象，比较熟悉，不然被问到就很麻烦，基本上这样了。

希望大家都能面试顺利，拿offer拿到手软～