---
title: React用法解析
date: 2018-09-11 16:58:26
tags:
---
#### dva是什么

dva是对 redux + react-router + redux-saga 等的一层封装，提供了一个models，常用的属性如下：namespace，state，effects，reducers，subscriptions。

models维护了一个全局的状态，models中可以通过namespace的属性对各个状态进行隔离，相当于划分了一个作用域，然后通过@connect可以将state注入到需要用到的组件中，组件可以通过自身的props访问到这个state，

如果需要改变state，就可以通过effects和reducers，前者是一个generator，用来处理异步逻辑，后者则是处理同步逻辑，在组件中可以通过使用dispatch函数去触发effects或reducers，dispatch函数会返回一个promise，因此我们可以在reducers或者effects中通过return将数据直接返回给dispatch，而subscriptions则是可以订阅路由的变化。
<!-- more -->
```javascript
namespace:'A',
state:{
        
},
effects:{
    *B(state,{call,put}){
        //其中的call则是用来触发services中的函数去请求数据，用法类似于js中的call
        //const a = yield call(X,参数)
        //put则是用来触发reducers更改state的状态
        //const action = {type:'B/XX',payload}
        //yield put(action) 
    }
},
reducers:{
    c(state,action){
        //处理逻辑
        return{
            ...state,
            XXX:action.payload,//XXX是需要改变的state，习惯上payload是action中的传过来的新state，可以自定义
        }
    }       
},
subscriptions: {
    //这里可以传入dispatch或者history等
    setup({ dispatch,history }) {\
      //可以进行history的监听，当路由变动时可以执行回调
      return history.listen(location) => {
       //回调的代码
       //也可以用dispatch触发reducers更改state状态
       //如果reducers是相同namespace下的则不需要添加前缀，如果不同则需要添加
       //const action = {type:'B/XX',payload}
       //例如有一个namespace是B的，则需要dispatch(action)
       //其中payload是需要最新的state
      });
    },
  },
```

由此我们可以实现一个MVC的结构，services文件负责管理向后端的接口请求，每个请求都需要用到fetch，那么对request可以进行简单的封装，models文件负责数据的储存和管理，然后components和routes文件则是用于渲染的组件，

```javascript
|--src
  |--routes //页面路由渲染组件
  |--components//公用组件
  |--services//请求接口
  |--models//状态管理和数据储存
```



### redux-saga

redux是react的全局状态管理的实现，它接收组件dispatch传来的type和payload，然后对type命中之后，用payload中的值更新redux中的state，这解决了react自身的跨层级和兄弟组件之间通信不方便的问题，也降低了组件之间的耦合度，在未使用redux的时候，react兄弟组件A，B间的通信需要借助父组件作为媒介，A将需要传递的信息通过props中的回调函数传递到父组件，父组件再通过props传递给B组件，而跨层级的通信则是需要通过层层props传递，或者使用contextAPI传递，这些都增加了组件之间的耦合度。

![](C:\Users\meijintao\Desktop\post5_1.jpg)

---

#### roadhog

roadhog是一个react的脚手架，它定制了一套配置，提供了一些字段，方便给开发者使用。使用的时候只需在.hoadloadrc文件中设置即可，在源码中还有如下逻辑，如果设置了roadhog.conifg.js，那么它将会以roadhog.conifg.js中的配置为准，在读取了配置之后，它会去调用config文件夹下的webpack.config.js中的逻辑。

虽然roadhog高度定制了一套配置，减少了开发者的配置时间，但是也是由此，而不方便针对项目进行调整。在roadhog1.X中，没有提供别名alias和公共包分离的功能，如果支持别名，则可以对公共组件的引用使用绝对地址，方便应对将来文件结构的变动的影响，公共包分离的功能虽然提供mulitpage字段，但是从源码中看到这个是默认去调用commonchulksplugin并输出到common文件中，但是得手动设置entry和将需要的插件分离，而且不便于将webpack打包的运行时分离，会导致每次打包的hash值都会改变，无法起到缓存作用。

这两个功能在2.X版本中进行了支持，分别提供了alias和common字段，在1.X版本的话就需要用到babel-transform-resolver提供别名，分包的话在github上作者提供了一个enhancement，在打包的时候加上--config参数，值是自己的webpack配置文件。

