---
title: wepy浅析
date: 2018-07-29 12:14:11
tags:
---

wepy是腾讯内部团队开源的一个类vue框架，相对于原生小程序，这个框架对开发流程有了更高的抽象，也降低了初学者的开发难度，可是对于vue开发者来说，开发体验也降低了。**不过wepy解决了以下问题:**

- 无法引用npm包的问题
- 如何进行编译
- 更有效的组织代码，以及组件间的通信
- request的并发限制，
- 原生频繁使用setdata，会有性能问题
- 如何划分作用域
- 提供了slot

<!-- more -->
#### 一、wepy是什么
wepy是腾讯内部团队开源的一个类vue框架，相对于原生小程序，这个框架对开发流程有了更高的抽象，也降低了初学者的开发难度，可是对于vue开发者来说，开发体验也降低了。

#### 二、wepy解决了什么问题
- 1.无法引用npm包的问题
- 2.如何进行编译
- 3.更有效的组织代码，以及组件间的通信
- 4.request的并发限制，
- 5.原生频繁使用setdata，会有性能问题
- 6.如何划分作用域
- 7.提供了slot
#### 三、wepy不足之处
- 1.脏检查有可能带来性能消耗，因为页面切换过程中会执行一次脏检查，如果开发者知道页面没变动，可以手动停止脏检查机制（可能可以通过设置$$phase的值解决）
- 2.repeat中无法使用watch
- 3.双向绑定过多会出现性能问题
- 4.一个组件的数据发生变化，全部组件的都会刷新一次，这个在无限滚动中容易出现性能问题（如果repeat过程中能分配不同的实例可能能解决）
#### 四、解决方案
- 1.针对原生小程序不支持npm包的问题，wepy做了改进，对package.json中的依赖包，wepy模仿require的加载机制，将文件拷贝到node_modules中，针对一些特殊的包进行了hack处理，替换了相关字段，这个是在complie-script的resolveDeps方法中解决的。

- 2.compile根据不同的文件名后缀调用不同的方法进行编译，compile-wpy调用resolveWpy方法进行了一系列的处理，把代码变换成xmldom进行操作，将wpy文件中的config放入rst.config中，样式放入rst.style，将components放入rst.template.components（通过正则表达式获取），而components中的props以及events属性，则放入rst.script.code中，最后拆解style、template、script，生成rst对象
```javascript
rst = {
  moduleId: moduleId,
  style: [],
  template: {
    code: '',
    src: '',
    type: '',
    components: {},
  },
  script: {
    code: '',
    src: '',
    type: '',
  },
  config: {},
};
```
3.原生小程序中，一个page下有四个文件```.wsss，.wxml，.wxs，.json```，看得头晕。wepy提供了一种类vue的机制，把style，template，script写在一起，然后再编译成小程序能识别的文件，可读性提高了。而且整个page可以由一个page下的多个组件构成，在编译的时候会形成组件树，而各个组件之间可以通过$invoke、$emit方法进行通信，而父子组件之间可以通过props属性进行传递，源码上实现$watch的部分和Vue的$watch基本一致，$root属性指向的是该component所处的page页面的实例，而$parent则是指向改component的父组件，由于组件树的存在，实现$invoke、$emit变得不难，只要触发处一直向$parent递归即可。

