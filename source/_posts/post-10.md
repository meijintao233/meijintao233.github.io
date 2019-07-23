---
title: "[译]JavaScript箭头函数中的不规则"
date: 2019-07-21 14:00:41
tags:
---

> 原文地址：[Anomalies in JavaScript arrow functions](https://blog.logrocket.com/anomalies-in-javascript-arrow-functions/)
>
> 原文作者：[Glad Chinda](https://blog.logrocket.com/author/gladchinda/)

## 介绍

就我而言，我认为箭头函数是 Javascript 语言在 ES6 规范中最棒的语法补充之一——我的意见，顺便提一下。我知道它以后几乎每天都要使用它们，我猜测这对于大多数 Javascript 开发者也是这样的。

箭头函数在很多方面也可以像常规的函数一样使用。然而，它们通常用于在需要匿名函数语句的地方——例如，回调函数。

下面的例子展示了一个箭头函数是怎样被用作回调函数的，特别是数组方法如`map()`，`filter()`，`reduce()`，`sort()`等等。

```javascript
const scores = [
  /* ...some scores here... */
];
const maxScore = Math.max(...scores);

// Arrow Function as .map() callback
scores.map(score => +(score / maxScore).toFixed(2));
```

一眼过去，箭头函数可能看起来像可以被用于或者定义在每个常规函数也能这样的地方，但事实并不是这样的。出于非常好的原因，箭头函数并不是恰好表现得像常规函数一样。也许箭头函数可以被认为是不规则的 Javascript 函数。

尽管箭头函数有一个相当简单的语法，这不是本文的重点。本文的目标是揭示箭头函数不同于常规函数的主要行为方式以及怎样将这些知识用于开发者的优势。

请注意：在本文中，我会用常规函数或者常规 Javascript 函数的术语表示传统的 Javascript 函数语句或者使用 function 关键字定义的表达式。

## TL;DR

- 箭头函数没有重复的命名参数，无论是在严格或者非严格模式。
- 箭头函数没有`arguments`绑定，然而，它们可以访问最接近的非箭头父函数的 arguments 对象。命名和剩余参数非常依赖于捕获的传递给箭头函数的参数。
- 箭头函数不能用作构造函数。于是它们不能被`new`关键字调用。因此箭头函数上不存在原型属性。
- 箭头函数里`this`的值在整个函数的生命周期中保持一样而且总是受到最接近的非箭头父函数的`this`的值约束

## 命名函数参数

Javascript 中的函数通常使用命名参数定义。命名参数是用来根据位置将参数映射到函数作用域里的局部变量中

考虑以下的 Javascript 函数：

```javascript
function logParams(first, second, third) {
  console.log(first, second, third);
}

// first => 'Hello'
// second => 'World'
// third => '!!!'
logParams("Hello", "World", "!!!"); // "Hello"  "World"  "!!!"

// first => { o: 3 }
// second => [ 1, 2, 3 ]
// third => undefined
logParams({ o: 3 }, [1, 2, 3]); // {o: 3}  [1, 2, 3]
```

`logParams()`函数定义了三个命名参数:`first`，`second`和`third`。命名参数会基于位置映射到被调用函数的参数中。如果命名参数比传递给函数的参数还多，剩下的参数值就是`undefined`。

关于命名参数，常规 Javascript 函数在非严格模式下表现了一个奇怪的行为。在非严格模式中，常规 Javascript 函数允许重复的命名参数。以下的代码段展示了这种行为的结果：

```javascript
function logParams(first, second, first) {
  console.log(first, second);
}

// first => 'Hello'
// second => 'World'
// first => '!!!'
logParams("Hello", "World", "!!!"); // "!!!"  "World"

// first => { o: 3 }
// second => [ 1, 2, 3 ]
// first => undefined
logParams({ o: 3 }, [1, 2, 3]); // undefined  [1, 2, 3]
```

正如我们所看到的，`first`参数重复了。因此，它被映射到了传给函数调用的第三个参数的值，完全覆盖了传入的第一个参数。这不是一个有利的行为。

好消息是这个这个行为在严格模式下不被允许。在严格模式下定义一个有重复参数的函数会抛出一个语法错误的异常，这表明不允许重复参数。

```javascript
// Throws an error because of duplicate parameters (Strict mode)
function logParams (first, second, first) {
  "use strict";
  console.log(first, second);
}
```

## 箭头函数怎样对待重复参数？

现在这有关于箭头函数的描述：

_Unlike regular functions, arrow functions do not allow duplicate parameters, wether in strict or non-strict mode. Duplicate parameters will cause a Syntax Errorto be thrown._

```javascript
// Always throws a syntax error
const logParams = (first, second, first) => {
  console.log(first, second);
}
```

## 函数重载

函数重载是定义函数的能力，这样函数就可以被不同的调用签名(形态或者参数的数量)调用。好消息是 Javascript 函数的参数绑定让这个成为可能。

一开始，考虑这非常简单的重载函数，它计算传递给它的任意数量参数平均值：

```javascript
function average() {
  // the number of arguments passed
  const length = arguments.length;

  if (length == 0) return 0;

  // convert the arguments to a proper array of numbers
  const numbers = Array.prototype.slice.call(arguments);

  // a reducer function to sum up array items
  const sumReduceFn = function(a, b) {
    return a + Number(b);
  };

  // return the sum of array items divided by the number of items
  return numbers.reduce(sumReduceFn, 0) / length;
}
```

我尝试让函数定义尽可能详细，这样它的行为就可以清楚地理解。这个函数可以用任意数量的参数调用，从 0 到函数可以取到的最大数量的参数——那应该是 255。

这是其中一些调用`avarage()`函数的结果：

```javascript
average(); // 0
average("3o", 4, 5); // NaN
average("1", 2, "3", 4, "5", 6, 7, 8, 9, 10); // 5.5
average(1.75, 2.25, 3.5, 4.125, 5.875); // 3.5
```

现在尝试用箭头函数的语法复制`avarage()`函数。我的意思是，那是一件多难的事情？首先猜猜——你所有要做的在这：

```javascript
const average = () => {
  const length = arguments.length;

  if (length == 0) return 0;

  const numbers = Array.prototype.slice.call(arguments);
  const sumReduceFn = function(a, b) {
    return a + Number(b);
  };

  return numbers.reduce(sumReduceFn, 0) / length;
};
```

当你现在测试这个函数的时候，你会意识到它抛出了一个`Reference Error`引用错误，猜猜是什么？在所有可能的原因中，抱怨到`arguments`没定义。

## 你有什么问题？

现在这有一些其他的关于箭头函数的东西：

_Unlike regular functions, the arguments binding does not exist for arrow functions. However, they have access to the arguments object of a non-arrow parent function._

基于这个理解，你可以修改`average()`函数，让它成为一个常规函数，其返回结果是立即调用的嵌套箭头函数，这个箭头函数可以访问父函数的`arguments`。这看起来像是这样的：

```javascript
function average() {
  return (() => {
    const length = arguments.length;

    if (length == 0) return 0;

    const numbers = Array.prototype.slice.call(arguments);
    const sumReduceFn = function(a, b) {
      return a + Number(b);
    };

    return numbers.reduce(sumReduceFn, 0) / length;
  })();
}
```

显而易见，这解决了你`arguments`对象未定义的难题。然而，你得使用一个嵌套在常规函数中的箭头函数，对于像这样的简单函数来说，这似乎是不必要的。

## 你可以做得不同吗？

既然这里访问`arguments`对象是一个显而易见的困难，有其他选择吗？答案是有。跟 ES6 的剩余参数打招呼。

使用 ES6 的剩余参数，你可以得到一个数组，它能访问所有或者部分传递给函数的参数。这适用于所有函数，无论是常规函数还是箭头函数。这是它的样子：

```javascript
const average = (...args) => {
  if (args.length == 0) return 0;
  const sumReduceFn = function(a, b) {
    return a + Number(b);
  };

  return args.reduce(sumReduceFn, 0) / args.length;
};
```

哇！剩余参数的援救——你最终找到了一个优雅的解决方法，用箭头函数实现了`average()`函数。

这有一些针对依赖剩余参数访问函数参数的注意事项：

- 剩余参数和函数里的内部`arguments`对象不同。剩余参数是真实的函数参数，然而`arguments`对象是一个受到函数作用域约束的内部对象。
- 一个函数只能有一个剩余参数，而且它必须总是函数的最后一个参数。这意味着一个函数可以有多个命名参数和一个剩余参数的结合。
- 剩余参数，如果存在，可能无法捕获所有的函数参数，尤其是它和命名参数一起使用的时候。然而，当它是唯一函数参数时，它可以捕获所有的参数。另一方面，函数的 arguments``对象总是能捕获所有的函数参数。
- 剩余参数指向一个数组对象，它包含了所有捕获的函数参数，而`arguments`对象指向一个类数组对象，它包含了所有的函数参数。

在你继续之前，考虑另一个简单的重载函数，它将数字从一个数字转换成另一个。这个函数可以被一个到三个参数调用。然而，当它被两个或更少的参数调用时，在执行时，它会交换第二个和第三个参数。

这看起来像一个常规函数：

```javascript
function baseConvert(num, fromRadix = 10, toRadix = 10) {
  if (arguments.length < 3) {
    // swap variables using array destructuring
    [toRadix, fromRadix] = [fromRadix, toRadix];
  }
  return parseInt(num, fromRadix).toString(toRadix);
}
```

以下是其中一些`baseConvert()`函数的调用：

```javascript
// num => 123, fromRadix => 10, toRadix => 10
console.log(baseConvert(123)); // "123"

// num => 255, fromRadix => 10, toRadix => 2
console.log(baseConvert(255, 2)); // "11111111"

// num => 'ff', fromRadix => 16, toRadix => 8
console.log(baseConvert("ff", 16, 8)); // "377"
```

基于你所知道的关于箭头函数没有它自己的`arguments`绑定，你可以用箭头函数语法重写`baseConvert()` 函数，如下：

```javascript
const baseConvert = (num, ...args) => {
  // destructure the `args` array and
  // set the `fromRadix` and `toRadix` local variables
  let [fromRadix = 10, toRadix = 10] = args;

  if (args.length < 2) {
    // swap variables using array destructuring
    [toRadix, fromRadix] = [fromRadix, toRadix];
  }

  return parseInt(num, fromRadix).toString(toRadix);
};
```

注意到之前的我用 ES6 数组结构语法用数组元素设置本地变量的代码片段，也可以交换变量。你可以学到更多关于结构的内容，通过阅读这个指南：[[ES6 Destructuring: The Complete Guide](https://codeburst.io/es6-destructuring-the-complete-guide-7f842d08b98f)]。

## 构造函数

一个常规 Javascript 函数能被`new`关键字调用，这样函数表现得会像一个创造新的实例对象的构造类。

这是一个简单被用作构造器的函数例子：

```javascript
function Square(length = 10) {
  this.length = parseInt(length) || 10;

  this.getArea = function() {
    return Math.pow(this.length, 2);
  };

  this.getPerimeter = function() {
    return 4 * this.length;
  };
}

const square = new Square();

console.log(square.length); // 10
console.log(square.getArea()); // 100
console.log(square.getPerimeter()); // 40

console.log(typeof square); // "object"
console.log(square instanceof Square); // true
```

当一个常规 Javascript 函数被`new`关键字调用，函数内部的`[[Construct]]`方法会被调用来创建一个新的实例对象和分配内存。在那之后，函数体正常执行，将`this`映射到刚被创建的实例对象上。最后，函数隐式的返回`this`(刚被创建的实例对象)，除了在函数定义中指定了不同的返回值。

所有常规的 Javascript 函数都有`prototype`属性。函数的`prototype`属性是一个对象，其中包含了在所有被用作构造器的函数所创建的实例对象中共享的属性和方法。

刚开始，`prototype`属性是一个空对象，`constructor`属性指向该函数。然而，它可以用属性和方法进行扩充，这便于为该函数作为构造函数创建的对象添加更多功能。

这是对先前的`Square`函数的一个轻微的改动，在函数的原型上定义方法而不是在构造函数里。

```javascript
function Square(length = 10) {
  this.length = parseInt(length) || 10;
}

Square.prototype.getArea = function() {
  return Math.pow(this.length, 2);
};

Square.prototype.getPerimeter = function() {
  return 4 * this.length;
};

const square = new Square();

console.log(square.length); // 10
console.log(square.getArea()); // 100
console.log(square.getPerimeter()); // 40

console.log(typeof square); // "object"
console.log(square instanceof Square); // true
```

正如你所说的，所有事情仍然和预想的一样。实际上，这有一点秘密：ES6 的类在背后做了和以上代码片段类似的一些事情——它们是简单的语法糖。

## 那么箭头函数怎么样？

它们也和常规的 Javascript 函数那样共享这个行为吗？答案是否。现在在这里又有一些其他的关于箭头函数的东西：

_Unlike regular functions, arrow functions can never be called with the new keyword because they do not have the [[Construct]] method. As such, the prototypeproperty also does not exist for arrow functions._

令人伤心的是，这是真的。箭头函数不能用作构造器。它们不能被`new`调用。那样做会抛出一个错误，表明该函数不是一个构造器。

结果是，例如`new.target`的绑定，存在于可以被构造器调用的函数中，这并不在箭头函数里。相反，它们使用最接近的非箭头父函数的`new.target`值。

因为箭头函数不能被`new`关键字调用，它们确实也不需要有原型。于是`prototype`属性不在箭头函数中。

既然箭头函数的`prototype`是`undefined`，那试图用属性和方法扩充它或者访问它上面的属性都会抛出错误。

```javascript
const Square = (length = 10) => {
  this.length = parseInt(length) || 10;
};

// throws an error
const square = new Square(5);

// throws an error
Square.prototype.getArea = function() {
  return Math.pow(this.length, 2);
};

console.log(Square.prototype); // undefined
```

## `this`是什么？

现在如果你已经写了一段时间 Javascript 程序，你会注意到每个 Javascript 函数的调用都关联着一个调用上下文，依赖于函数是怎样或者在哪被触发的。

函数中`this`的值非常依赖于函数调用上下文触发的时间，这通常将 Javascript 开发者置于一个境地，他们得问自己一个著名的问题：`this`的值是什么？

这是一个`this`值的总结，指向不同情况的函数调用：

- **使用`new`关键字调用**：`this`指向由内部`[[Construct]]`函数方法创建的新实例对象。`this`(新创建的实例对象)通常默认返回，除非在函数定义时明确指定了不同的返回值。
- 不使用`new`关键字直接调用：在非严格模式，`this`指向 Javascript 宿主环境的全局对象(在 web 浏览器中，通常是`window`对象)。然而，在严格模式下，`this`的值是`undefined`；因此，尝试访问或者设置属性在`this`上会抛出错误。
- **间接使用绑定对象调用**：`Function.prototype`对象提供了三个方法让函数在被调用的时候绑定任意一个对象成为可能，方法名是：`call()`，`apply()`和`bind()`。用这些方法调用函数，`this`会指向指定的绑定对象。
- **作为对象的方法调用**：函数(方法)被调用的时候，`this`指向调用的对象，无论这个方法是否定义在对象自身的属性上或者在对象的原型链中被解析出来。
- **作为事件处理器被调用**：对于用在 DOM 事件监听中的常规 Javascript 函数来说，`this`在事件触发的时候指向目标对象，DOM 元素，`document`或者`window`。

开始，考虑这个绑定在表单提交按钮的简单的点击事件监听 Javascript 函数：

```javascript
function processFormData(evt) {
  evt.preventDefault();

  // get the parent form of the submit button
  const form = this.closest("form");

  // extract the form data, action and method
  const data = new FormData(form);
  const { action: url, method } = form;

  // send the form data to the server via some AJAX request
  // you can use Fetch API or jQuery Ajax or native XHR
}
button.addEventListener("click", processFormData, false);
```

如果你尝试这个代码，你会看到所有东西正确运作。像你先前看到的，在事件监听函数中，`this`的值是触发点击事件的 DOM 元素，在本例中是`button`。

因此，可以使用指向提交按钮的父表单：

```javascript
this.closest("form");
```

这时，你用了一个常规 Javascript 函数作为事件处理器。如果你使用全新的箭头函数语法更改函数定义，会发生什么呢？

```javascript
const processFormData = evt => {
  evt.preventDefault();

  const form = this.closest("form");
  const data = new FormData(form);
  const { action: url, method } = form;

  // send the form data to the server via some AJAX request
  // you can use Fetch API or jQuery Ajax or native XHR
};

button.addEventListener("click", processFormData, false);
```

如果你现在尝试这个，你会注意到你得到了错误。从表面看，`this`的值似乎不是你所期望的。因为一些原因，`this`的值没有指向`button`元素——反而指向了全局`window`对象。

## 你可以做什么去修复`this`绑定？

你还记得`Function.prototype.bind()`？你可以在为提交按钮设置事件监听的时候，用那强制把`this`的值绑定在`button`元素上。这是：

```javascript
// Bind the event listener function (`processFormData`) to the `button` element
button.addEventListener("click", processFormData.bind(button), false);
```

噢！这似乎不是你想要的修复。`this`仍然指向全局`window`对象。这是箭头函数特有的问题吗？这意味着箭头函数不能用在依赖于`this`的事件处理器中吗？

## 你有什么问题？

现在，这是关于箭头函数的最后一件事：

**_不像常规函数，箭头函数没有它们自己的`this`绑定。`this`的值从它们最接近的非父箭头函数中或者全局对象中解析出来。_**

这解释了为什么在事件监听中箭头函数的`this`值指向全局 window 对象(global 对象)了。因为它没有嵌套在父函数中，它会使用最接近的父作用域的`this`值，这里是全局作用域。

然而，这解释不了为什么你不能用`bind()`将事件监听箭头函数绑定在`button`元素上。这有一个解释：

**_不像常规函数，箭头函数里`this`的值保持不变而且在它们的生命周期中不会被改变，与触发上下文无关。_**

箭头函数的这个行为让 Javascript 引擎优化它们成为可能，因为函数绑定可以预先确定。

考虑一个稍微不同的脚本，事件处理器使用对象方法中的常规函数来定义而且依赖于相同对象的另一个方法：

```javascript
({
  _sortByFileSize: function(filelist) {
    const files = Array.from(filelist).sort(function(a, b) {
      return a.size - b.size;
    });

    return files.map(function(file) {
      return file.name;
    });
  },

  init: function(input) {
    input.addEventListener(
      "change",
      function(evt) {
        const files = evt.target.files;
        console.log(this._sortByFileSize(files));
      },
      false
    );
  }
}.init(document.getElementById("file-input")));
```

这是一个有`_sortByFileSize()`方法和`init()`方法的一次性对象字面量，它会立即触发。`init()`方法接受一个文件`input`元素和为这个输入元素设置了一个 change 事件处理器，根据上传文件的大小排序并且记录它们到浏览器的控制台上。

如果你测试这个代码，你会明白在你选择文件上传的时候，文件列表不会被排序和记录到控制台上；相反，控制台会抛出错误。问题来自这一行：

```javascript
console.log(this._sortByFileSize(files));
```

在事件监听函数中，`this`指向触发事件的 DOM 元素，在本例中是`input`元素；于是，`this._sortByFileSize`是 undefined。

为了解决这个问题，你需要绑定事件监听中的`this`到包含这个方法的外部对象中，这样你就能够触发`this._sortByFileSize()`。这里你可以用`bind()`，如下：

```javascript
init: function (input) {
  input.addEventListener('change', (function (evt) {
    const files = evt.target.files;
    console.log(this._sortByFileSize(files));
  }).bind(this), false);
}
```

现在所有事情都如预想的工作。这里不用`bind()`，你可以简单的把常规的事件监听函数替换成箭头函数。箭头函数会使用父`init()`方法中的`this`值，这方法是必须的对象。

```javascript
init: function (input) {
  input.addEventListener('change', evt => {
    const files = evt.target.files;
    console.log(this._sortByFileSize(files));
  }, false);
}
```

在你继续之前，再考虑一个脚本。假设你有一个简单的作为构造器触发的计数器函数，以秒为单位创建倒计时计数器。它使用了`setInterval()`一直倒计时，直到限定时间过去或者 interval 被清除。如下：

```javascript
function Timer(seconds = 60) {
  this.seconds = parseInt(seconds) || 60;
  console.log(this.seconds);

  this.interval = setInterval(function() {
    console.log(--this.seconds);

    if (this.seconds == 0) {
      this.interval && clearInterval(this.interval);
    }
  }, 1000);
}

const timer = new Timer(30);
```

如果你执行代码，你会看到倒计时计时器似乎坏了。它不断在控制台上记录`NaN`。

问题是在回调函数中传递的`setInterval()`，`this`指向全局`window`对象而不是在`Timer`函数作用域中新创建的`instance`对象。于是，`this.seconds`和`this.interval`都是`undefined`。

像之前为了修复它，你可以用`bind()`将`setInterval()`回调函数中的`this`值绑定到新创建的实例对象上，如下：

```javascript
function Timer(seconds = 60) {
  this.seconds = parseInt(seconds) || 60;
  console.log(this.seconds);

  this.interval = setInterval(
    function() {
      console.log(--this.seconds);

      if (this.seconds == 0) {
        this.interval && clearInterval(this.interval);
      }
    }.bind(this),
    1000
  );
}
```

或者更好的是，你可以用箭头函数替换`setInterval()`中的常规回调函数，这样你就能用最接近的非箭头父函数的`this`值，本例中是`Timer`。

```javascript
function Timer(seconds = 60) {
  this.seconds = parseInt(seconds) || 60;
  console.log(this.seconds);

  this.interval = setInterval(() => {
    console.log(--this.seconds);

    if (this.seconds == 0) {
      this.interval && clearInterval(this.interval);
    }
  }, 1000);
}
```

现在你完全理解了箭头函数怎样处理`this`关键字，重要的是要注意箭头函数在这些例子中并不是完美的，例如在你需要保存`this`值的地方，你要定义一个对象方法并且需要引用该对象的时候或者用方法扩充函数的原型并且在方法中需要引用到该目标对象的时候。

## 不存在的绑定

在本文中，你已经知道了几个绑定，它们可以在常规函数中应用，但不存在于箭头函数中。相反，箭头函数从最接近的非箭头父函数中得到这样绑定的值。

综上，这是箭头函数中不存在的绑定列表：

- `arguments`：函数调用时传递的参数列表。
- `new.target`：对使用`new`关键字作为构造器调用的函数引用。
- `super`：函数所属对象原型的引用，倘若它被定义为简洁的对象方法。
- `this`： 函数触发上下文对象的引用

## 结论

嗨，我真的很高兴，你到了本文的结尾，尽管阅读时间很长，并且我非常希望你在阅读中学到一到两样东西。感谢你的时间。

Javascript 箭头函数真的很棒，有这些酷的特点(我们在本文中已经回顾的)，这会让 Javascript 引擎更容易优化它们，这是常规 Javascript 函数不具备的方式。

在我看来，我会说你应该尽可能多的使用箭头函数——除了这些你用不了的情况。
