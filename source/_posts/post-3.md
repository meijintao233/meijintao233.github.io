---
title: JS的原型链和继承
date: 2018-08-14 21:51:32
tags:
---

## 原型和原型链

原型prototype，在创建新函数的时候，会自动生成，而prototype中也会有一个constructor，回指创建该prototype的函数对象。

\_\_proto\_\_是对象或者实例中内置的[[prototype]]，其指向的是产生该对象的对象的prototype,在浏览器中提供了\_\_proto\_\_让我们可以访问，通过\_\_proto\_\_的指向形成的一个链条，就称做原型链，原型链的整个链路是：**实例对象- ->构造函数的prototype- ->Object的prototype- ->null**。

我们在访问对象的属性或者方法的时候，首先从本对象寻找，如果本对象不存在该属性或者方法时，就会沿着原型链向上寻找，直至找到该属性或者方法，或者到null时停止。

这也解释了为什么数组对象上没有push，pop，shift，unshift等方法，却可以访问。
<!-- more -->
---	
## constructor
constructor属性指向的是生成该函数(对象)的函数(对象)，例如
```javascript
var a = function(){};
var b = new a();
var c = {};
var d = [];
//以下皆为true
console.log(b.constructor === a) //因为实例b是由构造函数产生的
console.log(a.constructor === Function)//函数a实际是Function的实例，同理
console.log(c.constructor === Object)//空对象c是Object的实例
console.log(d.constructor === Array)//空对象c是Object的实例
console.log(Object.constructor === Function)//Object自身就是一个构造函数，同理
console.log(Array.constructor === Function)//Array自身也是一个构造函数
//---------------------------------------------------------------
//首先__proto__指向的是产生该对象的对象的prototype，
//也即a.prototype，prototype中也的constructor，回指创建该prototype的函数对象，也即函数a
console.log(b.__proto__.constructor === a)
```
这里顺便说一下instanceof，**A instanceof B **是在 A 的原型链中找 B 的 prototype，找到返回 true，找不到返回 false
```javascript
//有个奇怪的现象,下面都返回true，这是为什么呢？
//因为JS中一切都继承自Object，除了最顶层的null，
//所以在Function的原型链中能找到Object.prototype
console.log(Function instanceof Object)
//而Object自身就是一个构造函数，因此在Object的原型链中也能找到Function.prototype
console.log(Object instanceof Function)
```

---

## 通过原型链实现继承
由上面的分析，我们可以利用原型链实现继承的逻辑，继承是面向对象中的一个很重要的概念
```javascript
function Dog(name){
	this.name = name;
	this.say1 = function(){
		console.log(this.name)
	}
}
Dog.prototype.say2 = function(){
	console.log(this.name)
}
Dog.prototype.test = 1
//say本来应该是所有Dog实例的共有方法，
//如果放在构造函数中，那么就会导致没办法数据共享，每一个实例都有自己的属性和方法的副本，这是对资源的极大浪费
//如果放在Dog.prototype中，那么利用原型链的特性，就可以让所有实例共用一个方法，
//需要注意的是，由于共用了一个方法，对属性的更改是对所有实例透明的

var dog1 = new Dog('lalala'); 
let dog2 = new Dog('wahaha');
dog1.test++;//2
dog2.test++;//3
console.log(dog1.say1 === dog2.say1)// false
console.log(dog1.say2 === dog2.say2)// true



//现在，我们可以尝试着去实现继承了
//我们是通过原型链去实现继承的，
//之前的原型链是：Dog实例 --> Dog函数 --> Object --> null
//那么现在的原型链需要改成 Dog实例 --> Dog函数 --> Dog父类(Animal函数) --> Object --> null
//第一种方案，改变Dog函数的prototype，让他指向Animal的实例
function Animal(){
	this.species = 'unknown';
}
Dog.prototype = new Animal();
//这里改变后会导致prototype中的constructor改变
Dog.prototype.constructor = Dog;

//第二钟方案，改变Dog函数的prototype，让他指向Animal的prototype
function Animal(){}
Animal.prototype.species = 'unknown';
Dog.prototype = Animal.prototype;
//这里改变后会导致prototype中的constructor改变
Dog.prototype.constructor = Dog;

//第三种方案，调用apply或call,将Animal的this绑定到Dog中
function Animal(){
	this.species = 'unknown';
}
function Dog(name){
	Animal.apply(this, arguments);
	this.name = name;
}

//第四种方法，通过Object.create()方法实现继承，过滤掉了父类实例属性,Dog.prototype中就没有了Animal的实例化数据了
//这种方法也是ES6中Class被babel编译成ES5所用的方法
function Animal(){
	this.species = 'unknown';
}
function Dog(name){
	Animal.apply(this, arguments);
	this.name = name;
}
//这里模拟了 Dog.prototype = Object.create(Animal.prototype)
var f = function(){};
f.prototype = Animal.pototype;
Dog.prototype = new f();
Dog.__proto__ = Animal;
//这里改变后会导致prototype中的constructor改变
Dog.prototype.constructor = Dog;


//现在就能访问到Animal中的species属性了
var dog = new Dog('lalala');
dog.species;//unknown
```
以上这些就是利用原型链实现继承的一些方法

