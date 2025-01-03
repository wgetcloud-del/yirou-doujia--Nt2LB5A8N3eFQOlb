

> 我们是[袋鼠云数栈 UED 团队](https://github.com)，致力于打造优秀的一站式数据中台产品。我们始终保持工匠精神，探索前端道路，为社区积累并传播经验价值。


# 前言


对于 ref 的理解，我们一部人还停留在用 ref 获取真实 dom 元素和获取组件层面上，但实际 ref 除了这两项功能之外，在使用上还有很多小技巧。本章我们就一起深入探讨研究一下 React ref 的用法和原理；本章中所有的源码节选来自 16\.8 版本


# 基本概念和使用


此部分将分成两个部分去分析，第一部分是 ref 对象的创建，第二部分是 React 本身对 ref 的处理；两者不要混为一谈，所谓 ref 对象的创建，就是通过 React.createRef 或者 React.useRef 来创建一个 ref 原始对象。而 React 对 ref 处理，主要指的是对于标签中 ref 属性，React 是如何处理以及 React 转发 ref 。下面来仔细介绍一下。


## ref 对象的创建


### 什么是 ref ？


所谓 ref 对象就是用 createRef 或者 useRef 创建出来的对象，一个标准的 ref 对象应该是如下的样子：



```
{
  current: null, // current指向ref对象获取到的实际内容，可以是dom元素，组件实例。
}

```

#### React 提供两种方法创建 ref 对象


#### 类组件 React.createRef



```
class Index extends React.Component{
    constructor(props){
       super(props)
       this.currentDom = React.createRef(null)
    }
    componentDidMount(){
        console.log(this.currentDom)
    }
    render= () => <div ref={ this.currentDom } >ref对象模式获取元素或组件div>
}

```

打印


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153218973-553270919.png)


React.createRef 的底层逻辑很简单。下面一起来看一下：



> react/src/ReactCreateRef.js



```
export function createRef() {
  const refObject = {
    current: null,
  }
  return refObject;
}

```

createRef 一般用于类组件创建 Ref 对象，可以将 Ref 对象绑定在类组件实例上，这样更方便后续操作 Ref。
**注意：不要在函数组件中使用 createRef，否则会造成 Ref 对象内容丢失等情况。**


### 函数组件


函数组件创建 ref ，可以用 hooks 中的 useRef 来达到同样的效果。



```
export default function Index(){
    const currentDom = React.useRef(null)
    React.useEffect(()=>{
        console.log( currentDom.current ) // div
    },[])
    return  <div ref={ currentDom } >ref对象模式获取元素或组件div>
}

```


> react\-reconciler/ReactFiberHooks.js



```
function mountRef(initialValue: T): {current: T} {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  hook.memoizedState = ref;
  return ref;
}

```

useRef 返回一个可变的 ref 对象，其 current 属性被初始化为传入的参数（initialValue）。返回的 ref 对象在组件的整个生命周期内保持不变。
这个 ref 对象只有一个current属性，你把一个东西保存在内，它的地址一直不会变。


#### 拓展和总结


useRef 底层逻辑是和 createRef 差不多，就是 ref 保存位置不相同，类组件有一个实例 instance 能够维护像 ref 这种信息，但是由于函数组件每次更新都是一次新的开始，所有变量重新声明，所以 useRef 不能像 createRef 把 ref 对象直接暴露出去，如果这样每一次函数组件执行就会重新声明 ref，此时 ref 就会随着函数组件执行被重置，这就解释了在函数组件中为什么不能用 createRef 的原因。
为了解决这个问题，hooks 和函数组件对应的 fiber 对象建立起关联，将 useRef 产生的 ref 对象挂到函数组件对应的 fiber 上，函数组件每次执行，只要组件不被销毁，函数组件对应的 fiber 对象一直存在，所以 ref 等信息就会被保存下来。


## react 对 ref 属性的处理\-标记 ref


首先我们先明确一个问题就是 DOM 元素和组件实例必须用 ref 来获取吗？答案肯定是否定的，比如 react 还提供了一个 findDOMNode 方法可以获取 dom 元素，有兴趣的可以私下去了解一下。不过通过 ref 的方式来获取还是最常用的一种方式。


#### 类组件获取 ref 二种方式


