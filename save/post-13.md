---
title: webpack源码分析及loader、plugin功能实现
date: 2019-10-10 21:05:11
tags:
  - webpack
  - 前端
  - frontend
---

### 一切皆模块，模块皆可看成是依赖，入口也是依赖，而且是一切依赖的起点

### **概念**

#### dependency

- dependency 抽象类 -》 ModuleDependency -〉SingleEntryDependency、MuiltEntryDependency，定义了不同入口的信息，由抽象到具体实现，保存一些位置信息和包的信息，request 是请求的地址，即该文件位置，type 是文件类型，dependencies 保存的是依赖的信息

  <!-- more -->

  ![image-20191007164615180](./image-20191007164615180.png)

  ![image-20191007165415786](./image-20191007165415786.png)

  ![image-20191007165547947](./image-20191007165547947.png)

  ![image-20191007164644498](./image-20191007164644498.png)

  ![image-20191007164659016](./image-20191007164659016.png)

#### Source

- Source 抽象类，source，size，map，sourceAndMap 等抽象方法，

  RawSource 子类，用于纯文本 代码字符串存在\_value

  ![image-20191010103403330](./image-20191010103403330.png)

  concatSource 子类，连接各种代码模版，存于 children 数组中，children 中可能是代码字符串或者其他 Source 实例

  ![image-20191010103307217](./image-20191010103307217.png)

  CachedSource 子类，用于缓存其他 Source 实例

  ![image-20191010103338688](./image-20191010103338688.png)

#### XXXModuleFactory

- XXXModuleFactory 工厂方法都带 create 方法

  ![image-20191010104254496](./image-20191010104254496.png)

  （nmf）

  ![image-20191010104220405](./image-20191010104220405.png)

  ![image-20191010103734944](./image-20191010103734944.png)

-

#### childCompiler

- "make", "compile", "emit", "afterEmit", "invalid", "done", "thisCompilation" 不能用于子 compiler，thisCompilation 和 compilation 区别，childCompile 分离部分 compile 的处理逻辑，最终通过 asset 放回到 compile 中，childCompile 没有 emit 的钩子，因此无法生成最终文件

  ![image-20191010113527011](./image-20191010113527011.png)

####Module

- dependenciedBlock 抽象类->Module-》（normalmodule，contextmodule，dllmodule，externalmodule，delegatemodule）

  ![image-20191010105914175](./image-20191010105914175.png)

  ![image-20191010110028457](./image-20191010110028457.png)

  Issuer:发起人，意思是发起请求的文件，dependencies：依赖，被依赖的文件，module 指向发起请求的文件，与 node 的 parent 和 children 类似

  ![image-20191010111554257](./image-20191010111554257.png)

  ![image-20191010111621313](./image-20191010111621313.png)

-

#### parser 注入

- nmrfactor 中触发顺序 beforeresolve，factory，resolver，afterresolve，createModule，module，createParser，parser，其中在 createmodule 和 module 阶段都可以生成或改变 module 实例，createParser 可以改变 parser，目前默认 parser 为 acron，[Js|json]ModulesPlugin 注入

  ![image-20191010104635345](./image-20191010104635345.png)

  ![image-20191010104534033](./image-20191010104534033.png)

#### chunk

- chunk 的划分，Entrypoint(入口，在 addEntry 时记录)，按需加载（require.ensure），他们都是 chunkGroup 的子类，自成一个 group

  ![image-20191010102833987](./image-20191010102833987.png)

按需加载（require.ensure）

![image-20191010113109055](./image-20191010113109055.png)

![image-20191010113209274](./image-20191010113209274.png)

![image-20191010113227387](./image-20191010113227387.png)

#### compiliation