-4.原生小程序中有并发请求限制，而wepy则通过RequestMQ方法解决了这个问题，主要是通过维护一个待请求队列和请求未返回队列，以及一个映射对象，并且通过递归进行轮询，如果超过了则等待前面的请求回来再发起下一个请求。
```javascript
//单例模式
let RequestMQ = {
    map: {},//映射对象
    mq: [],//等待发起请求
    running: [],//正在进行的请求
    MAX_REQUEST: 5,//最大并发数
    push (param) {
        param.t = +new Date();//生成时间戳，作为请求的识别码
        while ((this.mq.indexOf(param.t) > -1 || this.running.indexOf(param.t) > -1)) {
            //如果时间戳存在，则进行处理，保证识别码是唯一的
            param.t += Math.random() * 10 >> 0;//这里右移0是为了取整
        }
        this.mq.push(param.t);//将请求放入等待发起请求的队列中
        this.map[param.t] = param;//用map对象记录这个请求，以时间戳作为标记
    },
    next () {
        let me = this;//保存该作用域
        if (this.mq.length === 0)//如果没有需要发起请求，那么直接返回
            return;
        if (this.running.length < this.MAX_REQUEST - 1) {
          //正在发起的请求数不能超过最大允许请求数
          //如果不足，那么可以从待请求队列中获取需要发起的请求
          let newone = this.mq.shift();
          //从映射对象中根据时间戳标识码获取实际的请求参数或者对象
          let obj = this.map[newone];
          let oldComplete = obj.complete;
          obj.complete = (...args) => {
              //请求完成之后将请求其从正在发起的队列中移除
              me.running.splice(me.running.indexOf(obj.t), 1);
              //并且从map映射对象中移除
              delete me.map[obj.t];
              //这里是安全性检查，利用&&的短路原理，
              //不太理解为什么还要再执行一次
              oldComplete && oldComplete.apply(obj, args);
              //递归调用，相当于是轮询
              me.next();
          }
          //记录下该请求已经发起
          this.running.push(obj.t);
          //调用wx.request发起请求
          return wx.request(obj);
        }
    },
    request (obj) {
        obj = obj || {};
        obj = (typeof(obj) === 'string') ? {url: obj} : obj;
        this.push(obj);
        return this.next();
    }
};
```
5.由于小程序将逻辑层和视图层完全分离，而他们通信的时候是通过wxJSBridge 实现的方式，每次setData的调用都是一次进程间的通信过程，因此花销较大。而wepy则对其进行了优化，将所有数据收集进行脏检查之后再调用setdata。
```javascript
$apply (fn) {
    //如果传入函数作为参数，那么会先执行函数再进行脏检查
    if (typeof(fn) === 'function') {
        fn.call(this);
        this.$apply();
    } else {
        //如果流程中正在进行脏检查($$phase为digest)时等待，因为流程中同一时间只能有一次脏检查
        if (this.$$phase) {
            this.$$phase = '$apply';
        } else {
            this.$digest();
        }
    }
}

$digest () {
    let k;
    //初始值，脏检查阶段进行比对
    let originData = this.$data;
    //脏检查开始的标识
    this.$$phase = '$digest';
    this.$$dc = 0;
    while (this.$$phase) {
        this.$$dc++;
        if (this.$$dc >= 3) {
            throw new Error('Can not call $apply in $apply process');
        }
        //传入setData的参数
        let readyToSet = {};
        //组件如果有computed属性，则先执行computed中的方法
        if (this.computed) {
            for (k in this.computed) { 
                let fn = this.computed[k], val = fn.call(this);
                if (!util.$isEqual(this[k], val)) { 
                    readyToSet[this.$prefix + k] = val;
                    this[k] = util.$copy(val, true);
                }
            }
        }
        //比较data的初始值和现值，深度比较，不同则进行深拷贝
        for (k in originData) {
            if (!util.$isEqual(this[k], originData[k])) {
            //如果有watcher，则执行watch属性中对应的观测数据函数，如果是字符串，则执行methods属性中对应的函数(参数为newValue，oldValue)
                if (this.watch) {
                    if (this.watch[k]) {
                        if (typeof this.watch[k] === 'function') {
                            this.watch[k].call(this, this[k], originData[k]);
                        } else if (typeof this.watch[k] === 'string' && typeof this.methods[k] === 'function') {
                            this.methods[k].call(this, this[k], originData[k]);
                        }
                    }
                }
                
                //将初始值和现值不一致的数据收集到对象中，稍后执行一次setdata
                readyToSet[this.$prefix + k] = this[k];
                this.data[k] = this[k];
                //深拷贝现值到初始值中，作为下一次脏检查的初始值
                originData[k] = util.$copy(this[k], true);
                //对repeat中的组件数据进行更新，repeat中不能watch有可能是这里的逻辑有冲突
                if (this.$repeat && this.$repeat[k]) {
                    let $repeat = this.$repeat[k];
                    this.$com[$repeat.com].data[$repeat.props] = this[k];
                    this.$com[$repeat.com].$setIndex(0);
                    this.$com[$repeat.com].$apply();
                }
                //对props的值进行更新，实现父子传值的关键，以及双向绑定的关键
                if (this.$mappingProps[k]) {
                    Object.keys(this.$mappingProps[k]).forEach((changed) => {
                        let mapping = this.$mappingProps[k][changed];
                        if (typeof(mapping) === 'object') {
                            this.$parent.$apply();
                        } else if (changed === 'parent' && !util.$isEqual(this.$parent.$data[mapping], this[k])) {
                            this.$parent[mapping] = this[k];
                            this.$parent.data[mapping] = this[k];
                            this.$parent.$apply();
                        } else if (changed !== 'parent' && !util.$isEqual(this.$com[changed].$data[mapping], this[k])) {
                            this.$com[changed][mapping] = this[k];
                            this.$com[changed].data[mapping] = this[k];
                            this.$com[changed].$apply();
                        }
                    });
                }
            }
        }
        //对readyToSet对象收集到的变化数据进行一次setData
        //$$nextTick为视图渲染完成后，执行的回调，可以是promise，也可以是普通函数，如果在调用this.$nextTick中不传参数，则必为promise
        //我们可以利用this.$nextTick进行一些操作
        if (Object.keys(readyToSet).length) {
            this.setData(readyToSet, () => {
                if (this.$$nextTick) {
                    let $$nextTick = this.$$nextTick;
                    this.$$nextTick = null;
                    if ($$nextTick.promise) {
                        $$nextTick();
                    } else {
                        $$nextTick.call(this);
                    }
                }
            });
        } else {
            if (this.$$nextTick) {
                let $$nextTick = this.$$nextTick;
                this.$$nextTick = null;
                if ($$nextTick.promise) {
                    $$nextTick();
                } else {
                    $$nextTick.call(this);
                }
            }
        }
        this.$$phase = (this.$$phase === '$apply') ? '$digest' : false;
    }
}

$nextTick (fn) {
        if (typeof fn === 'undefined') {
            return new Promise((resolve, reject) => {
                this.$$nextTick = function () {
                    resolve();
                };
                this.$$nextTick.promise = true;
            });
        }
        this.$$nextTick = fn;
    }
```