因为 ref 属性是字符串的这种方式，react 高版本已经舍弃掉，这里就不再介绍了。


### ref 属性是个函数



```
class Children extends React.Component{  
    render=()=><div>hello,worlddiv>
}
/* TODO: Ref属性是一个函数 */
export default class Index extends React.Component{
    currentDom = null
    currentComponentInstance = null
    componentDidMount(){
        console.log(this.currentDom)
        console.log(this.currentComponentInstance)
    }
    render=()=> <div>
        <div ref={(node)=> this.currentDom = node }  >Ref属性是个函数div>
        <Children ref={(node) => this.currentComponentInstance = node  }  />
    div>
}

```

打印


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153219275-1078553519.png)


当用一个函数来标记 ref 的时候，将作为 callback 形式，等到真实 DOM 创建阶段，执行 callback ，获取的 DOM 元素或组件实例，将以回调函数第一个参数形式传入，所以可以像上述代码片段中，用组件实例下的属性 currentDom 和 currentComponentInstance 来接收真实 DOM 和组件实例。


### ref 属性是一个 ref 对象



```
class Children extends React.Component{  
    render=()=><div>hello,worlddiv>
}
export default class Index extends React.Component{
    currentDom = React.createRef(null)
    currentComponentInstance = React.createRef(null)
    componentDidMount(){
        console.log(this.currentDom)
        console.log(this.currentComponentInstance)
    }
    render=()=> <div>
         <div ref={ this.currentDom }  >Ref对象模式获取元素或组件div>
        <Children ref={ this.currentComponentInstance }  />
   div>
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153219512-1469748627.png)


### 函数组件获取 ref 的方式（useRef）



```
const Children = () => {  
  return <div>hello,worlddiv>
}
const Index = () => {
  const currentDom = React.useRef(null)
  const currentComponentInstance = React.useRef(null)

  React.useEffect(()=>{
    console.log( currentDom ) 
    console.log( currentComponentInstance ) 
  },[])

  return (
    <div>
      <div ref={ currentDom }  >通过useRef获取元素或者组件div>
      <Children ref={ currentComponentInstance }  />
   div>
  )
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153219726-1684372389.png)


# ref 的拓展用法


## 不得不说的 forwardRef


forwardRef 的初衷就是解决 ref 不能跨层级捕获和传递的问题。 forwardRef 接受了父级元素标记的 ref 信息，并把它转发下去，使得子组件可以通过 props 来接受到上一层级或者是更上层级的 ref 。


### 跨层级获取


我们把上面的例子改一下



> 获取子元素的dom



```
const Children = React.forwardRef((props, ref) => {
  return <div ref={ref}>hello,worlddiv>
}) 
const Index = () => {
  const currentDom = React.useRef(null)
  const currentComponentInstance = React.useRef(null)

  React.useEffect(()=>{
    console.log( currentDom ) 
    console.log( currentComponentInstance ) 
  },[])

  return (
    <div>
      <div ref={ currentDom }  >通过useRef获取元素或者组件div>
      <Children ref={ currentComponentInstance }  />
   div>
  )

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153219926-1257554755.png)



> 想要在 GrandFather 组件通过标记 ref ，来获取孙组件 Son 的组件实例。



```
// 孙组件
function Son (props){
  const { grandRef } = props
  return <div>
      <div> i am alien div>
      <span ref={grandRef} >这个是想要获取元素span>
  div>
}
// 父组件
class Father extends React.Component{
  render(){
      return <div>
          <Son grandRef={this.props.grandRef}  />
      div>
  }
}
const NewFather = React.forwardRef((props,ref)=> <Father grandRef={ref}  {...props} />)
// 爷组件
class GrandFather extends React.Component{
  node = null 
  componentDidMount(){
      console.log(this.node) // span #text 这个是想要获取元素
  }
  render(){
      return <div>
          <NewFather ref={(node)=> this.node = node } />
      div>
  }
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153220317-640189062.png)


### 合并转发ref


通过 forwardRef 转发的 ref 不要理解为只能用来直接获取组件实例，DOM 元素，也可以用来传递合并之后的自定义的 ref



> 场景：想通过 Home 绑定 ref ，来获取子组件 Index 的实例 index ，dom 元素 button ，以及孙组件 Form 的实例



