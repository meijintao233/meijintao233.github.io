
---
title: lodash源码学习（一）
date: 2018-10-01 11:58:26
tags:
---

趁着有时间研究了一下lodash库的一些函数的实现，发现里面对于一些传入参数的判定比较完备，而且也对一些原有的js函数进行了一些扩展。 
<!-- more -->
**after**

```javascript
/**
 *
 * @param {number} 
 * @param {Function} 
 * @returns {Function} 
 * 
 *
 * const saves = ['profile', 'settings']
 * const done = after(saves.length, () => console.log('done saving!'))
 *
 * forEach(saves, type => asyncSave({ 'type': type, 'complete': done }))
 * // => Logs 'done saving!' after the two async saves have completed.
 */
function after(n, func) {
  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  return function(...args) {
    //n小于1的时候都会马上执行
    if (--n < 1) {
      return func.apply(this, args)
    }
  }
}

export default after
```

after的功能是待函数的触发了n次后才真正执行。after接受两个参数，一个是执行次数n，还有一个是待执行的函数，如果需要传入参数可以使用bind。 

---

**before**

```javascript

```

before的功能是限制函数的执行次数为n-1，如果n小于0则不执行。before接受两个参数，一个是执行次数n，还有一个是待执行的函数，如果需要传入参数可以使用bind。

---

**chunk**

```javascript

```

首先引入了slice，其目的是利用扩展了的slice实现深复制，chunk接受两个参数，分别是待分组的数组array，以及每一分组的元素数目size，如果不够分则剩下的归为一组。 

---

**defer**

```javascript
/**
 *
 * @param {Function} 
 * @param {...*} [args] 
 * @returns {number} 
 * 
 * defer(text => console.log(text), 'deferred')
 * // => Logs 'deferred' after one millisecond.
 */
function defer(func, ...args) {
  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  //延迟1ms执行
  return setTimeout(func, 1, ...args)
}

export default defer
```

defer函数的功能是延迟执行，接受两个参数，分别是需要延迟执行的函数func，还有需要传给func的参数。

这里利用了settimeout将函数放入宏任务队列中实现了延迟，会在js主线程和微任务队列执行结束后执行。

---

**delay**

```javascript
/**
 * @param {Function} 
 * @param {number} 
 * @param {...*} 
 * @returns {number} 
 *
 * delay(text => console.log(text), 1000, 'later')
 * // => Logs 'later' after one second.
 *
 */
function delay(func, wait, ...args) {
  //类型判断，必须是函数
  if (typeof func != 'function') {
    throw new TypeError('Expected a function')
  }
  //返回一个定时器,保证wait为Number类型
  return setTimeout(func, +wait || 0, ...args)
}

export default delay
```

delay接受至少三个参数，分别是需要延迟执行的函数func，延迟执行的时间wait，以及需要向func传入的参数，返回的是定时器的id，可以根据该id对定时器进行清除，delay实现了延迟执行的功能。 



像这里这种在settimeout给func传参写法在IE9以下是不兼容的，需要polyfill 

```javascript
(function(){
    setTimeout(function(arg){
        if(arg==='test'){
            return;
        }
        const __nativeST__ = window.setTimeout;
        window.setTimeout = function(callback,delay){
            const args = Array.prototype.slice.call(arguments, 2);
            return __nativeST__(function(callback instanceof Function?function(){
                callback.apply(null, args);
            }):callback,delay)
        }
    },0,'test')
}())
```

---

**forEach && each**

```javascript
import arrayEach from './.internal/arrayEach.js' //处理数组的遍历
import baseEach from './.internal/baseEach.js'   //处理类数组的遍历

/**
 * 
 * @param {Array|Object} 
 * @param {Function} 
 * @returns {Array|Object} 
 *
 * forEach([1, 2], value => console.log(value))
 * // => Logs `1` then `2`.
 *
 * forEach({ 'a': 1, 'b': 2 }, (value, key) => console.log(key))
 * // => Logs 'a' then 'b' (iteration order is not guaranteed).
 */
function forEach(collection, iteratee) {
  //判断是数组还是类数组，分别进行不同的处理
  const func = Array.isArray(collection) ? arrayEach : baseEach
  return func(collection, iteratee)
}

export default forEach
```

forEach的功能是遍历数组和类数组，扩展了原生forEach的功能，主要是通过循环遍历之后将新变量注入到遍历器触发的函数中，实现了该功能，forEach还有一个alias为each 