```json
{
    "entry":""//入口文件
    "output":""//打包输出路径，默认为./dist
    "env":{}//设置开发环境和线上环境的配置
	"hash":false,//是否对打包出来的文件添加hash内容，默认false
	"proxy":{},//设置请求代理
	"muiltpage":false，//多页面，开启可以进行分离公共包，默认false
	"extraBabelPlugins":[]//引入babel
}
```

---

#### react-router

单页面应用都需要借助路由实现进行页面内的无刷新跳转，路由主要是依靠在route组件中传入path属性，path属性是一个路径，如果浏览器的url中path命中，那么就会将该route的组件进行渲染，因此还可以传入component属性，值为需要渲染的组件，使用getComponent函数则可以实现路由的按需加载，dva提供了dynamic函数进行按需加载，使用的route组件需要包裹在router组件中

```javascript
<Router>
	<Route path="/a" component={}></Route>
    <Route path="/a" getComponent={fn(component,callback)}></Route> //按需加载
</Router>
```

路由跳转则依靠Link组件，可以传入to或replace属性，route组件会被渲染成a标签，to和replace属性都可以传入一个location对象，其属性是pathname，state，search，hash，区别是to相当于pushState，往浏览器的路由栈中压入路径，replace相当于替换当前页面的路径。

```javascript
<Link to={location}></Link>
```

redirect组件则是当path命中时，to到指定路径，也即是实现了重定向的功能。

```javascript
<Redirect path="/a" to="/abc"></Redirect> // 路径/a会跳转到/abc
```

在被渲染的组件中，我们可以通过props访问到location属性和match对象属性，match对象中用的比较多的是params，它是一个对象，存放着pathname中的变量，例如```/a/b/:xx的xx```。我们也可以在组件中通过props访问history对象，它里面封装了一些方法，和window.history内的方法效果一致。

```javascript
location = {
    pathname,//这个的值是不包括域名的路径
    state,//如果页面刷新，state的数据会丢失，正常情况state会随着路由跳转可以在下一个页面的location中取到，类似post的body数据，不会出现在浏览器url中
    hash,
    search, //search的值'?XXX=aaa&XX=bbb'，需要自己进行解析
}
match:{
    params:{} // 假设/a/b/c/:d/e/:f,那么params中的值则为d，f的键值对
    url, //和浏览器中的url一致
    path,//和route中写的path一致
}
```



### react

传统的JS框架都是以操作dom为主，由于dom十分臃肿，创建开销大，频繁操作更是会引起重回和回流，这些都会对性能造成影响。

![](C:\Users\meijintao\Desktop\post5_2.png)

react则提出了虚拟dom，解决了这个问题，虚拟dom是一个JS对象，它包含了tagName，props，children

```javascript
const virtualDom = {
    tagName: 'div', //组件命名
    props:{}, //组件的属性
    children:[],//组件中的子元素
}
```

虚拟dom保留了主要的信息，利用这些信息我们可以构建出一个虚拟dom树结构，这个虚拟dom树相当于是真实dom的一个映射，消耗的资源也是极少的。有了这个映射，那么我们的所有逻辑都无需直接去操作真实dom了，直接操作虚拟dom然后再一次性的映射到真实dom上改变真实dom。

这里我们就需要一个diff算法去对比虚拟dom和真实dom，找出被更改的元素，状态，属性等，原本对比两棵树的时间复杂度是O(n3)，这在实际操作中是不能接受的，而react就对diff算法进行了改进，把diff算法的复杂度降到了O(n)。

---

#### diff策略

这是建立在三个基本假定之上的diff策略：

- dom节点跨层级的移动操作特别少，可以忽略不计

- 拥有相同类的两个组件将会生成相似的树形结构，拥有不同类的两个组件将会生成不同的树形结构

- 对于同一层级的一组子节点，它们可以通过唯一 key 进行区分

在进行diff算法的时候，如果两个组件不同，那么就不会继续往下比对而是直接将其销毁，并生成新的component，如果组件相同就会对state和props进行比对，state和props不同则直接更新组件的state和props，这里需要注意一点，子组件是无法变更props的，除非是通过子向父传递信息的方式调用父组件的setState方法，去改变父组件的state进而造成子组件的props发生变化，除此之外，react还会通过唯一的key对组件进行区分