## ES6的class类
有了以上的知识，我们就可以研究一下ES6的class类了，这个语法糖能让我们更容易的实现类和继承，其提供了extends，static，super等关键字

```javascript
//这是es6的代码实现
class Parent {
  static l(){
    console.log(222)
  }
  constructor(m){
    this.m = m
  }
  get(){
    return this.m;
  }
}
class Child extends Parent {
  constructor(n){
    super(4);
    this.n = n;
  }
  get(){
    return this.n
  }
  set(a){
    this.n = a;
  }
}

//这是利用babel编译之后的es5的实现
//_createClass是一个自执行函数，作用给构造函数绑定静态方法和动态方法
//对于静态的static关键字声明的变量，会直接绑定在函数对象上，作为静态属性(方法)
//对于在class中声明的函数方法，则会绑定在构造函数的prototype上，通过Object.definePropety方法
var _createClass = function () {
  function defineProperties(target, props) {
    for (var i = 0; i < props.length; i++) {
      var descriptor = props[i];
      descriptor.enumerable = descriptor.enumerable || false;
      descriptor.configurable = true;
      if ("value" in descriptor) descriptor.writable = true;
      Object.defineProperty(target, descriptor.key, descriptor);
    }
  }
  return function (Constructor, protoProps, staticProps) {
    if (protoProps) defineProperties(Constructor.prototype, protoProps);
    if (staticProps) defineProperties(Constructor, staticProps);
    return Constructor;
  };
}();

//如果父函数没有返回值或者返回值不为object或者function，则返回子类的this
function _possibleConstructorReturn(self, call) {
  if (!self) {
    throw new ReferenceError("this hasn't been initialised - super() hasn't been called");
  }
  return call && (typeof call === "object" || typeof call === "function") ? call : self;
}

//_inherits就是extends关键字发挥的作用，实现了继承的功能。利用&&的短路特性，对superClass做了容错性处理，然后将子类Object.create()传了两个参数，一个参数是父类superClass.prototype,作用在上面解释继承的方法时讲过了，第二个参数是一个键值对，key代表着属性，value则和Object.definePropety中descriptor一样，这里改变constructor的目的，也在解释继承时讲过了，最后将subClass.__proto__指向superClass
function _inherits(subClass, superClass) {
  //...省略
  subClass.prototype = Object.create(superClass && superClass.prototype, {
    constructor: {
      value: subClass,
      enumerable: false,
      writable: true,
      configurable: true
    }
  });
  if (superClass) Object.setPrototypeOf ? Object.setPrototypeOf(subClass, superClass) : subClass.__proto__ = superClass;
}

//_classCallCheck是保证构造函数不能被当成普通函数调用，需要用new关键字
function _classCallCheck(instance, Constructor) {
  if (!(instance instanceof Constructor)) {
    throw new TypeError("Cannot call a class as a function");
  }
}

var Parent = function () {
  _createClass(Parent, null, [{
    key: "l",
    value: function l() {
      console.log(222);
    }
  }]);

  function Parent(m) {
    _classCallCheck(this, Parent);

    this.m = m;
  }

  _createClass(Parent, [{
    key: "get",
    value: function get() {
      return this.m;
    }
  }]);

  return Parent;
}();

var Child = function (_Parent) {
  _inherits(Child, _Parent);

  function Child(n) {
    _classCallCheck(this, Child);
	
	//由于在_inherits中将subClass(child).__proto__指向了superClass(Parent),所以这里即是Parent.call(this,4),即这里执行的是super函数，super也可以调用父类的静态方法，
	//如果父函数没有返回值或者返回值不为object或者function，则返回子类的this    
	var _this = _possibleConstructorReturn(this, (Child.__proto__ || Object.getPrototypeOf(Child)).call(this, 4));

    _this.n = n;
    return _this;
  }

  _createClass(Child, [{
    key: "set",
    value: function set(a) {
      this.n = a;
    }
  }]);

  return Child;
}(Parent);
```

---
## 总结
- 通过以上分析，对原型和原型链有了更加深入和清晰的了解，也熟悉了constructor和instanceof的用法，加深了基于原型链的继承方式的了解，理清了这块知识。
- 在对ES6的class通过babel编译后的源码的分析中，也了解到了Object.create和Object.setPrototypeOf的用法，挖掘了如何去模拟super，extends和static的实现。