---
layout: post
title: "Promise 总结"
date: 2016-02-28 15:24:30 +0800
comments: true
categories: 
---

##什么是 Promise
简单来说， Promise 是把类似的异步处理对象和处理规则进行规范化，并按照统一的接口来编写，这样我们就能够避免写很多的 callback ( 相信有不少人已经对 callback 有恐惧心理了 )

比如正常 callback 写法：

```ruby
getAsync("fileA.txt", function(error, result){
    if(error){// 取得失败时的处理
        throw error;
    }
    // 取得成功时的处理
});
```

Promise 写法：
```ruby
var promise = getAsyncPromise("fileA.txt"); 
promise.then(function(result){
    // 获取文件内容成功时的处理
}).catch(function(error){
    // 获取文件内容失败时的处理
});
```

相比之下，Promise 更加清晰

## 如何支持 Promise

![](http://7jpo3s.com1.z0.glb.clouddn.com/promise-1.png)

目前来说，上图的相应浏览器已经原生支持 `ES6 Promise` 标准了，就是说你可以不用加任何库就可以在浏览器使用`Promise`

那么在不支持`Promise`的浏览器上该怎么办呢？我们则需要引入`Promise`的实现库 `Polyfill`

####jakearchibald/es6-promise
一个兼容 ES6 Promises 的Polyfill类库。 它基于 RSVP.js 这个兼容 Promises/A+ 的类库， 它只是 RSVP.js 的一个子集，只实现了Promises 规定的 API。
####yahoo/ypromise
这是一个独立版本的 YUI 的 Promise Polyfill，具有和 ES6 Promises 的兼容性。 本书的示例代码也都是基于这个 ypromise 的 Polyfill 来在线运行的。

####getify/native-promise-only
以作为ES6 Promises的polyfill为目的的类库 它严格按照ES6 Promises的规范设计，没有添加在规范中没有定义的功能。 如果运行环境有原生的Promise支持的话，则优先使用原生的Promise支持。

如果是使用了`Babel`来编译`ES6`的话，则需要使用`babel-polyfill`

##基本用法

```ruby
function asyncFunction() {    
    return new Promise(function (resolve, reject) {
        var num = Math.random() * 100;
        if (num > 50) {
            // 成功事件
            resolve('yo');
        } else {
            // 失败事件
            reject('sad');
        }
    });
}

// 第一种写法
asyncFunction().then(function (value) {
    console.log(value);    // => 'yo'
}).catch(function (error) {
    console.log(error);    // => 'sad'
});

// 第二种写法
asyncFunction().then(function (value) {
    console.log(value);    // => 'yo'
}, function (error) { // 注意这里是个逗号
    console.log(error);    // => 'sad'
});
```
```ruby
# then() 方法可以同时处理成功状态以及失败状态
promise.then(function onResolve(){}, function onReject(){})

# 不过最好还是使用下面这种方法
# 因为这样在 then() 中发生的错误也可以在 catch() 内捕捉到
promise.then(
    function onResolve(){}
).catch(
    function onReject(){}
)

# promise.catch() 相当于
promise.then(undefined, onReject(){
    ...
});
```

### Promise.resolve
一般来说我们会用 `new Promise()` 来创建 promise, 但也可以用其他方法
```ruby
Promise.resolve(42)
```
可以认为是下面的语法糖
```ruby
# 直接将状态变为成功状态
new Promise(function(resolve){
    resolve(42);
})
```
`Promise.resolve(42)` 返回的也是一个 promise 对象，因此我们可以直接连用`then`方法
```ruby
Promise.resolve(42).then(function (value){
    console.log(value)
})
```
### Promise.reject
相应的，`Promise.reject`也是一个快捷方式
```ruby
Promise.reject(42)
```
相当于
```ruby
new Promise(function(resolve, reject){
    reject(42);
})
```
那么我们可以直接:
```ruby
Promise.reject(new Error("BOOM!")).catch(function(error){
    console.error(error);
});
```

###异步调用
在使用`Promise.resolve()`时，promise对象立马进入了 resolve 状态，那`then()`的方法是不是立马同步调用呢?

```ruby
var promise = new Promise(function (resolve){
    console.log("inner promise"); // 1
    resolve(42);
});
promise.then(function(value){
    console.log(value); // 3
});
console.log("outer promise"); // 2
```
也许你会以为输出结果是：
```ruby
inner promise // 1
42            // 2
outer promise // 3
```
但其实是:
```ruby
inner promise // 1
outer promise // 2
42            // 3
```
说明了`then()`方法并不是同步调用的，而是异步调用。

因为同步与异步一起使用的话，会造成混乱，因为 Promise 规范也指明了：`Promise 只能够使用异步调用方式`

> 绝对不能对异步回调函数（即使在数据已经就绪）进行同步调用。
>
> 如果对异步回调函数进行同步调用的话，处理顺序可能会与预期不符，可能带来意料之外的后果。

> 对异步回调函数进行同步调用，还可能导致栈溢出或异常处理错乱等问题。

> 如果想在将来某时刻调用异步回调函数的话，可以使用 setTimeout 等异步API。

> Effective JavaScript
> — David Herman

##方法链
想必使用过 JQuery 的人都已经使用过`方法链`了。
```ruby
promise.then(function taskA(value){
    return 'hello'
}).then(function taskB(value){
    console.log(value)   // => 'hello'
}).catch(function onRejected(error){
    console.log(error);
});
```

`then()`和`catch()`都会返回一个新的`promise对象`，这也是为什么可以使用`方法链`的原因。

前一个`then()`的回调函数中，返回的值不同也会有不同的情况:

1. 回调函数返回的是`非 promise 对象`,那么该返回值将会传给第二个`then()`的`resolved`回调函数中（即 promise 默认为 resolved 状态)
2. 回调函数返回的是`promise 对象 A`,那么`then()`生成的`新 promise B`将会把`A`的状态以及返回值占为己有。
3. 没有返回值，`then()`返回的默认为 resolved 状态的 promise