```
// 表单组件
class Form extends React.Component{
  render(){
     return <div>{...}div>
  }
}
// index 组件
class Index extends React.Component{ 
  componentDidMount(){
      const { forwardRef } = this.props
      forwardRef.current={
          form:this.form,      // 给form组件实例 ，绑定给 ref form属性 
          index:this,          // 给index组件实例 ，绑定给 ref index属性 
          button:this.button,  // 给button dom 元素，绑定给 ref button属性 
      }
  }
  form = null
  button = null
  render(){
      return <div   > 
        <button ref={(button)=> this.button = button }  >点击button>
        <Form  ref={(form) => this.form = form }  />  
    div>
  }
}
const ForwardRefIndex = React.forwardRef(( props,ref )=><Index  {...props} forwardRef={ref}  />)
// home 组件
const Home = () => {
  const ref = useRef(null)
   useEffect(()=>{
       console.log(ref.current)
   },[])
  return <ForwardRefIndex ref={ref} />
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153220535-1638253880.png)


### 高阶组件转发


如果通过高阶组件包裹一个原始类组件，就会产生一个问题，如果高阶组件 HOC 没有处理 ref ，那么由于高阶组件本身会返回一个新组件，所以当使用 HOC 包装后组件的时候，标记的 ref 会指向 HOC 返回的组件，而并不是 HOC 包裹的原始类组件，为了解决这个问题，forwardRef 可以对 HOC 做一层处理。



```
function HOC(Component){
  class Wrap extends React.Component{
     render(){
        const { forwardedRef ,...otherprops  } = this.props
        return <Component ref={forwardedRef}  {...otherprops}  />
     }
  }
  return  React.forwardRef((props,ref)=> <Wrap forwardedRef={ref} {...props} /> ) 
}

class Index1 extends React.Component{
  state={
    name: '222'
  }
  render(){
    return <div>hello,worlddiv>
  }
}

const HocIndex =  HOC(Index1)

const AppIndex = ()=>{
  const node = useRef(null)
  useEffect(()=>{
    console.log(node.current)  /* Index 组件实例  */ 
  },[])
  return <div><HocIndex ref={node}  />div>
}

```


> 源码位置 react/src/forwardRef.js



```
export default function forwardRef<Props, ElementType: React$ElementType>(
  render: (props: Props, ref: React$Ref) => React$Node,
) {
  return {
    $$typeof: REACT_FORWARD_REF_TYPE,
    render,
  };
}

```

## ref 实现组件通信


如果有种场景不想通过父组件 render 改变 props 的方式，来触发子组件的更新，也就是子组件通过 state 单独管理数据层，针对这种情况父组件可以通过 ref 模式标记子组件实例，从而操纵子组件方法，这种情况通常发生在一些数据层托管的组件上，比如  表单，经典案例可以参考 antd 里面的 form 表单，暴露出对外的 resetFields ， setFieldsValue 等接口，可以通过表单实例调用这些 API 。



```
/* 子组件 */
class Son extends React.PureComponent{
  state={
     fatherMes:'',
     sonMes:'我是子组件'
  }

  fatherSay=(fatherMes)=> this.setState({ fatherMes  }) /* 提供给父组件的API */
  
  render(){
      const { fatherMes, sonMes } = this.state
      return <div className="sonbox" >
          <p>父组件对我说：{ fatherMes }p>
          <button className="searchbtn" onClick={ ()=> this.props.toFather(sonMes) }  >to fatherbutton>
      div>
  }
}
/* 父组件 */
function Father(){
  const [ sonMes , setSonMes ] = React.useState('') 
  const sonInstance = React.useRef(null) /* 用来获取子组件实例 */

  const toSon =()=> sonInstance.current.fatherSay('我是父组件') /* 调用子组件实例方法，改变子组件state */
  
  return <div className="box" >
      <div className="title" >父组件div>
      <p>子组件对我说：{ sonMes }p>
      <button className="searchbtn"  onClick={toSon}  >to sonbutton>
      <Son ref={sonInstance} toFather={setSonMes} />
  div>
}