- addEntry（EntryOptionPlugin）-》\_addModuleChain-〉 buildModule-》build-》dobuild-》afterBuild-〉processModuleDependencies-》addModuleDependencies-〉processModuleDependencies

  seamaphore 相当于是信号量，并发时上锁防止竞争问题，addReason 表示依赖的来源，create 执行完生成了 module 实例，依赖的 module 属性指向了该 module，同时 issure 表示是发起者，意思就是哪个模块发起了打包请求，在 addModule 中进行是否 rebuild 和缓存处理，然后调用 module.build 方法使用 loader 进行解析，各种类型的 module 都实现了抽象类 Module 的 build 方法，进而调用 doBuild，生成了 loaderContext，runloaders 调用 loaders，这里的 result 是一个对象，

  result.result 数组中第一个元素就是我们自定义 loader 时通过 return 返回的字符串，第二个元素是 sourcemap，第三个元素则是 info 对象，ast 存放在里面，然后创建一个 source 对象，我们在 emit 阶段构建文件时也是以这种方式，对象 source，size 函数，添加依赖。module 有 source 方法可以返回字符串。

  紧接着在 build 的回调中借助 acron 的 parse 方法将 string 解析成 ast，在遍历 ast 的过程中会触发 parser 对应的钩子，而 parser 钩子通过各种 XXXdependencyPlugin 注册的，parse 方法将会解析出依赖。

  根据 normalModule 调用 build、dobuild 方法，利用 run-loader 读取文件，

  normalModuleFactory parse 阶段调用 acron 处理成 ast

  acron，parser.parse 转化 ast，XXXparserPlugin 监听 hooks 调用 addDependency 方法收集依赖

  normalModuleFactory 比较重要的钩子 beforeresolve 可拦截 module 生成，afterResolve，parser 阶段

  至此完成整个 module 的 create 过程，这时候回到 create 的 callback，会执行 afterbuild 方法，开始处理 module 的依赖。

  processModuleDenpendcy 方法，首先针对依赖的类型将不同类型的依赖调用不同的依赖处理 plugin，这里调用 create 时的 issurer 已经有值，为被依赖 module 的路径，然后开始不断循环构建 module

  ![image-20191007170307562](./image-20191007170307562.png)

  ![image-20191007170422250](./image-20191007170422250.png)

  ![image-20191009160956574](./image-20191009160956574.png)

  ![image-20191007170859880](./image-20191007170859880.png)![image-20191008093117503](./image-20191008093117503.png)

  ![image-20191008094052460](./image-20191008094052460.png)

  ![image-20191008094707782](./image-20191008094707782.png)

  ![image-20191007163637647](./image-20191007163637647.png)

  ![image-20191008152417578](./image-20191008152417578.png)

- build 完成后用 parser.parse 进行解析，获取依赖，parser 阶段绑定了各种 xxxPlugin 的解析钩子，最迟可以在 parse 阶段用 addDependcy 添加依赖，parser 钩子能拿到 ast 的各种信息，例如 require.ensure 等

  ![image-20191010105202556](./image-20191010105202556.png)

- dllplugin 单独打包第三方文件

- 一般在 compiler，compilation 阶段设置一些 nmr 的东西，例如依赖工厂类

- externals module，会在 nmr 的 factory 阶段创建 module 实例

- 对入口的处理会放在 compiliation 阶段(能拿到 nmf 参数)和 make 阶段（compilition 的 addEntry 方法），

  ![image-20191007161348519](./image-20191007161348519.png)

  ![image-20191007164943595](./image-20191007164943595.png)

#### Loader

- loader pitch 递归结束后 normal，如果 pitch return 了值，那么后续 loader 不执行（style-loader），loader 中的 this 即为 loadercontext，绑定了很多 getter，emitFile 可以快速生成 asset（file-loader），pitch 和 normal 可以传递信息，通过 pitch 第三个参数，pitch 通常用作拦截，当 pitch 函数返回值不为 undefined 时。

  当 a.pitch()，b.pitch()均为 undefined

  ​ a.pitch->b.pitch->processResource->b.normal->a.normal

  当 b.pitch()不为 undefined，

  ​ a.pitch->b.pitch->a.normal

  ![image-20191010120234205](./image-20191010120234205.png)

- loadercontext 中有几个比较好用的属性，remainingRequest 不包括当前 loader 路径，data 在 pitch 和 normal 中通信(类似进程通信的共享内存)，resource 当前 module 的文件路径，request 完整路径，dependency 添加 path 依赖，contextdependency 添加 diretory 依赖

  ![image-20191010120107715](./image-20191010120107715.png)