![](C:\Users\meijintao\Desktop\post5_3.jpg)

对于这种情况，react只会移动组件，而不是销毁重新创建新组件，这往往对列表的渲染影响比较大，如果列表渲染时不带key，那么如果对列表按某一列进行排序，那么react会对这些数据重新销毁重建，而实际情况是这些组件只是改变了位置。

```javascript
//还有一种情况如下，在渲染列表的时候，从头部插入数据和从尾部插入数据也会有不同的结果，
//react会对每一项进行比对，从头部插入，那么每一项的对比都会认为是不同的数据，从尾部插入，那么前三项则是相同的，只是最后一项是不同的数据
[1,2,3] 
[4,1,2,3]
[1,2,3,4]
```

---

#### react的生命周期

ES5：

```javascript
getDefaultProps(){} //获得props
getInitialState(){}	//初始化state
componentWillMount(){} //虚拟dom即将挂载，在这里调用setState不会触发重新渲染，还有一点需要注意的是未成功挂载之前，是无法获得component的ref的
render(){}//视图渲染
componentDidMount(){}//挂载完成，这个阶段只会执行一次，重渲染时不会执行，一般会在这个阶段请求数据，在这里需要注意的是，如果利用props的值时有可能props还未返回，所以需要进行容错性处理

componentWillRecieveProps(nextProps){}//props值变化时会触发重新渲染，这里使用setState不会触发重新渲染
shouldComponentUpdate(nextProps,nextState)(){}//如果return false则不会进行重新渲染，否则会重新渲染，因此我们可以在这里进行一些逻辑判断避免多余的重新渲染
componentWillUpdate(){}
render(){}//视图渲染，不要在这里setState，会导致死循环
componentDidUpdate(){}//不要在这里setState，会导致死循环

componentWillUnmount(){} //组件销毁前的事件钩子，一般用来销毁定时器和移除自己用addEventListener的事件监听，避免组件销毁后造成内存泄漏
```

ES6：

```javascript
Class Acomponent extends Component{
    //getDefaultProps和getInitialState放在这里进行
    constructor(props){ 
        super(props);
       	this.state = {};
        //在ES6中我们可以用箭头函数代替ES5中的Afun.bind(this)转换this的指向，因为在react的合成事件中的this会指向触发该事件的元素
        //或者我们也可以在这里调用bind，因为可以使bind只是调用一次，如果在合成事件例如onClink={Afun.bind(this)}会导致每次点击的时候都会触发bind事件
    }
}
```

对于react的合成事件，react已经将所有的事件都自动挂载在了document上，因此所有的事件都会冒泡到document上进行触发，使用合成事件我们就不需要在组件销毁时对移除事件监听，react自动帮我们进行管理。

---

#### setState

在react中我们如果希望更新视图，那么通常是会通过setState触发状态改变，然后触发react的重新渲染的过程，如果只是对state赋值进行的变更，是不会触发重新渲染的。

```javascript
//setState接受两个参数，setState（fn|Object,callback）
//第一个参数如果是Object，那么setState的处理就是把Object合并，如果是函数，那么就会被放入一个队列中，一次执行。而第二个参数则是状态变更完成后的回调，即能再回调中获取到最新的state
//针对setState，react也进行了一些优化
//setState的执行是"异步"的，加引号是因为这里并不是真正意义上的异步，只是react为了避免频繁的对state进行改动，它的处理方式是将setState的执行内容放到js主线程执行完之后再一次性执行，然后进行一次性进行diff算法

//以下执行了三个setState，由于传入的是对象，那么react的处理是，将这三个对象和state进行合并，类似Object.assign(),然后setState的执行是"异步”的，那么在回调中能获取到a的值为1，可是同步的获取a的值为0
state = {a:0}
this.setState({a:this.state.a+1})
this.setState({a:this.state.a+1})
this.setState({a:this.state.a+1},()=>{console.log(this.state.a) //1})
console.log(this.state.a) //0

//以下执行了三个setState，由于传入的是函数，那么react的处理是，将这三个函数放到一个队列中，依此执行，每个函数中都有一个state参数，这里能拿到最新的state，在回调中能获取到a的值为3
state = {a:0}
this.setState(state=>{state.a++;console.log(state.a))//1
this.setState(state=>{state.a++;console.log(state.a))//2
this.setState(state=>{state.a++;console.log(state.a),//3
                        ()=>{console.log(this.state.a)})//3
```