```

子组件暴露方法 fatherSay 供父组件使用，父组件通过调用方法可以设置子组件展示内容。
父组件提供给子组件 toFather，子组件调用，改变父组件展示内容，实现父 \<\-\> 子 双向通信。


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153220755-1687901417.png)


### 


函数组件 forwardRef \+ useImperativeHandle


对于函数组件，本身是没有实例的，但是 React Hooks 提供了，useImperativeHandle 一方面第一个参数接受父组件传递的 ref 对象，另一方面第二个参数是一个函数，函数返回值，作为 ref 对象获取的内容。一起看一下 useImperativeHandle 的基本使用。


useImperativeHandle 接受三个参数：


第一个参数 ref : 接受 forWardRef 传递过来的 ref 。
第二个参数 createHandle ：处理函数，返回值作为暴露给父组件的 ref 对象。
第三个参数 deps : 依赖项 deps，依赖项更改形成新的 ref 对象。



```
const Son = React.forwardRef((props, ref) => {
  const state = {
      sonMes:'我是子组件'
  }
  const [fatherMes, setFatherMes] = React.useState('')
    useImperativeHandle(ref,()=>{
      const handleRefs = {
        fatherSay(fatherMss){            
          setFatherMes(fatherMss)
        }
      }
      return handleRefs
  },[])
  const { sonMes } = state
  return (
    <div>
      <p>父组件对我说： {fatherMes}p>
      <button  onClick={ ()=> props.toFather(sonMes) }  >to fatherbutton>
    div>
  ) 
})
/* 父组件 */
function Father(){
  const [ sonMes , setSonMes ] = React.useState('') 
  const sonInstance = React.useRef(null) /* 用来获取子组件实例 */

  const toSon = () => {
    sonInstance.current.fatherSay('我是父组件')
  }
  return <div className="box" >
      <div className="title" >父组件div>
      <p>子组件对我说：{ sonMes }p>
      <button className="searchbtn"  onClick={toSon}  >to sonbutton>
      <Son ref={sonInstance} toFather={setSonMes} />
  div>
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153220936-1265549623.png)


forwardRef \+ useImperativeHandle 可以完全让函数组件也能流畅的使用 Ref 通信。其原理图如下所示：


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153221781-459678193.png)


## 函数组件缓存数据


函数组件每一次 render ，函数上下文会重新执行，那么有一种情况就是，在执行一些事件方法改变数据或者保存新数据的时候，有没有必要更新视图，有没有必要把数据放到 state 中。如果视图层更新不依赖想要改变的数据，那么 state 改变带来的更新效果就是多余的。这时候更新无疑是一种性能上的浪费。


这种情况下，useRef 就派上用场了，上面讲到过，useRef 可以创建出一个 ref 原始对象，只要组件没有销毁，ref 对象就一直存在，那么完全可以把一些不依赖于视图更新的数据储存到 ref 对象中。这样做的好处有两个：


第一个能够直接修改数据，不会造成函数组件冗余的更新作用。
第二个 useRef 保存数据，如果有 useEffect ，useMemo 引用 ref 对象中的数据，无须将 ref 对象添加成 dep 依赖项，因为 useRef 始终指向一个内存空间，所以这样一点好处是可以随时访问到变化后的值。



> 清除定时器



```
const App = () => {
  const [count, setCount] = useState(0)
  let timer;

  useEffect(() => {
    timer = setInterval(() => {
      console.log('触发了');
    }, 1000);
  },[]);

  const clearTimer = () => {
    clearInterval(timer);
  }

  return (
    <>
     <button onClick={() => {setCount(count + 1)}}>点了{count}次button>
      <button onClick={clearTimer}>停止button>
    )
}

```

但是上面这个写法有个巨大的问题，如果这个 App 组件里有state变化或者他的父组件重新 render 等原因导致这个 App 组件重新 render 的时候，我们会发现，点击按钮停止，定时器依然会不断的在控制台打印，定时器清除事件无效了。
为什么呢？因为组件重新渲染之后，这里的 timer 以及 clearTimer 方法都会重新创建，timer 已经不是定时器的变量了。
所以对于定时器，我们都会使用 useRef 来定义变量。



```
const App = () => {
  const [count, setCount] = useState(0)
  const timer = useRef();

  useEffect(() => {
    timer.current = setInterval(() => {
      console.log('触发了');
    }, 1000);
  },[]);

  const clearTimer = () => {
    clearInterval(timer.current);
  }

  return (
    <>
      <button onClick={() => {setCount(count + 1)}}>点了{count}次button>
      <button onClick={clearTimer}>停止button>
    )
}

```