- runloader 有 pitch 从右往左读取 loader，async 方法返回 callback，source,sourcemap

  ![image-20191010115916315](./image-20191010115916315.png)

  ![image-20191010115933123](./image-20191010115933123.png)

  ![image-20191010115953944](./image-20191010115953944.png)

### 事件钩子

- entryOption — 读取入口信息 字符串或数组 对象 函数（动态加载），MultiEntryPlugin,SingleEntryPlugin,DynamicEntryPlugin，静态方法 createDependency（工厂方法），一般在 compilation（可拿到 normalfactory 实例）和 make 阶段（编译开始），constructor 作为 map 的 key，在 build 阶段会用 dep.constructor 从 map 中取值，compilation.dependencyFactories

  ![image-20191010113715944](./image-20191010113715944.png)![image-20191010113742752](./image-20191010113742752.png)

  ![image-20191010113806448](./image-20191010113806448.png)

  ![image-20191010113839340](./image-20191010113839340.png)

- succeedEntry — 添加成功

- afterPlugins — 系统 plugin 注册完毕

- environment

- afterEnvironment — 环境配置完成

- beforeRun

- run

  ![image-20191010121235073](./image-20191010121235073.png)

- readRecord

- afterResolve

- normalModuleFactory — 根据 enhance-resolve，context 路径，module 属性，生成 normalModuleFactory 实例，这时已能拿到所有 loader 信息，

  生成实例的时候会根据 module.rules 生成 ruleSet 实例，同时 normalModuleFactory 绑定了一系列 hooks，触发顺序 beforeresolve，factory，resolver，createParser，parser，afterresolve，createModule，module，XXXFactory 工厂类都会带 create 方法，这里是根据 module 分析依赖，这是 normalmodule 工厂类，create 调用结束后能拿到 module 实例，未打包

  ![image-20191007162615649](./image-20191007162615649.png)

  ![image-20191007163047276](./image-20191007163047276.png)

  ![image-20191007163637647](./image-20191007163637647.png)

- contextModuleFactory — 根据 enhance-resolve，生成 contextModuleFactory 实例

- beforeCompile — 能拿到所有处理完的编译参数，异步

- compile — 能拿到所有处理完的编译参数，同步

  ![image-20191010121520891](./image-20191010121520891.png)

- thisCompilation - 根据 compiler 实例拿到 compilation 实例和编译参数 params，监听不会被复制给 child compiler

- compilation - 根据 compiler 实例拿到 compilation 实例和编译参数 params，即 nmf 实例

  ![image-20191010121456502](./image-20191010121456502.png)

-

* make - 编译开始，根据入口 entry 模块进行解析，

  - addEntry — 添加入口文件

    ![image-20191007165901487](./image-20191007165901487.png)

  - buildModule — module 各项信息完备

  - normalModuleLoader — dobuild 中 createLoaderContext 中，可以扩展 loadercontext

  - succeedModule — 成功

  - failModule — 失败

  - finishModules — module 生成结束

  - finish

  - seal — 开始进行组装

  - optimizeDependenciesBasic

  - optimizeDependencies

  - optimizeDependenciesAdvanced

  - afterOptimizeDependencies

  - beforeChunks

  - afterChunks — chunk，chunkGroup 生成

  - optimize

  - optimizeModulesBasic

  - optimizeModules

  - optimizeModulesAdvanced

  - afterOptimizeModules

  - optimizeChunksBasic

  - optimizeChunks

  - optimizeChunksAdvanced

  - afterOptimizeChunks

  - optimizeTree

  - afterOptimizeTree

  - optimizeChunkModulesBasic

  - optimizeChunkModules

  - optimizeChunkModulesAdvanced

  - afterOptimizeChunkModules

  - shouldRecord

  - reviveModules

  - optimizeModuleOrder

  - advancedOptimizeModuleOrder

  - beforeModuleIds

  - moduleIds — 生成唯一 moduleId

  - optimizeModuleIds

  - reviveChunks

  - optimizeChunkOrder

  - beforeChunkIds

  - optimizeChunkIds

  - afterOptimizeChunkIds

  - recordModules

  - recordChunks

  - beforeHash — 触发 createHash 函数

  - chunkHash — 生成 chunkhash

  - contentHash — 生成 contentHash

  - afterHash — createHash 执行完毕

  - recordHash

  - beforeModuleAssets

  - moduleAsset — 可获取 assets 信息

  - shouldGenerateChunkAssets

  - beforeChunkAssets

  - chunkAsset — 可获取 asset 信息

  - additionalChunkAssets

  - record — 根据 shouldRecord 结果触发

  - additionalAssets

  - optimizeChunkAssets

  - afterOptimizeChunkAssets

  - optimizeAssets

  - afterOptimizeAssets

  - needAdditionalSeal

  - afterSeal — chunk 生成结束，待生成的本地文件存在 asset 中，可通过 getAssets 获取

  - childCompiler — 子编译器已经生成，可以获取子编译器对象，由 compilation 创建