---

#### 父子之间的传值

父->子：父组件通过props即可向子组件传值

子->父：通过props中传递函数的形式传值

```javascript
class Parent extends Component {
    constructor(props){
        super(props)
        this.state={
            parentval:0
        }
    }
    parentfn(childval){
        console.log(childval)//555
    }
    render(){
        return <Child parentfn={this.parentfn} ></Child>
    }
}
class Child extends Component {
    constructor(props){
        super(props)
        this.state={
           childval:555
        }
    }
    render(){
       return <div onClick={this.props.parentfn(this.state.childval)}></div>
    }
}
```

---

#### React v16

react v16提供了一些新的特性，例如：

- 利用React.Fragment作为render中return的标签或者组件的根元素，而这个Fragment在渲染的时候不会被渲染出来，而在v16之前我们一般用div作为根元素进行包裹
- render中return可以返回一个标签数组['<div></div>','<div></div>','<div></div>']
- 提供了createRef去获取组件的ref

```javascript
//v16
class A extends Component {
    constructor(props){
        super(props)
        this.state={}
        this.input = React.createRef();
    }
    render(){
        return <input ref={this.input}/>
    }
}
//V16之前
class A extends Component {
    constructor(props){
        super(props)
        this.state={}
    }
    render(){
        return <input ref={(input)=>{this.input=input}}/>
    }
}
```

- 虚拟dom的结构不是树而是变成了链表，提出了fiber模式，过去如果我们需要渲染大量结点的时候，由于diff算法的比对是不能中断的，占用了js主线程时间，表现出来就是页面假死不能对用户交互做出响应，而如今架构换成了链表之后可以随时暂停，那么diff算法就可以拆分在不同的时间片中，根据不同的优先级进行资源的抢占，例如用户交互的优先级高，在时间片中就可以根据优先级对不同情况进行响应。

---

### 总结

- React与Angular相比，它在处理数据量不大的情况时，比angular的脏检查机制更有优势，因为angular需要对所有数据进行比对，可是在处理数据量大时由于每次都需要进行依赖收集，就不如angular有效率。
- React与Vue相比，没有双向绑定，也没有指令，但是通过虚拟dom建立model层和view层的联系，每个组件维护自身的state，更适合组件化的开发模式。
- 在React v16的fiber模式，解决了大量结点渲染造成页面假死的情况，拆分成不同时间片中执行diff，体现了化整为零的思想，而这种思想在计算机领域处处可见。
- 目前用了react和antd做了半个月的业务，总结了部分所得：
  - pagination的current属性的值只接受number，要注意赋值之前对变量的处理
  - 因为用了redux，很多时候请求接口获得的数据是通过props注入的，对props要习惯进行容错性处理，由于props变动会导致组件重新渲染，因此需要在shouldComponentUpdate中进行逻辑判断，避免无意义的重新渲染，也要注意对setState的使用
  - 在TimePicker组件中，onchage回调会返回一个表示TimePicker的开闭情况的参数，根据返回的参数直接setState可以实现点击非TimePicker区域关闭TimePicker，如果自己更改了会导致该功能失效
  - 组件的ref属性会在组件挂载之后才能拿到，未挂载之前获取的值是undefined
  - redux中的state会在浏览器刷新的时候被清掉，Route组件中to属性传的location中的state也会被清掉，因此可以对一些数据利用sessionStroage进行保存，或者将数据存服务器，在刷新的时候从服务器获取，或者记录在浏览器的URL中，在做面包屑功能的时候我就记录在浏览器URL中，因为有一些页面可以从两个不同的页面跳转，只是建立路由和名字的映射表记录跳转无法解决刷新清除state的问题，需要根据URL上记录的from信息，再根据映射表将祖先路由找回来