# ref 原理探究


对于 ref 标签引用，React 是如何处理的呢？ 接下来先来看看一段 demo 代码



```
class DomRef extends React.Component{
  state={ num:0 }
  node = null
  render(){
      return <div >
          <div ref={(node)=>{
             this.node = node
             console.log('此时的参数是什么：', this.node )
          }}  >ref元素节点div>
          <button onClick={()=> this.setState({ num: this.state.num + 1  }) } >点击button>
      div>
  }
}

```

控制台输出结果


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153222158-1328472624.png)


**提问: 第一次打印为 null ，第二次才是 div ，为什么会这样呢？ 这样的意义又是什么呢？**


## ref 执行的时机和处理逻辑


根据 React 的生命周期可以知道，更新的两个阶段 render 阶段和 commit 阶段，对于整个 ref 的处理，都是在 commit 阶段发生的。之前了解过 commit 阶段会进行真正的 Dom 操作，此时 ref 就是用来获取真实的 DOM 以及组件实例的，所以需要 commit 阶段处理。
但是对于 ref 处理函数，React 底层用两个方法处理：**commitDetachRef** 和 **commitAttachRef** ，上述两次 console.log 一次为 null，一次为 div 就是分别调用了上述的方法。


这两次正好，一次在 DOM 更新之前，一次在 DOM 更新之后。


第一阶段：一次更新中，在 commit 的 mutation 阶段, 执行commitDetachRef，commitDetachRef 会清空之前ref值，使其重置为 null。
源码先来看一下



> react\-reconciler/src/ReactFiberCommitWork.js



```
function commitDetachRef(current: Fiber) {
  const currentRef = current.ref;
  if (currentRef !== null) {
    if (typeof currentRef === 'function') { /* function获取方式。 */
      currentRef(null); 
    } else {   /* Ref对象获取方式 */
      currentRef.current = null;
    }
  }
}

```

第二阶段：DOM 更新阶段，这个阶段会根据不同的 effect 标签，真实的操作 DOM 。
第三阶段：layout 阶段，在更新真实元素节点之后，此时需要更新 ref



> react\-reconciler/src/ReactFiberCommitWork.js



```
function commitAttachRef(finishedWork: Fiber) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    let instanceToUse;
    switch (finishedWork.tag) {
      case HostComponent: //元素节点 获取元素
        instanceToUse = getPublicInstance(instance);
        break;
      default:  // 类组件直接使用实例
        instanceToUse = instance;
    }
    if (typeof ref === 'function') {
      ref(instanceToUse);  //* function 和 字符串获取方式。 */
    } else {
      ref.current = instanceToUse; /* ref对象方式 */
    }
  }
}

```

这一阶段，主要判断 ref 获取的是组件还是 DOM 元素标签，如果 DOM 元素，就会获取更新之后最新的 DOM 元素。上面流程中讲了二种获取 ref 的方式。 如果是 函数式 ref\={(node)\=\> this.node \= node } 会执行 ref 函数，重置新的 ref 。如果是 ref 对象方式，会更新 ref 对象的 current 属性。达到更新 ref 对象的目的。


## ref 的处理特性


![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153222478-2034210329.png)


接下来看一下 ref 的一些特性，首先来看一下，上述没有提及的一个问题，React 被 ref 标记的 fiber，那么每一次 fiber 更新都会调用 commitDetachRef 和 commitAttachRef 更新 Ref 吗 ？
答案是否定的，只有在 ref 更新的时候，才会调用如上方法更新 ref ，究其原因还要从如上两个方法的执行时期说起


## 更新 ref


在 commit 阶段 commitDetachRef 和 commitAttachRef 是在什么条件下被执行的呢 ？ 来一起看一下：
commitDetachRef 调用时机



> react\-reconciler/src/ReactFiberWorkLoop.js



```
function commitMutationEffects(){
     if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }
}

```

commitAttachRef 调用时机



```
function commitLayoutEffects(){
     if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }
}

```

从上可以清晰的看到只有含有 ref tag 的时候，才会执行更新 ref，那么是每一次更新都会打 ref tag 吗？ 跟着我的思路往下看，什么时候标记的 ref .