## Reject or Throw?
`Promise`的构造函数，以及被 `then` 调用执行的函数基本上都可以认为是在 `try...catch` 代码块中执行的，所以在这些代码中即使使用 `throw` ，程序本身也不会因为异常而终止。
```ruby
var promise = new Promise(function(resolve, reject){
    throw new Error("message");
});
promise.catch(function(error){
    console.error(error);// => "message"
});
```
但是一般来说我们使用`Reject`会更符合我们想把`promise`置为`reject`的意思。
```ruby
var promise = new Promise(function(resolve, reject){
    reject(new Error("message"));
});
promise.catch(function(error){
    console.error(error);// => "message"
})
```

那么在`then`中如何使用`reject`呢？我们可以在回调函数中返回`rejected`状态的`promise`对象，那么此时promise就会变成`rejected`状态了。
```ruby
var onRejected = console.error.bind(console);
var promise = Promise.resolve();
promise.then(function () {
    return Promise.reject(new Error("this promise is rejected"));
}).catch(onRejected);

```
## Then Or Catch ?

前面我们说过，`catch(function(){})`其实就是等于`then(undefined, function(){})`。

那为何我们建议用`catch()`来进行捕获`rejected`状态的 promise 呢？

```ruby
function throwError(value) {
    // 抛出异常
    throw new Error(value);
}
// <1> onRejected不会被调用
function badMain(onRejected) {
    return Promise.resolve(42).then(throwError, onRejected);
}
// <2> 有异常发生时onRejected会被调用
function goodMain(onRejected) {
    return Promise.resolve(42).then(throwError).catch(onRejected);
}
// 运行示例
badMain(function(){
    console.log("BAD");
});
goodMain(function(){
    console.log("GOOD");
});
```
就是说，当`then(resolve(){}, reject(){})` 内的`resolve(){}`方法发生错误的时候，此时`reject(){}`函数并不能够去捕捉到。而使用`catch()`则可以。


## Promise.all
`Promise.all` 接收一个 promise对象的数组作为参数，当这个数组里的所有promise对象全部变为resolve或reject状态的时候，它才会去调用 .then 方法。

```ruby
first_promise = new Promise(function(resolve, reject){
    ...
})
second_promise = new Promise(function(resolve, reject){
    ...
})
Promise.all([first_promise, second_promise]).then(function(results)){
    console.log(results)  // 按照数组顺序返回的值
}
```
需要注意的是，数组内的 promise 是同时执行的。

## Promise.race
`Promise.all` 在接收到的所有的对象promise都变为 `FulFilled` 或者 `Rejected` 状态之后才会继续进行后面的处理， 与之相对的是 `Promise.race` 只要有一个`promise`对象进入 `FulFilled` 或者 `Rejected` 状态的话，就会继续进行后面的处理。
```ruby
// `delay`毫秒后执行resolve
function timerPromisefy(delay) {
    return new Promise(function (resolve) {
        setTimeout(function () {
            resolve(delay);
        }, delay);
    });
}
// 任何一个promise变为resolve或reject 的话就执行then()方法
Promise.race([
    timerPromisefy(1),
    timerPromisefy(32),
    timerPromisefy(64),
    timerPromisefy(128)
]).then(function (value) {
    console.log(value);    // => 1
});
```

## 关于 IE8
现在的浏览器基本都是基于 ECMAScript 5 的，但是 IE8 以及以下的浏览器都是基于 ECMAScript 3 的，而 `catch` 在 `ECMAScript 3`是保留字，所以使用`.catch()`会报错

解决方法：
```ruby
var promise = Promise.reject(new Error("message"));
promise["catch"](function (error) {
    console.error(error);
});
```