---

**isArrayLike**

```javascript
import isLength from './isLength.js'

/**
 *
 * @param {*} 
 * @returns {boolean}
 * 
 *
 * isArrayLike([1, 2, 3])
 * // => true
 *
 * isArrayLike(document.body.children)
 * // => true
 *
 * isArrayLike('abc')
 * // => true
 *
 * isArrayLike(Function)
 * // => false
 */
function isArrayLike(value) {
  return value != null && typeof value != 'function' && isLength(value.length)
}

export default isArrayLike
```

isArrayLike功能是判断是否是类数组对象，即value是否是有效的length，isLength函数实现了该功能，isArrayLike接受一个参数value，由于历史遗留问题，null也被视为一个对象，因此需要排除，同时Function类型也有length属性，但是代表的是参数的个数，不是类数组，因此也需要排除。 

---

**isLength**

```javascript
//最大安全整数
const MAX_SAFE_INTEGER = 9007199254740991

/**
 *
 * @param {*} 
 * @returns {boolean} 
 * 
 *
 * isLength(3)
 * // => true
 *
 * isLength(Number.MIN_VALUE)
 * // => false
 *
 * isLength(Infinity)
 * // => false
 *
 * isLength('3')
 * // => false
 */
function isLength(value) {
  return typeof value == 'number' &&
    value > -1 && value % 1 == 0 && value <= MAX_SAFE_INTEGER
}

export default isLength
```

isLenth的功能是判断该对象是否有一个有效的length属性，其接受一个参数value，value必须是Number类型且必须为非负整数，同时不能超过最大整数 

---

**last**

```javascript
/**
 *
 * @param {Array} 
 * @returns {*}
 * 
 * last([1, 2, 3])
 * // => 3
 */
function last(array) {
  //判断是否数组，如果是则获取数组长度，否则设置为0
  const length = array == null ? 0 : array.length
  //如果length大于1，则返回数组最后一个元素，否则返回undefined
  return length ? array[length - 1] : undefined
}

export default last
```

last函数的功能是获取数组最后一个元素，last接受一个参数，期望该参数是数组，并对其不是数组时进行了异常处理 

---

**slice**

```javascript
/**
 * @param {Array} 
 * @param {number} 
 * @param {number} 
 * @returns {Array} 
 *
 * const array = [1, 2, 3, 4]
 * _.slice(array, 2) //[3, 4]
 */
function slice(array, start, end) {
  //数组长度，如果是空数组,则为0
  let length = array == null ? 0 : array.length
  //如果是空数组，则返回新数组
  if (!length) {
    return []
  }
  //处理start，默认为0
  //由于undefined==null为true，因此缺省时默认深复制整个数组
  start = start == null ? 0 : start
  //处理end，缺省时默认为length
  end = end === undefined ? length : end
  
  if (start < 0) {
    //当start小于0时，判断边界，如果超出数组长度则置0
    //否则就是length减去start的绝对值
    start = -start > length ? 0 : (length + start)
  }
  //同理，当end大于数组长度时，则设置为数组长度
  end = end > length ? length : end
  if (end < 0) {
    //end小于0则length减去end的绝对值
    end += length
  }
  //当start大于end的时候，则返回空数组
  //否则算出复制的范围，不包括end
  //>>>是无符号右移，目的是取整
  //这里的判断包含了start超过length的情况
  length = start > end ? 0 : ((end - start) >>> 0)
  start >>>= 0
  
  //这个是flag值
  let index = -1
  //创建一个新数组
  const result = new Array(length)
  //进行复制
  while (++index < length) {
    result[index] = array[index + start]
  }
  //返回新的数组
  return result
}

export default slice
```

根据提供的start、end参数，深复制一个数组的一部分。扩展了原生数组的slice的功能，start和end可以接受负数，表示到end的距离，start默认为0，end默认为传入数组的长度。利用>>>0右移实现取整，对start和end的可能情况进行处理，分别是start和end缺省时，start和end超出边界值时，start大于end时，start和end为非整数时 

---

**once**

```javascript
import before from './before.js'

/**
 *
 * @param {Function} 
 * @returns {Function} 
 * 
 * const initialize = once(createApplication)
 * initialize()
 * initialize()
 * // => `createApplication` is invoked once
 */
function once(func) {
  return before(2, func)
}

export default once
```

once函数的功能是只会执行一次函数，通过before函数控制，once接受一个参数func 

