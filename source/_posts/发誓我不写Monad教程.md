---
title: 发誓我不写Monad教程
categories: JavaScript
tags:
  - JavaScript
  - Functional
updated: '2016-03-30 02:21:28'
date: 2016-03-30 02:21:06
---

> 我保证，不再在网上又发布一篇Monad教程(By Erik Meijer)

自我娱乐，附赠各种Functional with OO，参见[GitHub][1]

```javascript
//Define
Monad = function() {
    this.value = arguments[0];
};
Monad.prototype.unit = function (value) {
    this.value = value;
    return this;
}
Monad.prototype.bind = function (func) {
    var value = func(this.value);
    var monad = new Monad(value);
    return monad;
}
Monad.prototype.extract = function () {
    return this.value;
}

//Use
var monad = new Monad;
var monad = new Monad(10);

monad.unit(20);
monad.unit(new Monad(30));

monad.bind(function (value) { return value });
var result = monad.bind(function (value) {
    return value.extract() * 2;
})

console.log(result.extract());//60

//Test
var monad = new Monad;
monad.unit(10);

console.log(monad.extract());//10

var firstMonad = monad.bind(function (value) {
    return value / 2;
});

console.log(firstMonad.extract());//5

var secondMonad = monad.bind(function (value) {
    return value + 1;
}).bind(function (value) {
    return value + 2;
}).bind(function (value) {
    return value + 3;
});

console.log(monad.extract())//10
console.log(secondMonad.extract());//16
```

 [1]: https://github.com/lizhuoli1126/Functional-OO