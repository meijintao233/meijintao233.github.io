---
title: 聊聊Webpack
date: 2018-09-23 13:58:26
tags:
---
### webpack是什么

webpack是目前常用的前端打包构建工具，提高前端工程化水平，将一些重复性的工作交由工具完成，在开发阶段只需要专注于业务的实现即可，webpack也提供了一些优化构建的方法。

webpack在生产发布中主要用到的属性是entry，output，resolve，module，plugins，开发阶段还用到devServer等，在webpack4中提供了dev和prod环境的一些默认配置，通过mode属性去控制。
<!-- more -->
- ***entry***

```javascript
//entry是一个入口，webpack的依赖解析就是从这个入口开始，接收的参数为字符串或者数组或者object
//如果是object，则它的key会被保存在[name]中，可以在output中使用
{
    entry:{
        a:'./src/aaa.js'
    }
}
```

- ***output***

```javascript
//output是输出的路径,将entry中的文件输出到指定路径path中，其中__dirname是当前的配置文件对应的路径
//publicPath在开发环境一般用的比较少，用在生产环境较多，可以自动替换静态资源的路径
//例如我们引入静态资源可能是'./xxx.png',生产环境中有可能需要会放在其他地方，例如放到CDN上，候就可以通过配置这个自动替换，将./替换成CDN的路径
//filename则是输出的文件的名字，[name]则是对应entry中的key，而[hash]则是根据每次打包生成的一个hash值，还有[chunkhash]，这是根据内容生成的hash值，可用于长时间缓存，不过还需要搭配namedchunkplugin和namedmoduleplugin等使用
{
    ouput:{
        filename:'[name].[hash].js',
        path: path.resolve(__dirname,'dist'),
        publicPath: './ZZZ'
    }
}
```

- ***resolve***

```javascript
//resolve是解析方法，webpack为我们提供了一套enhance-resolve
//extensions可以设置后缀，例如对于import aaa from 'XXX';webpack会根据extensions的配置的扩展名进行匹配，而modules则是可以为webpack指定搜索范围，减少不必要的检索，提高效率
//alias则是可以用来配置别名，例如import aaa from '@/XXX'；那么webpack就会根据别名去src/components/XXX.js中加载
//设置别名的好处就是使用了绝对路径，当以后文件结构发生变动的时候，只需要在webpack的配置中更改一次就可以了，不需要每一个文件都更改一次，提高了效率
{
    extensions:['.js','.less'],
    alias:{
      ‘@’：path.resolve(__dirname,'src/components')    
    },
    modules:[path.resolve(__dirname,'node_modules')]
}
```

- ***module***

```javascript
//rules一般是用来设置loader，因为webpack会通过loader将其他文件转换为它能处理的js文件
//rules是一个数组，数组的元素是对象，对象中的test是用正则表达式来筛选处理的文件，include或者exclude可以是字符串或者数组或者函数，表示搜索或者排除其中设置的路径
//use也是一个数组，其中的元素的是对象，loader指定处理的加载器，options是为加载器提供参数，如果传入多个loader，则处理顺序是从数组尾部到头部
{
    rules:[
        {
            test: /\.js$/,
            include:[path.resolve(__dirname, 'src')],
            exclude:[path.resolve(__dirname, 'node_modules')],
            use:[
                {
                    loader: 'babel-loader',
                    options:{
                        presets:[],
                        plugins:[]
                    }
                }
            ]
        }
    ]
}
```

- ***plugins***

```javascript
//lugins中则是配置一系列plugin的实例，处理经过加载器处理过的chunk
[
   new CleanWebpackPlugin(['dist']),
    new HtmlWebpackPlugin({
      template: './src/index.ejs',
      filename: 'index.html',
      inject: true,
    }),
]
```

- ***devServer***

