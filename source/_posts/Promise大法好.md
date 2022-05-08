---
title: Promise大法好
categories: JavaScript
tags:
  - JavaScript
  - 异步
updated: '2016-03-30 02:18:18'
date: 2016-03-30 02:14:46
---

# Promise简介

Promise是一种解决异步回调问题而发展的编程语言特性，在各种语法中都有支持，比如在JavaScript（ECMAScript 6）／Java（8）／Node.js（0.12）都有原生的支持，而没有原生支持的语言更可以通过第三方框架来简单引入（比如大名鼎鼎的[Q][1]）

## 为什么说异步回调不是好的解决方案

1\.Callback Hell

```javascript
loadScript("a.js",function(){
    loadScript("b.js",function(){
        loadScript("c.js",function(){
            loadScript("d.js",function(){
                loadScript("e.js",function(){
                    console.log("Fuck to load async files")
                }
            })
        })
    })
})
```

2\.依赖习惯编码

```javascript
function asyncFunc(callback){
    doSomething(function(err, result){
        if(err){
            callback(err, null);
        }
        callback(null, result);
    })
}

asyncFunc(function(err,result){
    if(err){
        console.error(err);
    }
    console.log(result);
})

anotherAsyncFunc(function(result,err){
    if(result){
        console.log("Fuck why the first is result");
    }
})
```

还有种种，比如控制流难以书写（多个不同异步任务的条件判断和进度控制），try/catch无法捕获回调异常…………种种原因逼迫人们选择更好的解决方案

## Promise的核心

> 此处以Node.js（v0.12）为例，部分术语可能不同编程语言实现时不同，在JavaScript中，Promise是一个内置对象

~~友情提示：以下东西仅为装逼，可跳过~~

1.  三种状态：  
    `fullfied`：被`resolve`后的状态  
    `rejected`：被`reject`后的状态  
    `pending`：初始状态

2.  两个调用：  
    `then(onFulfilled, onRejected)`：在被`resolve`后执行`onFulfilled`函数,而被`rejected`后执行`onRejected`函数。并且实际上每次调用Promise的`then`都会返回一个`新的`Promise对象  
    `catch(onRejected)`：`then(undefined, onRejected)`的语法糖，被`rejected`后执行`onRejected`函数

3.  状态转变：  
    `resolve(value)`：语法糖，返回一个立即被`fullfied`且转变为`fullfied`状态的Promise对象，等价于：
    
```javascript
new Promise(function(resolve){
    resolve(value);
});
```


`reject(value)`：语法糖，返回一个立即被`resolve`且转变为`rejected`的Promise对象，等价于：

```javascript
new Promise(function(resolve){
    resolve(value);
});
```
    

1.  控制流：  
    `Promise.all([promise1, promise2, ...])`：当`全部`Promise对象被`resolve`时调用`then(onFulfilled(value))`（value为数组的值）；或者任何一个Promise对象被`reject`时调用`catch(onRejected)`  
    `Promise.race([promise1, promise2, ...])`：当`任何`Promise对象被`resolve`或者`reject`时调用`then(onFulfilled(value))`或者`catch(onRejected)`

## Promise的简单用法

1\.Easy Mode

```javascript
function asyncFunction() {
    if (1 === 1){
        return Promise.resolve("OK");
    }
    else{
        return Promise.reject(new Error("Wrong"));
    }
}

asyncFunction().then(function(val){
    console.log(val);
}).catch(function(err){
    console.error(err);
})

/*
OK
*/
```

2\.Normal Mode

```javascript
asyncFunction().then(function(val){
    console.log(val);
    return "First";
}).then(function(val){
    console.log(val);
    return "Second";
}).then(function(val){
    throw new Error("Fuck");
    console.log(val);
}).catch(function(err){
    console.error(err);
})

console.log("Start!");

/*
Start
OK
First
[Error: Fuck]
*/
```

3\.Hard Mode

```javascript
function anotherAsyncFunction(){
    if (2 != 2){
        return Promise.resolve("OK");
    }
    else{
        return Promise.reject(new Error("Wrong"));
    }
}

Promise.all([asyncFunction(),anotherAsyncFunction()]).then(function(val){
    console.log("All:" + val);
}).catch(function(err){
    console.error("All:" + err);
});

Promise.race([asyncFunction(),anotherAsyncFunction()]).then(function(val){
    console.log("Race:" + val);
}).catch(function(err){
    console.error("Race:" + err);
});

/*
Race:OK
All:Error: Wrong
*/
```

# 后话

拥抱Promise，告别Callback，你我值得拥有。连古老的OO圣教——Java大法都拥抱Lambda和Promise，你还在等什么？

```java
public F.Promise<JsonNode> post(String formData){
   WSRequest request = client.url(url);
   F.Promise<WSResponse> responsePromise = request
           .setContentType("application/x-www-form-urlencoded")
           .post(formData);
   F.Promise<JsonNode> jsonNodePromise = responsePromise.map(value -> {
       return value.asJson();
   });

   responsePromise.onFailure(error -> {
       Logger.error(error);
   });

   return jsonNodePromise;
}
```

[1]: https://github.com/kriskowal/q