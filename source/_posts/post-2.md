---
title: 异步回调的解决方案
date: 2018-08-06 12:58:26
tags:
---
#### 发现问题
在实践过程中经常会遇到a，b，c，d...等函数，后者依赖于前者请求之后的结果，或者是d依赖于a，b，c...等函数这些情形，在第一种情况中会出现回调嵌套的情形
```javascript
a(){
	b(){
		c(){
			...
		}
	}
}
```
而第二种情况中，为了能更快获取到a，b，c...等的回调内容，会考虑进行并发请求，而这些又该如何解决呢？请继续往下看。
<!-- more -->
--- 
#### 解决方案
针对第一种情况，我们可以把异步回调以同步的写法表示出来
```javascript
function async(){}
//传入两个参数，一个数组，一个函数
//数组用来存放需要顺序执行的回调，
//函数执行有两种情况，一种是顺序执行完数组中的元素后执行，另一种往下看
async.prototype.waterfall = function(arr,cb) {
  let index = 0;//标记位，标记进行到第几个回调
  let len = arr.length;//记录数组长度
  let callback = cb || function(){}//最后执行的回调，如果没有就执行一个空函数
  //done是每个函数的最后一个参数，用来标记回调已经结束，执行下一个回调
  let done = function(){
    //将类数组arguments转化为数组，
    //done函数第一个参数控制是否直接去执行cb，如果为真则直接去执行，否则继续顺序执行
    //done还能传入需要传给下一个回调的参数
    var args = Array.prototype.slice.call(arguments);
    //如果第一个参数为真或者已经把所有函数都执行了一次，则执行最终的回调
    if(args[0] || index == len) {
      callback.apply(null，args.slice(1));
      return;
    }
    //如果还没执行完则继续执行
    if(index<len){
      args.push(done);//注入函数参数done
      arr[index++].apply(null,args.slice(1));
    }
  }
  //启动
  done(null);
}
```
主要思路就是通过一个done函数对顺序执行进行控制，然后还需要设置一个观测值记录已经执行了多少个回调。
```javascript
let b = new async();
let c = function(done){
   setTimeout(function(){
     console.log(111)
     done(null,"111"); 
   },300)
 };
 let d = function(p1,done){
   setTimeout(function(){
     console.log(222);
     done(null)
   },200);
 }
 b.waterfall([c,d],function(){
   console.log(3333)
 })
```
最终会顺序输出111，222，333；

---- 
针对第二种情况，在阿里文学读书签约活动中遇到需要向芝麻信用和支付宝分别发起请求获取状态，对于在依赖限制情况下进行并发请求，主要思路是可以设置一个变量计数器，等到需要的依赖都返回之后执行下一个请求。
```javascript
//假设d依赖于a,b
let rec = 0;
let globalData = {};
a(){
	......
	//成功获取到数据后存在全局变量globalData中
	globalData[aData] = {...};
	++ret === 2 && d()
}
b(){
	......
	//成功获取到数据后存在全局变量globalData中
	globalData[bData] = {...};
	++ret === 2 && d()
}
```
以上的这些解决方法其实都可以用Promise进行实现。

----- 
#### Promise是什么
Promise是在**ES6**中出现的一个异步解决方法，它是一个状态机，初态是**Pending(待解决)**，可以通过**resolve和reject**方法将其状态变为**fulfiiling(成功)**和**rejected(失败)**，状态一旦变化之后，那么就不会再发生改变，而且其值能一直保存下来，之后还能再获取到原来的值，而不像事件监听，如果发生的时候没监听到，那么之后再去监听也不能获取到原来的值，它还可以进行链式调用**a.then().then()...**，可以用来解决回调地狱的问题。

--- 