脏检查机制在Angular中使用，数据量大的时候具有优势，而Vue和React用的是依赖收集的方式，这需要在每次更新数据之后重新进行依赖收集，在数据量大的时候频繁的收集依赖就不如脏检查高效了。
- 6.通过给不同组件的变量属性加上$prefix组件名作为前缀，为组件提供了一个"作用域"，而这个前缀是components中的key值，因此如果多个同名组件设置相同的key值，就会共享同一个实例。
- 7.wepy提供了slot插槽的功能，这为提高组件的可复用性提供了帮助。可以将通用的组件高度抽象，像vue可以利用scope-slot进行封装
```javascript
<toggle>
  it's a slot.
</toggle>

//toggle.wpy
<template>
 <div>
   <slot />
 </div>
</temlate>
```
#### 五、总结
wepy比起原生小程序，对于开发过程更便捷了，在编译过程中生成了rst对象，抽象出各个组件间的信息，也支持了npm包以及解决了并发限制的问题，在解决npm包上用了拷贝的方式，以及用消息队列的方式对请求数进行控制。利用脏检查机制，找出改变的数据，然后再一次过进行setData也解决了多次setData带来的性能损耗，不过还是难以解决视图层和逻辑层进程间通信制约了数据传输不能过大的问题。以组件树这种数据结构对page和组件的关系进行构建，便于进行父子间组件通信的实现，而通过以组件命名作为前缀的变量和属性命名，让组件之间的变量有了"作用域"。针对wepy中不足的地方，我也正在思考如何解决，希望能为wepy贡献一份力。
此外wepy利用了getter对事件进行劫持，实现了拦截器的效果，以及mixins复用，提供了可以作为基类组件的用法，这些都是wepy的优点，

#### 未完待续。。。。