* afterCompile — 在 seal 封装之后触发，表示编译结束

* shouldEmit

* emit — 修改 compliation.asset 可改变最终生成的文件

  ![image-20191010121027741](./image-20191010121027741.png)

* assetEmitted — 每次成功生成本地文件都会触发

  ![image-20191010121053239](./image-20191010121053239.png)

* needAdditionalPass — 是否重新 compile

* done — 编译完成

* additionalPass

  ![image-20191010121153780](./image-20191010121153780.png)

### Watch 监听

- watchRun 钩子

- NodeEnvironmentPlugin 设置 input、output 和 watch，

  ![image-20191010185816510](./image-20191010185816510.png)

- NodeWatchFileSystem 实例化后有一个 watcher 属性，用户设置的 watchoption 都会用来生成这个 watchpack 实例

  ![image-20191010185951594](./image-20191010185951594.png)

- compile.watch 初始化部分信息 -》调用 Watching -》\_go 方法 -〉 \_done-》调用 this.watch 观测 fileDependencies（文件 file），contextDependencies（路径 dir），missingDependencies (不观测的依赖)不断循环-》this.watch 调用 watchFileSystem.watch，即 NodeWatchFileSystem-》调用 watchpack.watch，把 watch 中的 ignore 命中的 filter 后，用 watcherManager 对每一个 file 和 dir 进行观测，改变了会 emitchange 一层一层向上传，能拿到 mtime 修改时间-》实际是 Watcher，核心是借助了 fs.watch 绑上 change 事件, fs.stat 读取时间信息，当路径变化时进行更新，触发 onWatchEvent 事件

- compile.watch->Watching.watch->NodeWatchFileSystem.watch-》Watchpack.watch->DirectoryWatcher.watch

  compile

  ![image-20191010190118929](./image-20191010190118929.png)

  Watching

  ![image-20191010190207721](./image-20191010190207721.png)

  ![image-20191010190232773](./image-20191010190232773.png)

  ![image-20191010190419625](./image-20191010190419625.png)

  ![image-20191010190444000](./image-20191010190444000.png)

  NodeWatchFileSystem

  ![image-20191010190723005](./image-20191010190723005.png)

  ![image-20191010190735926](./image-20191010190735926.png)

  Watchpack

  ![image-20191010191423540](./image-20191010191423540.png)

  ![image-20191010191407340](./image-20191010191407340.png)

  ![image-20191010191506426](./image-20191010191506426.png)

  WatcherManager directoryWatch 按路径生成

  ![image-20191010191603030](./image-20191010191603030.png)

![image-20191010193356024](./image-20191010193356024.png)

Fs.readdir 轮询

![image-20191010194744494](./image-20191010194744494.png)

fs.watch 监听路径变动

![image-20191010193506049](./image-20191010193506049.png)

fs.lstat 获取变更时间

![image-20191010194200914](./image-20191010194200914.png)

![image-20191010194136967](./image-20191010194136967.png)

更新时间

![image-20191010194431795](./image-20191010194431795.png)

#### fs.lstat 和 fs.stat 的区别 :