> react\-reconciler/src/ReactFiberBeginWork.js



```
function markRef(current: Fiber | null, workInProgress: Fiber) {
  const ref = workInProgress.ref;
  if (
    (current === null && ref !== null) ||      // 初始化的时候
    (current !== null && current.ref !== ref)  // ref 指向发生改变
  ) {
    workInProgress.effectTag |= Ref;
  }
}

```

首先 markRef 方法执行在两种情况下：
第一种就是类组件的更新过程
第二种就是更新 HostComponent 的时候
markRef 会在以下两种情况下给 effectTag 标记 ref，只有标记了 ref tag 才会有后续的 commitAttachRef 和 commitDetachRef 流程。（ current 为当前调和的 fiber 节点 ）


第一种 current \=\=\= null \&\& ref !\=\= null：就是在 fiber 初始化的时候，第一次 ref 处理的时候，是一定要标记 ref 的。
第二种 current !\=\= null \&\& current.ref !\=\= ref：就是 fiber 更新的时候，但是 ref 对象的指向变了。


所以回到最初的那个 DomRef 组件，为什么每一次按钮，都会打印 ref ，那么也就是 ref 的回调函数执行了，ref 更新了。每一次更新的时候，都给 ref 赋值了新的函数，那么 markRef 中就会判断成 current.ref !\=\= ref，所以就会重新打 Ref 标签，那么在 commit 阶段，就会更新 ref 执行 ref 回调函数了。
要想解决这个问题，我们把 DomRef 组件做下修改：



```
 class DomRef extends React.Component{
    state={ num:0 }
    node = null
    getDom= (node)=>{
        this.node = node
        console.log('此时的参数是什么：', this.node )
     }
    render(){
        return <div >
            <div ref={this.getDom}>ref元素节点div>
            <button onClick={()=> this.setState({ num: this.state.num + 1  })} >点击button>
        div>
    }
}

```

![file](https://img2024.cnblogs.com/other/2332333/202412/2332333-20241231153222711-2023316692.png)



```
function DomRef () {
  const [num, setNum] = useState(0)
  const node = useRef(null)
      return (
      <div >
          <div ref={node}>ref元素节点div>
          <button onClick={()=> {
            setNum(num + 1)
            console.log(node)
          }} >点击button>
      div>
    )
}

```

## 卸载 ref


上述讲了 ref 更新阶段的特点，接下来分析一下当组件或者元素卸载的时候，ref 的处理逻辑是怎么样的。



> react\-reconciler/src/ReactFiberCommitWork.js



```
function safelyDetachRef(current) {
  const ref = current.ref;
  if (ref !== null) {
    if (typeof ref === 'function') {  // 函数式 
        ref(null)
    } else {
      ref.current = null;  // ref 对象
    }
  }
}

```

被卸载的 fiber 会被打成 Deletion effect tag ，然后在 commit 阶段会进行 commitDeletion 流程。对于有 ref 标记的 ClassComponent （类组件） 和 HostComponent （元素），会统一走 safelyDetachRef 流程，这个方法就是用来卸载 ref。


# 总结


* ref 对象的二种创建方式
* 两种获取 ref 方法
* 介绍了一下forwardRef 用法
* ref 组件通信\-函数组件和类组件两种方式
* useRef 缓存数据的用法
* ref 的处理逻辑原理


希望本次的分享能给大家带去一点收货；


## 最后


欢迎关注【袋鼠云数栈UED团队】\~
袋鼠云数栈 UED 团队持续为广大开发者分享技术成果，相继参与开源了欢迎 star


* **[大数据分布式任务调度系统——Taier](https://github.com)**
* **[轻量级的 Web IDE UI 框架——Molecule](https://github.com)**
* **[针对大数据领域的 SQL Parser 项目——dt\-sql\-parser](https://github.com):[veee加速器官网](https://veee6.com/)**
* **[袋鼠云数栈前端团队代码评审工程实践文档——code\-review\-practices](https://github.com)**
* **[一个速度更快、配置更灵活、使用更简单的模块打包器——ko](https://github.com)**
* **[一个针对 antd 的组件测试工具库——ant\-design\-testing](https://github.com)**