```javascript
//devServer则是开发环境常用的一个服务器，提供热更新和替换的功能，在文件变动的时候可以自动刷新，需要配合new webpack.HotModuleReplacementPlugin()这个plugin使用
//devServer并不会在磁盘上生成文件，而是把文件储存在内存中，所以我们通过fs模块没办法操作，可以通过memory-fs模块操作，contentBase则是类似于output中的path
//在开发环境可以利用proxy将一些请求代理到我们的服务器中进行测试
{
    contentBase: './dist',
    hot: 'true',
    proxy:{}
}
```



### webpack原理及使用

- ***打包流程***

webpack的理念是一切皆模块，首先从entry出发，构建出依赖图，然后根据依赖图加载文件，通过loader把各种文件转换成它能处理的js模块，在loader中的处理就像是水流从水管中流过，buffer数据流就像水流，流经每个loader，经过处理后流出到下一个loader中，然后用plugin进行处理。webpack中有两个关于模块的概念，分别是module和chunk，我们打包出来的都是chunk，而chunk由很多module构成，每个文件中引入的外部文件就是module。

- ***加载的实现***

对module的加载，webpack的处理和node的require模块差不多，也是加载文件，缓存加载过的文件，避免循环引用，然后在外层套一层函数定义把参数传入，隔离出作用域。

```javascript
(function(modules) { 
 	//将module缓存
 	var installedModules = {};

 	//这里是形成闭包返回，这里的实现和node的require模块的实现一样
 	function __webpack_require__(moduleId) {
		//如果module已经被加载过，直接返回出去
 		if(installedModules[moduleId]) {
 			return installedModules[moduleId].exports;
 		}
 		//未被加载过则进行加载并记录
 		var module = installedModules[moduleId] = {
 			i: moduleId,//唯一的moduleid
 			l: false,//是否加载过的标志
 			exports: {}//要暴露的方法
 		};

 		//将context绑定到module.exports中，将module，module.exports,__webpack_require__作为参数传入
 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

 		//标志加载完成
 		module.l = true;

 		//将module.exports返回
 		return module.exports;
 	}


 	//静态方法，将modules绑定，此处的modules是一个对象，对象里面是一个函数
 	__webpack_require__.m = modules;

 	//静态方法，将缓存的module信息返回出去
 	__webpack_require__.c = installedModules;

 	// 静态方法，这里可以处理es6的ESM，ESM引入的时候_esModele的值为true，exports default将变为commonJS的exports.default
 	__webpack_require__.r = function(exports) {
 		if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
 			Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
 		}
 		Object.defineProperty(exports, '__esModule', { value: true });
 	};

 	//......

 	//moduleid以路径为映射，输出的是module.exports的内容
 	return __webpack_require__(__webpack_require__.s = "./b.js");
 })
 ({//这里是将字符串通过eval()函数激活
  (function(module, exports) {
    eval("\n\n\nmodule.exports = {\n  test: function(){\n    console.log('bbbbb')\n  }\n}\n\n//# sourceURL=webpack:///./b.js?");
  })
});
```



我们需要注意的是，webpack对module和chunk的命名处理是在内部维护了一个id的映射，并且id是自增的，也就是说，文件的增加减少也会导致命名的变化，即使内容并没有改变，因此当我们需要对一些第三方库和框架进行缓存的时候，需要用到namedmoduleplugin和namedchunkplugin进行处理，使得命名不会因为文件的变动而在打包构建的时候发生变化，破坏了浏览器的缓存效果。

- ***分包优化机制***

针对打包构建，webpack3用的是commonschunkplugin，基于父子关系依赖图进行的公用包分割模式，主要是根据minchunk，也就是引用的次数进行划分。而在webpack4中，舍弃了commonschunkplugin而起用了spiltchunkplugin，这是基于graghGroup进行的分割，其中有一个chunks属性，提供了三种模式，默认是"async"，还有的是"initial"，"all"，为了弄清这三种模式的区别，利用了可视化工具尝试进行了打包，其中在a，b文件中，lodash都是动态引入的，而jquery都是静态引入的，moment则是一个动态一个静态的

![微信图片_20180923185725](C:\Users\meijintao\Desktop\微信图片_20180923185725.png)

