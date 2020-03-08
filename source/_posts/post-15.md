---
title: Generator 生成器
date: 2020-03-08 15:52:55
tags:
---

## 一、什么是 Generator

> The **Generator** object is returned by a [generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) and it conforms to both the [iterable protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterable_protocol) and the [iterator protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterator_protocol).
>
> **生成器**对象是由一个 [generator function](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*) 返回的,并且它符合[可迭代协议](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Iteration_protocols#iterable)和[迭代器协议](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Iteration_protocols#iterator)。

### 可迭代协议

**可迭代协议**是指允许 JavaScript 对象（可迭代对象）去定义或定制它们的迭代行为，这个对象（或其原型链中的任意一个对象）必须具有一个带 `Symbol.iterator`键的属性，其值返回一个对象的无参函数，被返回对象符合**迭代器协议**

### 迭代器协议

**迭代器协议**定义了一种标准的方式来产生一个有限或无限序列的值，并且当所有的值都已经被迭代后，就会有一个默认的返回值

**迭代器**需要实现 **next() **方法，返回一个对象的无参函数，被返回对象拥有两个属性，分别是 (done :Boolean) 迭代器已经超过了可迭代次数时为 true，这时 value 的值可以被省略，除非该迭代器 return 了某个值

<!-- more -->

```javascript
var iterable = {
  *[Symbol.iterator](n) {
    yield 1;
    yield (n || 0) + 2;
    yield 3;
  }
};

for (let value of iterable) {
  console.log(value); // 1 2 3
}

console.log(...iterable[Symbol.iterator](4)); // 1 6 3
console.log(iterable[Symbol.iterator](4).next());
/*
Object {
  done: false,
  value: 1
}
*/
```

### 生成器函数

生成器函数的定义和普通函数类似，只是需要在函数名前关键字 **function **后添加 **\*** 号 **funtion\* generator(){}**，需要注意的是，生成器函数**不能**作为构造函数

调用生成器函数后会返回一个生成器对象，该对象是个可迭代对象，同时实现了 **next** 方法，所以也是一个迭代器，即生成器既是迭代器也是可迭代对象

**yield** 表达式就是用来表示迭代暂停的位置

```javascript
function* demo(d) {
  yield 10;
  x = yield d;
  yield x;
}

const gen = demo(666);
gen.next(); // 10
gen.next(); // 666
gen.next(); // undefined
gen.next(); // undefined
/*
Object {
  done: false,
  value: 10
}
*/
/*
Object {
  done: false,
  value: 666
}
*/
/*
Object {
  done: false,
  value: undefined
}
*/
/*
Object {
  done: true,
  value: undefined
}
*/

// 以下省略next（）返回对象，只表 value 值
const gen = demo(666);
gen.next(); // 10
gen.next(55); // 666
gen.next(); // 55
gen.next(); // undefined

function* demo1(d) {
  try {
    yield 10;
  } catch (e) {
    console.log(e); // 55
    x = yield d;
    yield x;
  }
}
const gen = demo1(666);
gen.next(); // 10 false
gen.return(55); // 55 true
gen.next(); // undefined true
gen.next(); // undefined true

const gen = demo1(666);
gen.next(); // 10 false
gen.throw(55); // 666 true
gen.next(); // undefined true
gen.next(); // undefined true
```

从上面的例子我们能看出来，执行 **next**() 的时候，代码的运行会到 yield 关键字的右侧，这时返回的值就是 **yield** 右侧的语句执行的结果，而 **next()** 中的传参则会被赋值到 **yield** 左侧，即作为 **yield** 的返回值使用。

如果生成器对象调用 **throw** 方法，那么能够被 tryCatch 捕获到，并且在 catch 的 body 中可以继续执行

如果生成器对象调用 **return** 方法，那么函数立即返回，同时后续的 **yield**都不会被执行，等同于函数在该语句的位置被 return 了

### yield 和 yield\*

**yield\*** 表达式用于委托给另一个可迭代对象

```javascript
function* g1() {
  yield 2;
  yield 3;
  yield 4;
}

function* g2() {
  yield 1;

  // 嵌套生成器
  yield* g1();
  yield g1();

  yield* [5, yield "hi4", 6, yield* "hi"]; // [5,undefined,6,undefined] 第一个是因为next()没有参数导致的，第二个是因为 yield* 没有 return 结果
  yield [7, 8];

  yield 9;
}

var iterator = g2();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: 4, done: false }
console.log(iterator.next()); // { value: [object Generator] {}, done: false }
console.log(iterator.next()); // { value: 'hi4', done: false }
console.log(iterator.next()); // { value: 'h', done: false }
console.log(iterator.next()); // { value: 'i', done: false }
console.log(iterator.next()); // { value: '5', done: false }
console.log(iterator.next()); // { value: undefined, done: false }
console.log(iterator.next()); // { value: '6', done: false }
console.log(iterator.next()); // { value: undefined, done: false }
console.log(iterator.next()); // { value: [7,8], done: false }
console.log(iterator.next()); // { value: 9, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

从上面的例子可以看出，**yield\*** 的作用实质上是解决嵌套可迭代对象（arguments，array，string，set，map 等都是可迭代的）的问题，更像是个语法糖，同时其 **yield** 还有一个区别是它是表达式，所以会有自己的返回值，而 **yield **的返回值由 **next()** 参数所决定。

## 二、意义（解决什么问题）

### 协程与化异步为同步写法

与协程相关的是进程、线程，进程是计算机资源分配的基本单位，线程是执行的基本单位。

计算机将存储资源分配给进程之后，进程就可以利用这些资源进行计算，进程可以生成线程去执行语句，线程间可以使用共享内存进行通讯，而协程则是更精细化的资源管理。为了防止某一进程占用过多资源，Cpu 有一套调度算法，随着粒度越细，切换的时候的消耗就越小，协程则可以理解成是在某个线程中的多线程而这个切换过程是无需系统参与的，也就少了系统调用，同时也是在同一个栈上切换的，例如 Go 的 coroutine 就是占用更少的资源，进行更精细的调度。

generator 的暂停执行特性，可以将函数执行的控制权交出，JS 的单线程特性使得可以利用 generator 实现协程，其中的关键在于实现流程自动调度功能。

JS 是运行在单线程中的，为了提高效率，引入了异步方式，当运行一些远程 IO 或者网络请求的时候可以无需等待直接往下执行，等拿到响应之后再执行回调。这就引入了一个典型的问题，就是回调嵌套地狱(callback hell)。

当引入了 **generator** 后，异步的代码就可以写成同步的方式，降低认知成本，提高代码可读性

```javascript
doAsync1(str, res1 => {
  doAsync2(res1, res2 => {
    doAsync3(res2, res3 => {});
  });
});

function* doAsync(str) {
  const res1 = yield doAsync1(str);
  const res2 = yield doAsync2(res1);
  const res3 = yield doAsync3(res2);
  console.log(res3);
}

const gen = doAsync(str);
const async1 = gen.next().value;
const async2 = gen.next(async1).value;
const async3 = gen.next(async2).value;
```

## 三、实践和库（ co 和 redux-saga）

co 和 redux-saga 的核心代码是类似的，都实现了一套流程自动调度，redux-saga 在此基础上添加了更多的功能

```javascript
const it = gen();
function next(res) {
  if (!it.done) {
    const ret = it.next(res);
    next(ret.value);
  }
}

next();
```

### co

**co **库主要做的一件事情就是为 generator 包裹一层 promise，然后通过 then 将控制权返回，同时对于 generator 实现了流程的自动调度，对于对象和数组类型还支持并发调用。

```javascript
// 将对象的value值转化为promise数组 在用Promise.all实现并发
function objectToPromise(obj) {
  var results = new obj.constructor();
  var keys = Object.keys(obj);
  var promises = [];
  for (var i = 0; i < keys.length; i++) {
    var key = keys[i];
    // 将非promise类型转为promise
    var promise = toPromise.call(this, obj[key]);
    if (promise && isPromise(promise)) defer(promise, key);
    else results[key] = obj[key];
  }
  // 并发调用
  return Promise.all(promises).then(function() {
    return results;
  });

  function defer(promise, key) {
    results[key] = undefined;
    // 返回值以对象形式返回
    promises.push(
      promise.then(function(res) {
        results[key] = res;
      })
    );
  }
}
// 将数组中的非promise类型元素转为promise
function arrayToPromise(obj) {
  return Promise.all(obj.map(toPromise, this));
}

// 主体部分非常简洁
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);

  return new Promise(function(resolve, reject) {
    if (typeof gen === "function") gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== "function") return resolve(gen);

    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        // 执行相邻 yield 之间语句, res 为上一个 yield 左侧值，即yield右侧语句执行后返回值
        ret = gen.next(res);
      } catch (e) {
        // 捕获异常抛出错误
        return reject(e);
      }
      //递归调用，将 yield 右侧的值赋值给 yield 的左侧
      next(ret);
      return null;
    }

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function next(ret) {
      // 递归结束后 将return值 返回
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value))
        // 递归调用promise
        return value.then(onFulfilled, onRejected);
      return onRejected(
        new TypeError(
          "You may only yield a function, promise, generator, array, or object, " +
            'but the following object was passed: "' +
            String(ret.value) +
            '"'
        )
      );
    }
  });
}
```

### redux-saga

```javascript
import createSagaMiddleware from "redux-saga";
import rootSaga from "./sagas";
const sagaMiddleware = createSagaMiddleware();
sagaMiddleware.run(rootSaga);
```

```javascript
// redux-saga/packages/core/src/internal/middleware.js
// createSagaMiddleware 执行的就是这个函数
export default function sagaMiddlewareFactory({
  context = {},
  channel = stdChannel(),
  sagaMonitor,
  ...options
} = {}) {
  let boundRunSaga;

  function sagaMiddleware({ getState, dispatch }) {
    boundRunSaga = runSaga.bind(null, {
      ...options,
      context,
      channel,
      dispatch,
      getState,
      sagaMonitor
    });

    return next => action => {
      if (sagaMonitor && sagaMonitor.actionDispatched) {
        sagaMonitor.actionDispatched(action);
      }
      const result = next(action);
      channel.put(action);
      return result;
    };
  }

  // 这里会传入rootSaga，即项目中的所有的saga effect，也就是 generator
  sagaMiddleware.run = (...args) => {
    return boundRunSaga(...args);
  };

  //...
  return sagaMiddleware;
}
```

```javascript
//redux-saga/packages/core/src/internal/runSaga.js
export function runSaga(
  {
    channel = stdChannel(),
    dispatch,
    getState,
    context = {},
    sagaMonitor,
    effectMiddlewares,
    onError = logError
  },
  saga,
  ...args
) {
  // 这里的 saga 即是上面的 rootSaga，iterator 为生成器对象
  const iterator = saga(...args);

  const effectId = nextSagaId();

  //...

  return immediately(() => {
    const task = proc(
      env,
      iterator,
      context,
      effectId,
      getMetaInfo(saga),
      /* isRoot */ true,
      undefined
    );

    if (sagaMonitor) {
      sagaMonitor.effectResolved(effectId, task);
    }

    return task;
  });
}
```

```javascript
//redux-saga/packages/core/src/internal/proc.js
export default function proc(env, iterator, parentContext, parentEffectId, meta, isRoot, cont) {


  //...

  next.cancel = noop


  const mainTask = { meta, cancel: cancelMain, status: RUNNING }


  const task = newTask(env, mainTask, parentContext, parentEffectId, meta, isRoot, cont)

  const executingContext = {
    task,
    digestEffect,
  }


  function cancelMain() {
    if (mainTask.status === RUNNING) {
      mainTask.status = CANCELLED
      next(TASK_CANCEL)
    }
  }


  if (cont) {
    cont.cancel = task.cancel
  }


  next()


  return task
  // 递归 iterator 生成器对象
  function next(arg, isErr) {
    try {
      let result
      if (isErr) {
        // 抛错误
        result = iterator.throw(arg)

        sagaError.clear()
      } else if (shouldCancel(arg)) {

        mainTask.status = CANCELLED
        // 取消并返回迭代器对象
        next.cancel()

        result = is.func(iterator.return) ? iterator.return(TASK_CANCEL) : { done: true, value: TASK_CANCEL }
      } else if (shouldTerminate(arg)) {
       // 中断并返回迭代器对象
        result = is.func(iterator.return) ? iterator.return() : { done: true }
      } else {
        result = iterator.next(arg)
      }
			// 如果迭代未结束则继续递归，并将返回值注入
      if (!result.done) {
        digestEffect(result.value, parentEffectId, next)
      }
      // ...
    } catch (error) {
      if (mainTask.status === CANCELLED) {
        throw error
      }
      mainTask.status = ABORTED

      mainTask.cont(error, true)
    }
  }

// ...

  // 递归调用 next，cb 即为传入的 next函数，effect 为返回值
  function digestEffect(effect, parentEffectId, cb, label = '') {
    const effectId = nextEffectId()

		//...

    let effectSettled


    function currCb(res, isErr) {
      if (effectSettled) {
        return
      }

      effectSettled = true
      cb.cancel = noop
      //...

      // 继续递归
      cb(res, isErr)
    }

	   //...



}
```