当 `path` 为实际路径时，两者查看的文件或文件系统状态是一样的，但是当 `path` 为符号链接路径时，`fs.stat` 查看的是符号链接所指向实际路径的文件或文件系统状态，而 `fs.lstat` 查看的是符号链接路径的文件或文件系统状态 。

Watcher 按文件生成，保存文件信息

![image-20191010192046058](./image-20191010192046058.png)

### 官方 Plugin

- EntryOptionPlugin— 根据 entry 的值获取入口，调用相应的 MultiEntryPlugin 或 SingleEntryPlugin 或 DynamicEntryPlugin 进行入口的处理，转换成 normalmodule

- HashedModuleIdsPlugin — 在 compilation 阶段，绑定 beforeModuleIds，这里能获取完整的 module 信息，根据 context 的值生成 hash，createHash 可以传入参数，设置 hash 算法，hash 转换，hash 取值长度等，如果重复则会增加一位长度直至值唯一为止，moduleId 钩子可以获取到 module 的 hash 信息，存在 module.id 中

  ![image-20191010143047394](./image-20191010143047394.png)

- IgnorePlugin — 在 normalModuleFactory 和 contextModuleFactory 中绑定获取 nmr 和 cmr，这里是属于编译前的模块设置，所以放在 normalModuleFactory 初始化完成时候进行而不是编译（compilation）时进行，绑定的钩子时 nmr 和 cmr 的 beforesolve 时，即解析前进行设置，beforesolve 如果返回了 null，则模块被忽略，返回模块则进入打包，参数可以是正则，直接对文件生效，可以是多个参数，分别作用于文件和路径，还可以是函数名称是 checkResource，checkContext

  ![image-20191010143148122](./image-20191010143148122.png)

- NamedModulesPlugin — 与 HashedModuleIdsPlugin 类似，但是能看到路径信息

- NamedChunksPlugin — beforeChunkIds 时用 chunk.name 作为 id

### **第三方 Plugin**

#### speed-mesure-plugin

使用 proxy 进行劫持，注入 addtimeevent，根据 plugin、loader 名字，或者触发的阶段作为 key，收集时间数据 start、end，最后计算最大最小值得出花费时间，loadercontext 中注入属性进行 plugin 和 loader 的数据共享

![image-20191007142552705](./image-20191007142552705.png)

![image-20191007142718271](./image-20191007142718271.png)

![image-20191007142748615](./image-20191007142748615.png)

![image-20191007143145092](./image-20191007143145092.png)

![image-20191007150726194](./image-20191007150726194.png)

![image-20191007145955276](./image-20191007145955276.png)

pluginname：plugin.constructor.name，type：construcNamesTowrap 中的一个，method：hooks 的 key

#### mini-css-extract-plugin

设置 cssmodule 继承自 module 和工厂类

![image-20191010174632277](./image-20191010174632277.png)

这里通过 renderManifest 钩子生成文件的渲染信息，createModulekAsset 时触发

![image-20191010145524567](./image-20191010145524567.png)

这里是 renderContentAsset 对 css 源码的处理，返回的是最终代码

![image-20191010145818696](./image-20191010145818696.png)

根据 chunkFileName 的参数设置 hash，[chunkhash:8]、[contenthash:8]等

![image-20191010145927783](./image-20191010145927783.png)

按需加载的处理

![image-20191010182357587](./image-20191010182357587.png)

### **Loader**

#### style-loader

使用了 pitch 阻止在他之后的 loader 执行，然后改造内容，将路径改成 inline，为了后续的 loader 进行处理

![image-20191010174525080](./image-20191010174525080.png)

![image-20191010173133285](./image-20191010173133285.png)

![image-20191010174447437](./image-20191010174447437.png)

![image-20191010174127391](./image-20191010174127391.png)

![image-20191010174148067](./image-20191010174148067.png)

![image-20191010174216834](./image-20191010174216834.png)

![image-20191010174239212](./image-20191010174239212.png)

![image-20191010174337627](./image-20191010174337627.png)

![image-20191010174357629](./image-20191010174357629.png)