这是initial模式的打包结果，这里a，b中的所有模块都被提取了出来，而且jquery被合并了 ，但是moment和lodash却没有被合并

![initial](C:\Users\meijintao\Desktop\initial.png)

这是async模式的打包结果，这里a，b中的jquery没有被提取出来 ，而lodash则被提取了出来，a中的moment被提取了，而b中的却没有

![async](C:\Users\meijintao\Desktop\async.png)

这是all模式的打包结果，a，b中的模块都被提取了，而且相同的模块都合并了

![alll](C:\Users\meijintao\Desktop\all.png)

从以上三张图可以看出，三种模式的区别，在async模式下，只有动态加载的公共包才会被提取出来，在initial模式下，其涵盖了async的功能，所有包都被提取了出来，但是只有静态引入的包才会被合并，而在all模式中，所有的包都被提取出来并且合并了。

由此可见，splitchunk的功能还是很强大的，我们可以根据不同的需求使用不同的模式进行分包，例如框架这些是很少会变动的，而且体积也比较大，这些可以利用all模式打包到一起进行缓存，然后是第三方库这些变动的可能行也是比较小的，这些可以打包到一起进行缓存，剩下的公共包根据实际情况打包，在第一次进入页面的时候进行加载，减少重复的代码。

- ***静态引入和动态*** 

通过动态引入可以实现按需加载的功能，webpack中支持es6的import()语法，对于文件的引入大致有三种方案。

1. CMD，这是同步的实现方案，因为这个是同步的加载，而且是运行时操作，加载后的文件变转换成一个exports对象，其中this会指向他，由于是在服务器中，直接从硬盘中读取文件，阻塞线程也不怕，所以同步加载也可以。
2. AMD，这是在异步的实现方案，这个一般运行在浏览器端，也是运行时操作，因为浏览器需要从服务器请求数据，所以使用异步的方案比较好，不然会阻塞当前线程，无法处理用户的交互信息。

3. ESM，这是ES6提出来的方案，利用import引入，也可以根据需要只引入部分代码，利用export将代码分割，相当于一个文件中每一个export都是一个小文件。这并不是在运行时进行加载，而是在编译的时候就确定了加载的文件，因此import中的文件路径不支持变量，只能写常量，能够export的只是有确定值的变量或者函数，对象等，否则会报错 。
4. 可以利用import()进行动态加载，webpack会将这些文件进行codesplit自动分割成不同的chunk，而且在webpack中可以以变量加字符串的形式引入，例如import(`../models/${component}`)，这是因为webpack有一个context，这种写法webpack会将../models中的文件全都加载到chunk中，或者可以使用require.context方法实现动态引入，这样也可以实现路由的去中心处理，也可以实现对一些使用频次较高的库或者插件的自动引入而不需要每个页面都手写一次

- API钩子

webpack也提供了很多钩子，例如module.hot的热更新钩子，还有loader处理相关的钩子，plugin相关的钩子可以控制plugin运行的位置

webpack的配置文件是一个对象，我们可以将这个对象config作为参数传入webpack(config)，这会产生一个compiler实例，通过compiler.run可以启动webpack等等



### 最后

由于框架提供的脚手架用起来不方便，也不能随便跟着脚手架和框架的更新而更新，于是尝试着用webpack去打包，并且在团队中用起来，现在搭好了开发和生产环境，在过程中也研究了一下webpack的相关内容，准备进行一些优化。

由于antd中的less文件的样式如果使用了cssmodules，那么会在每个命名后面加上hash，样式无法正常显示，因此在使用loader的时候需要分开处理，对node_modules中的less不进行cssmodules处理。

由于antd的默认icon无法满足实际需要，因此需要对其进行扩展，但是在引入静态资源的时候需要对路径进行映射，因此需要设置less-loader中options的path，改变context，将根路径/映射到/public中，而且在引入自己的iconfont的时候通过增加选择器权重，兼容性的扩充原有的anticon。