#### Promise的用法
- 区分**Promise.all和Promise.race**，它们都是需要传入一个数组作为参数，数组中的元素都是**promise**对象，前者需要等到数组中所有**promise**对象的状态都确定之后才会继续往下执行，而后者则是哪个**promise**对象的状态先确定则将其返回的值作为参数继续往下执行。
- **resolve和reject**，在new Promise实例的时候，需要传入一个函数**fn**作为参数，而函数中还有两个参数，就是**resolve和reject(当然也可以用其他命名)**
```javascript
//fn就是创建实例时传入的参数，promise则是指向Promise实例的上下文
function doResolve(fn, promise) {
  //记录下是否已经执行
  var done = false;
  //tryCallTwo就是将两个函数参数注入到fn中并执行fn
  var res = tryCallTwo(fn, function (value) {
    //这里就是只有第一个resolve或者reject能生效的原因，调用过一次resolve或者reject后done变为true
    if (done) return;
    done = true;
    //调用Promise内部定义的resolve函数，并将值作为参数传到then方法中
    resolve(promise, value);
  }, function (reason) {
    if (done) return;
    done = true;
    reject(promise, reason);
  });
  if (!done && res === IS_ERROR) {
    done = true;
    reject(promise, LAST_ERROR);
  }
}
function resolve(self, newValue) {
  ......省略一部分
  //如果传入的是基本类型，则直接保存在_value，
  //如果传入的值是对象或者函数，则进行处理后再保存在_value，通过doResolve循环调用实现
  if (
    newValue &&
    (typeof newValue === 'object' || typeof newValue === 'function')
  ) {
	//获取newValue对象的then方法，也就是说可以传入一个含then属性的对象
    var then = getThen(newValue);
    ......省略
    if (
      then === self.then &&
      newValue instanceof Promise
    ) {
      self._state = 3;
      self._value = newValue;
      finale(self);
      return;
    } else if (typeof then === 'function') {//如果then是函数则执行resolve
      doResolve(then.bind(newValue), self);
      return;
    }
  }
  
  self._state = 1;
  self._value = newValue;
  finale(self);
```
从上面可以看出，在new Promise实例的时候，需要传入一个函数**fn**或者是含**then**属性的对象作为参数，然后该参数会通过**doResolve**方法注入两个函数参数(实现**resolve和reject**的功能)，**resolve**中还能传入参数，分别是基本类型，**function**，**另一个promise**，最后将结果保存在**_value**中，接着我们可以通过then方法进行链式调用，而**_value**中的的值则传递给then作为then的参数，主要是通过**handleResolve**方法实现的，而且在调用then会返回一个新的promise实例，从而保证了promise的状态只能变更一次的效果
```javascript
Promise.prototype.then = function(onFulfilled, onRejected) {
  ......省略
  //生成新的实例，并最终返回
  var res = new Promise(noop);
  //Handler的作用是把then中传入的函数参数绑定到res的执行环境中
  handle(this, new Handler(onFulfilled, onRejected, res));
  return res;
};

function handle(self, deferred) {
  ......省略
  handleResolved(self, deferred);
}

function handleResolved(self, deferred) {
  asap(function() {
    //根据promise不同的状态执行不同的函数，1是成功，2是失败
    var cb = self._state === 1 ? deferred.onFulfilled : deferred.onRejected;
    //如果cb是null，那么就会直接将_value中的值传递下去
    if (cb === null) {
      if (self._state === 1) {
        resolve(deferred.promise, self._value);
      } else {
        reject(deferred.promise, self._value);
      }
      return;
    }
    //如果是函数，则将_value的值注入作为参数，并返回函数的结果
    var ret = tryCallOne(cb, self._value);
    if (ret === IS_ERROR) {
      reject(deferred.promise, LAST_ERROR);
    } else {
      //将then函数的返回值存到_value中，实现链式调用
      resolve(deferred.promise, ret);
    }
  });
}
```

#### 总结
经过上面的分析和梳理，应该对promise的运行机制和使用方式也很清晰了
- 在new Promise实例的时候可以传入函数或者含then的对象作为参数
- 在then中如果传入的不是函数，那么上一个_value的值将直接传给下一个then作为参数，所以下面将会输出**hahaha**而不是**lalala**
```javascript
let p = new Promise(function(resolve,reject){
	resolve('hahaha')
})
p.then(Promise.resolve('lalala')).then(function(arg){
	console.log(arg)
})
```
- 为了保证状态只能改变一次，每次链式调用过程中都会创建新的promise实例
```javascript
let a = new Promise(...)
b = a.then(...)
console.log(a === b) //false 
```
- promise中用了很多参数注入的方法，例如resolve和reject，_value的传递等
