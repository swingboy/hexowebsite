---
title: 简单梳理react优化相关
date: 2020-01-24 14:56:12
tags: 技术
---

我们知道关于react可以找到很多的最佳实践。这里从代码层面和一些业务层面简单梳理下
- 关于优化或者是调优一定是通过减少不必要的re-render和计算来提高性能。这些优化旨在提高维护性、可读性、扩展性。

## 1.从代码层面：

*首先，可能导致react重新render的方式有三种*

* setState
* new props
* forceUpdate


### 使用shouldComponentUpdate

一般情况
```
import React, { Component } from 'react';
class SomeComp extends Component{
  ...
}
```

React 做性能优化时有一个避免重复渲染的大招，就是使用 shouldComponentUpdate，但它默认返回 true，即始终会执行 render 方法，然后做 Virtual DOM 比较，并得出是否需要做真实 DOM 更新，这里往往会带来很多无必要的渲染并成为性能瓶颈

对于此类写法，每次一定是重新执行render的。所以，你需要用到shouldComponentUpdate这个生命周期函数, 来进行判断是否需要进行渲染。

- 这是在重新渲染组件之前触发的其中一个生命周期事件。
- 这个函数将 nextState 和 nextProps 作为输入，并可将其与当前 props 和state做对比，以决定是否需要重新渲染。如果没有变化，则返回false来阻止更新。

```
shouldComponentUpdate(nextProps, nextState) {
  if(nextState.a != this.state.a || netState.b = this.state.b) {
    return true;
  }
  return false;
}
```

### 使用PureComponent

React.PureComponent 与 React.Component 几乎完全相同，但 React.PureComponent 通过props和state的浅对比来实现 shouldComponentUpate

唯一的区别就是：PureComponent多了个标识isPureReactComponent


```
class SomeComp extends React.PureComponent {
    // Pure Components are the components that do not re-render if the State data or props data is still the same
    render() {
        return <div>{this.props.a}</div>;
    }
}
```

在react中有这么一段代码检查是否更新component

```
function checkShouldComponentUpdate(
  workInProgress,
  ctor,
  oldProps,
  newProps,
  oldState,
  newState,
  nextContext,
) {
  const instance = workInProgress.stateNode;
  if (typeof instance.shouldComponentUpdate === 'function') {
    startPhaseTimer(workInProgress, 'shouldComponentUpdate');
    const shouldUpdate = instance.shouldComponentUpdate(
      newProps,
      newState,
      nextContext,
    );
    stopPhaseTimer();

    return shouldUpdate;
  }

  if (ctor.prototype && ctor.prototype.isPureReactComponent) {
    return (
      !shallowEqual(oldProps, newProps) || !shallowEqual(oldState, newState)
    );
  }

  return true;
}
```

可以看出react中判断更新的几个判断：无论组件是否是 PureComponent，如果定义了 shouldComponentUpdate，那么会调用它并以它的执行结果来判断是否 update。在组件未定义 shouldComponentUpdate 的情况下，会判断该组件是否是 PureComponent，如果是的话，会对新旧 props、state 进行 shallowEqual 比较，一旦新旧不一致，会触发 update。

这里面的shallowEqual只会比较到两个对象的 ownProperty 是否符合 Object.js() 判等，不会递归地去深层比较,如下：

```
function shallowEqual(objA, objB) {
  if (is(objA, objB)) {
    return true;
  }

  if (
    typeof objA !== 'object' ||
    objA === null ||
    typeof objB !== 'object' ||
    objB === null
  ) {
    return false;
  }

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) {
    return false;
  }

  // Test for A's keys different from B.
  for (let i = 0; i < keysA.length; i++) {
    if (
      !hasOwnProperty.call(objB, keysA[i]) ||
      !is(objA[keysA[i]], objB[keysA[i]])
    ) {
      return false;
    }
  }

  return true;
}

function is(x, y) {
  return (
    (x === y && (x !== 0 || 1 / x === 1 / y)) || (x !== x && y !== y)
  );
}
```

### React.memo

React.memo 是一个高阶组件。和 PureComponent 很相似，它也是帮助我们控制何时重新渲染组件

- PureComponent 要依靠 class 才能使用。而 React.memo() 可以和 functional component 一起使用。
- 与纯组件类似，如果输入 props 相同则跳过组件渲染，从而提升组件性能。
- 它会记忆上次某个输入 prop 的执行输出并提升应用性能。即使在这些组件中比较也是浅层的。
- 通过自动shouldComponentUpdate帮助组件执行是否需要update，但也只是执行浅比较，其意义和价值有限。
- 当然，你还可以为这个组件传递自定义比较逻辑。

- 为什么它被称作 memo？在计算机领域，记忆化是一种主要用来提升计算机程序速度的优化技术方案。它将开销较大的函数调用的返回结果存储起来，当同样的输入再次发生时，则返回缓存好的数据，以此提升运算效率。

``` 
let func = React.memo((props) => {
    return <div {...props}>sck</div>
});

 
function compare(previosProps, nextProps) {
    if(previosProps.user.a == nextProps.user.a ||
       previosProps.user.b == nextProps.user.b ||
       ) {
        return false
    } else {
        return true;
    }
}
 
var memoComponent1 = React.memo(func);
var memoComponent2 = React.memo(func, compare);

```

### 使用React.Fragments 避免额外标记

react 中组件一定要有一个唯一的父级元素，使得我们不得不在顶层加一个并不需要的div来确保能够正确被react解析渲染。加深了层级和一些不必要的渲染。

而Fragments 允许你将子列表分组，代码没有额外的标记，而无需向 DOM 添加额外节点。因此节省了渲染器渲染额外元素的工作量。

```
export default class SomeComp extends React.Component {
  render() {
    return (
      <>
          <div>header</div>
          <div>xxxx</div>
      </>
    )
  }
}
```


### 不要使用内联函数定义

如果我们使用内联函数，则每次调用“render”时都会创建一个新的函数实例

当 React 进行虚拟 DOM diffing 时，它每次都会找到一个新的函数实例；因此在渲染阶段它会会绑定新函数并将旧实例扔给垃圾回收。

因此直接绑定内联函数就需要额外做垃圾回收和绑定到 DOM 的新函数的工作。

```
export default class SomeComp extends React.Component {
  render() {
    return (
      <input type="button" onClick={()=> {this.setState({a: 100})}} value="Click" />
    )
  }
}
```

上面的函数创建了内联函数。每次调用 render 函数时都会创建一个函数的新实例，render 函数会将该函数的新实例绑定到该按钮。

此外最后一个函数实例会被垃圾回收，大大增加了 React 应用的工作量。

修改:
```
export default class SomeComp extends React.Component {
  
  setNewData = () => {
    this.setState({
      a: 100
    })
  }
  
  render() {
    return (
      <input type="button" onClick={this.setNewData} value="click" />
    )
  }
}
```

### 在Constructor中绑定函数

当我们在 React 中创建函数时，我们需要使用 bind 关键字将函数绑定到当前上下文。

绑定可以在构造函数中完成，也可以在我们将函数绑定到 DOM 元素的位置上完成。

两者之间似乎没有太大差异，但性能表现是不一样的。

```
export default class Sth extends React.Component {
  constructor() {
  }
  
  handleButtonClick() {
  }
  
  render() {
    return (
      <>
        <input type="button" value="Click" onClick={this.handleButtonClick.bind(this)} />
      </>
    )
  }
}

```
在上面的代码中，我们在 render 函数的绑定期间将函数绑定到按钮上。

上面代码的问题在于，每次调用 render 函数时都会创建并使用绑定到当前上下文的新函数，但在每次渲染时使用已存在的函数效率更高。优化方案如下：

```
export default class Sth extends React.Component {
  constructor() {
    this.handleButtonClick = this.handleButtonClick.bind(this)
  }
  
  handleButtonClick() {
  }
  
  render() {
    return (
      <>
        <input type="button" value="Click" onClick={this.handleButtonClick} />
      </>
    )
  }
}
```
此类情况，最好在构造函数调用期间使用绑定到当前上下文的函数覆盖 handleButtonClick 函数。

这将减少将函数绑定到当前上下文的开销，无需在每次渲染时重新创建函数，从而提高应用的性能。


### 避免使用内联样式属性

使用内联样式时浏览器需要花费更多时间来处理脚本和渲染，因为它必须映射传递给实际 CSS 属性的所有样式规则。

```
	
export default class Sth extends React.Component {
  render() {
    return (
      <>
        <span style={{"backgroundColor": "red"}}>i am text</span>
      </>
    )
  }
}
```

在上面创建的组件中，我们将内联样式加到组件。添加的内联样式是 JavaScript 对象而不是样式标记。

样式 backgroundColor 需要转换为等效的 CSS 样式属性，然后才应用样式。这样就需要额外的脚本处理和 JS 执行工作。

而且，每次render会创建新的样式对象。而且当style作为props时，每次创建新对象会引起子组件的重新渲染。最好是通过导入类减少不必要的对象定义


### 不要在render函数做渲染意外的任何事

render作为纯函数，他的状态的变更只跟state有关。其他不想管的操作不要在此书写。应保持纯净，以确保组件以一致的方式运行和渲染。


### 为组件创建错误边界

```
export default class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    console.log('getDerivedStateFromError:', error)
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    elog.i(T, "componentDidCatch", { error, errorInfo })
  }

  render() {
    if (this.state.hasError) {
      return <h1>Something went wrong</h1>
    }

    return this.props.children;
  }
}

```

- 使用 static getDerivedStateFromError() 渲染备用 UI ，使用 componentDidCatch() 打印错误信息。

- 错误边界仅可以捕获其子组件的错误，它无法捕获其自身的错误。如果一个错误边界无法渲染错误信息，则错误会冒泡至最近的上层错误边界，这也类似于 JavaScript 中 catch {} 的工作机制。


### 避免 componentWillMount 中的异步请求

componentWillMount的 react 16.3开始已经是unsafe了。不推荐使用。新版的react请避免使用这个生命周期函数

在componentWillMount中执行this.setState是不会触发二次渲染的。

它也只会在挂载过程中被调用一次，它的作用和constructor没有太大差异。有很多人在componentWillMount中请求后台数据，认为这样可以更早的得到数据，componentWillMout是在render函数执行前执行的，虽然请求是在第一次render之前发送的，但是返回并不能保证在render之前完成。render不会等你慢慢请求.所以在渲染的时候没有办法等到数据到来再去setState触发二次渲染.

- 仔细思考一下，componentWillMount好像没啥卵用了。其实，在服务端渲染的场景中componentDidMount是不会被执行的，因此可以在componnetWillMount中发生AJAX请求。


### componentWillUpdate、componentDidUpdate不要setState

这个就太基本了，这两个生命周期中不能调用setState，在它们里面调用setState会造成死循环，导致程序崩溃。

### 尽量减少数组下标当做key

为什么我们要使用key？

使用key能够让组件保持结构的稳定性。我们都知道React以其DOM Diff算法而著名，在实际比对节点更新的过程中带有唯一性的key能够让React更快得定位到变更的节点，从而可以做到最小化更新。

在实际使用过程中，很多人常常图方便会直接使用数组的下标(index)作为key，这是很危险的。因为经常会对数组数据进行增删，容易导致下标值不稳定。所以在开发过程中，应该尽量避免这种情况发生。


- 使用唯一id作为key值。如果你的数据项有id并且是唯一的，就使用id作为key。如果没有，可以设置一个全局变量来保证id的唯一性
- 在实际生产环境中，一般使用第三方库来生成唯一id

```
const shortid = require('shortid');
```

而且错误的使用key值会有严重的后果

```
class Item extends React.Component {
  render() {
    return (
      <div className="form-group">
        <label className="col-xs-4 control-label">{this.props.name}</label>
        <div className="col-xs-8">
          <input type='text' className='form-control' />
        </div>
      </div>
    )
  }
}

class Example extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      list: []
    };
  }
  
  addItem() {
    const id = +new Date;
    this.setState({
      list: [ {name: 'Baz' + id, id} , ...this.state.list]
    });
  }
  
  render() {
    return (
      <div>
        <button className='btn btn-primary' onClick={this.addItem.bind(this)}>添加</button>
        <h3>错误<code>key=index</code></h3>
        <form className="form-horizontal">
          {this.state.list.map((todo, index) =>
            <Item {...todo} key={index} />
          )}
        </form>

        <h3>正确<code>key=id</code></h3>
        <form className="form-horizontal">
          {this.state.list.map((todo) =>
            <Item {...todo} key={todo.id} />
          )}
        </form>
      </div>
    )
  }
}

React.render(<Example />, document.getElementById('app'))
```

我们先点击添加按钮，在文本框中输入一些测试值。然后再点击一次添加按钮，在列表顶部添加一个空的文本框，这时候发现问题了。由于我们错误的使用 index 作为 key 值，React 在渲染 list 列表时，会假定 key = 0 就是我们之前添加的第一个文本框，于是错误的将第一次输入的内容渲染到了我们新添加的文本框上。


所以，除非列表是不可变的情况下，见谅减少index下表作为key

### 懒加载 lazy suspense

```
import React, {Suspense} from 'react';

const OtherComponent = React.lazy(() => import('./main/index.js'));

export default function MyComponent() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <OtherComponent />
      </Suspense>
    </div>
  );
}
```

- 延迟加载，就会有一个加载过程，之前在渲染的时候，我们基本都是自顶一个默认的组件，然后通过变量控制进行操作，如果加载完成，则取消掉默认的加载组件。
- Suspense 让你的组件在渲染之前进行“等待”，并在等待时显示 fallback 的内容
- React.lazy 函数能让你像渲染常规组件一样处理动态引入（的组件）
- React.lazy 和 suspense 并不适用于服务端渲染


这里展示如何在你的应用中使用 React.lazy 和 React Router 这类的第三方库，来配置基于路由的代码分割。
```
import React, { Suspense, lazy } from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';

const Home = lazy(() => import('./routes/Home'));
const About = lazy(() => import('./routes/About'));

const App = () => (
  <Router>
    <Suspense fallback={<div>Loading...</div>}>
      <Switch>
        <Route exact path="/" component={Home}/>
        <Route path="/about" component={About}/>
      </Switch>
    </Suspense>
  </Router>
);
```

#### Immutable

state或者props比较复杂时，我们可以在 shouldComponentUpdate 中使用使用 deepCopy 和 deepCompare 来避免无必要的 render，但 deepCopy 和 deepCompare 一般都是非常耗性能的。所以，我们一般使用Immuable.js


Immutable Data 就是一旦创建，就不能再被更改的数据。对 Immutable 对象的任何修改或添加删除操作都会返回一个新的 Immutable 对象。Immutable 实现的原理是 Persistent Data Structure（持久化数据结构），也就是使用旧数据创建新数据时，要保证旧数据同时可用且不变。同时为了避免 deepCopy 把所有节点都复制一遍带来的性能损耗，Immutable 使用了 Structural Sharing（结构共享），即如果对象树中一个节点发生变化，只修改这个节点和受它影响的父节点，其它节点则进行共享。

Immutable.is 比较的是两个对象的 hashCode 或 valueOf（对于 JavaScript 对象）。由于 immutable 内部使用了 Trie 数据结构来存储，只要两个对象的 hashCode 相等，值就是一样的。这样的算法避免了深度遍历比较，性能非常好。


使用：
```
import { is } from 'immutable';

shouldComponentUpdate: (nextProps = {}, nextState = {}) => {
  const thisProps = this.props || {}, thisState = this.state || {};

  if (Object.keys(thisProps).length !== Object.keys(nextProps).length ||
    Object.keys(thisState).length !== Object.keys(nextState).length) {
    return true;
  }

  for (const key in nextProps) {
    if (!is(thisProps[key], nextProps[key])) {
      return true;
    }
  }

  for (const key in nextState) {
    if (thisState[key] !== nextState[key] && !is(thisState[key], nextState[key])) {
      return true;
    }
  }
  return false;
}
```

- 虽然Immutable JS在性能上有它的优势，但请注意使用的影响面。不要让原生对象和Immutable对象进行混用，这样反而会导致性能下降，因为将Immutable数据再转换成原生JS对象在性能上是很差的。

- !!注意 在react中使用immutable 时，state 本身必须是普通对象，但是里面的值可以是 immutable 的数据。


#### 更小的seamless-Immutable 

你也可以选择seamless-Immutable 

seamless-Immutable是一套轻量级的持久化数据结构库 , seamless-immutable 并没有实现完整的 Persistent Data Structure 而是使用 Object.defineProperty（因此只能在 IE9 及以上使用）扩展了 JavaScript 的 Array 和 Object 对象来实现，只支持 Array 和 Object 两种数据类型，API 基于与 Array 和 Object 操持不变。代码库非常小，压缩后下载只有 2K。相比 Immutable.js 压缩后下载有 16K而言，小巧了很多, 而且API比较简洁，易懂。


### hooks相关

Hooks最大的优势就在于逻辑复用。16.8及以上版本提供了很多的hooks。

#### useMemo

父组件
```
function App() {
  const [name, setName] = useState('名称')
  const [content,setContent] = useState('内容')
  return (
      <>
        <button onClick={() => setName(new Date().getTime())}>name</button>
        <button onClick={() => setContent(new Date().getTime())}>content</button>
        <Button name={name}>{content}</Button>
      </>
  )
}
```

子组件
```
function Button({ name, children }) {
  function changeName(name) {
    console.log('11')
    return name + '改变name的方法'
  }

  const otherName =  changeName(name)
  return (
      <>
        <div>{otherName}</div>
        <div>{children}</div>
      </>

  )
}
```

当我们点击父组件的按钮的时候，子组件的name和children都会发生变化。
不管我们是改变name或者content的值，我们发现 changeName的方法都会被调用。
是不是意味着 我们本来只想修改content的值，但是由于name并没有发生变化，所以无需执行对应的changeName方法。但是发现他却执行了。 这是不是就意味着性能的损耗，做了无用功。


优化之后的子组件
```
function Button({ name, children }) {
  function changeName(name) {
    console.log('11')
    return name + '改变name的方法'
  }

const otherName = useMemo(()=>changeName(name), [name])
  return (
      <>
        <div>{otherName}</div>
        <div>{children}</div>
      </>

  )
}
export default Button
```

这个时候我们点击改变content值的按钮，发现changeName 并没有被调用。但是点击改变name值按钮的时候，changeName被调用了。
所以我们可以使用useMemo方法 避免无用方法的调用，当然一般我们changName里面可能会使用useState来改变state的值，那是不是就避免了组件的二次渲染。
达到了优化性能的目的。

#### useCallback

const fnA = useCallback(fnB, [a])

useCallback跟useMemo比较类似，但它返回的是缓存的函数

```
import React, { useState, useCallback } from 'react';
const set = new Set();
export default function Callback() {
    const [count, setCount] = useState(1);
    const [val, setVal] = useState('');
 
    const callback = useCallback(() => {
        console.log(count);
    }, [count]);
    set.add(callback);
 
 
    return <div>
        <h4>{count}</h4>
        <h4>{set.size}</h4>
        <div>
            <button onClick={() => setCount(count + 1)}>+</button>
            <input value={val} onChange={event => setVal(event.target.value)}/>
        </div>
    </div>;
}
```

可以看到，每次修改count，set.size就会+1，这说明useCallback依赖变量count，count变更时会返回新的函数；而val变更时，set.size不会变，说明返回的是缓存的旧版本函数。


还有这种场景：有一个父组件，其中包含子组件，子组件接收一个函数作为props；通常而言，如果父组件更新了，子组件也会执行更新；但是大多数场景下，更新是没有必要的，我们可以借助useCallback来返回函数，然后把这个函数作为props传递给子组件；这样，子组件就能避免不必要的更新。
```
import React, { useState, useCallback, useEffect } from 'react';
function Parent() {
  const [count, setCount] = useState(1);
  const [val, setVal] = useState('');

  const callback = useCallback(() => {
      return count;
  }, [count]);
  return <div>
      <h4>{count}</h4>
      <Child callback={callback}/>
      <div>
          <button onClick={() => setCount(count + 1)}>+</button>
          <input value={val} onChange={event => setVal(event.target.value)}/>
      </div>
  </div>;
}
 
function Child({ callback }) {
  const [count, setCount] = useState(() => callback());
  useEffect(() => {
      setCount(callback());
  }, [callback]);
  return <div>
      {count}
  </div>
```

- useMemo和useCallback都会在组件第一次渲染的时候执行，之后会在其依赖的变量发生改变时再次执行；并且这两个hooks都返回缓存的值，useMemo返回缓存的变量，useCallback返回缓存的函数。


#### 是否一定要用useMemo/useCallback进行优化

useMemo/useCallback的使用非常简单，不过我们需要思考一个问题，使用他们一定能够达到优化的目的吗？

React的学习经常容易陷入过度优化的误区。一些人在得知shouldComponentUpdate能够优化性能，恨不得每个组件都要用一下，不用就感觉自己的组件有问题。useMemo/useCallback也是一样。

我们知道记忆函数的原理，应该知道，记忆函数并非完全没有代价，我们需要创建闭包，占用更多的内存，用以解决计算上的冗余。
useMemo/useCallback也是一样，这是一种成本上的交换。那么我们在使用时，就必须要思考，这样的交换，到底值不值。

如果不使用useCallback，我们就必须在函数组件内部创建超多的函数，这种情况是不是就一定有性能问题呢？

答案是否定的

我们知道，一个函数执行完毕之后，就会从函数调用栈中被弹出，里面的内存也会被回收。因此，即使在函数内部创建了多个函数，执行完毕之后，这些创建的函数也都会被释放掉。函数式组件的性能是非常快的。相比class，函数更轻量，也避免了使用高阶组件、renderProps等会造成额外层级的技术。使用合理的情况下，性能几乎不会有什么问题。
而当我们使用useMemo/useCallback时，由于新增了对于闭包的使用，新增了对于依赖项的比较逻辑，因此，盲目使用它们，甚至可能会让你的组件变得更慢。
大多数情况下，这样的交换，并不划算，或者赚得不多。你的组件可能并不需要使用useMemo/useCallback来优化。


### mixin、hoc、render props、hooks ？

区别与联系，合理运用


### 数据流管理

“单向数据绑定”是 React 推崇的一种应用架构的方式。当应用足够复杂时才能体会到它的好处，虽然在一般应用场景下你可能不会意识到它的存在

- 单向数据流中的‘单向’-- 数据从父组件到子组件这个流向叫单向。

对于复杂的父级与孙子组件的数据通信，我们平常用的通过props传值的方法就显的不那么优雅简单了。

react提供了context 

老的版本：
Context需要用到两种组件，一个是Context生产者(Provider)，通常是一个父节点，另外是一个Context的消费者(Consumer)，通常是一个或者多个子节点。所以Context的使用基于生产者消费者模式。
对于父组件，也就是Context生产者，需要通过一个静态属性childContextTypes声明提供给子组件的Context对象的属性，并实现一个实例getChildContext方法，返回一个代表Context的纯对象 (plain object) 。

```
import React from 'react'
import PropTypes from 'prop-types'

class MiddleComponent extends React.Component {
  render () {
    return <ChildComponent />
  }
}

class ParentComponent extends React.Component {
  // 声明Context对象属性
  static childContextTypes = {
    propA: PropTypes.string,
    methodA: PropTypes.func
  }
  
  // 返回Context对象，方法名是约定好的
  getChildContext () {
    return {
      propA: 'propA',
      methodA: () => 'methodA'
    }
  }
  
  render () {
    return <MiddleComponent />
  }
}
```

而对于Context的消费者，通过如下方式访问父组件提供的Context。

```
import React from 'react'
import PropTypes from 'prop-types'

class ChildComponent extends React.Component {
  // 声明需要使用的Context属性
  static contextTypes = {
    propA: PropTypes.string
  }
  
  render () {
    const {
      propA,
      methodA
    } = this.context
    
    console.log(`context.propA = ${propA}`)  // context.propA = propA
    console.log(`context.methodA = ${methodA}`)  // context.methodA = undefined
    
    return ...
  }
}
```

新的context：

```
import React from 'react'
//定义context
const RootContext = React.createContext({
  showEyeCareDialog: false,
  eyeCareBtn: 'unchecked',
  clickEyeCareFunc: () => {},
});

// context 对象接受一个名为 displayName 的 property，类型为字符串。React DevTools 使用该字符串来确定 context 要显示的内容。
RootContext.displayName = 'RootContext';

export default RootContext


// 父组件render， 提供context
render() {
  let {showEyeCareDialog} = this.state
  return <div class="outer_container">
    <RootContext.Provider value={
      {
        ...this.state,
        clickEyeCareFunc: this.clickEyeCareFunc
      }
    }>
      {this.props.children}
      <EyeCareDialog
        visible={showEyeCareDialog}
        close={this.closeEyeCardDilog}/>
    </RootContext.Provider>
  </div>
}

//子组件，获取context数据
return <RootContext.Consumer>
  {({clickEyeCareFunc, eyeCareBtn}) => (
    <>
      <div class="set_container">
      xxxx
      </div>
    </>
  )}
</RootContext.Consumer>

```

useContext hooks 官方示例：
```
const themes = {
  light: {
    foreground: "#000000",
    background: "#eeeeee"
  },
  dark: {
    foreground: "#ffffff",
    background: "#222222"
  }
};

const ThemeContext = React.createContext(themes.light);

function App() {
  return (
    <ThemeContext.Provider value={themes.dark}>
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return (
    <button style={{ background: theme.background, color: theme.foreground }}>
      I am styled by theme context!
    </button>
  );
}
```

当然，必不可少的还有redux，关于redux的内容就太多了

比如：
redux尽量扁平化store
本身的增强combineReducer？compose？
一些常用中间件？ reselect避免冗余计算、redux-actions
处理异步？ redux-saga、redux-promise、redux-thunk、redux-observable
...

简单写下redux-ignore的使用：

我们在构建应用时，无法避免地，当项目越来越大时，action和相应reducer的数量会骤增。但是也许与大部分人的预期相反，当一个 action 触发时，不仅仅是该 action 对应的 reducer 执行了，其他所有的 reducer 也同时被执行了，只是 action type 不匹配，state 的值不变而已。这种空操作，也在无形中消耗了宝贵的性能

```
import { combineReducers } from "redux"
import { ignoreActions, filterActions } from 'redux-ignore'
import home from "./home/reducer"
import {USER_INFO, UPDATE_USER_INFO, CLEAR_USER_INFO } from "./my/action-type";
export default combineReducers({
  home, // 首页
  my: filterActions(my, [USER_INFO, UPDATE_USER_INFO, CLEAR_USER_INFO]), // 我的,
});
```
这样在调用非my的reducer的时候，不再执行定义的my的reducer


## 2.代码体积

- 使用 生产版本
- 使用 webpack-bundle-analyzer 可视化 webpack 输出文件的大小
- 使用动态 import，懒加载 React 组件
- 使用 Tree Shaking & 教程 & Tree Shaking 优化
- 使用 babel-plugin-import 优化业务组件的引入，实现按需加载
- 使用 SplitChunksPlugin 拆分公共代码
- 优化 Webpack 中的库
- 分析 CSS 和 JS 代码覆盖率
...

## 从业务流程方面：

一般的渐进式加载的全过程如下：
用户打开页面，这个时候页面是完全空白的；
然后 html 和引用的 css 加载完毕，浏览器进行首次渲染，我们把首次渲染需要加载的资源体积称为 “首屏体积”；
然后 react、react-dom、业务代码加载完毕，应用第一次渲染，或者说首次内容渲染；
然后应用的代码开始执行，拉取数据、进行动态import、响应事件等等，完毕后页面进入可交互状态；
然后直到页面的其它资源（如错误上报组件、打点上报组件等）加载完毕，整个页面的加载就结束了。


### 图片懒加载
在大量图片加载的情况下，会造成页面空白甚至卡顿，然而我们的视口就这么大，因此只需要让视口内的图片显示即可，同时图片未显示的时候给它一个默认的 src，让一张非常精简的图片占位。这就是图片懒加载的原理。我们采取一个成熟的方案 react-lazyload 库，易上手，效果不错。

```
npm install react-lazyload --save
//components/list.js
// 引入
import LazyLoad from "react-lazyload";

//img 标签外部包裹一层 LazyLoad
<LazyLoad placeholder={<img width="100%" height="100%" src={require ('./music.png')} alt="music"/>}>
  <img src={item.picUrl + "?param=300x300"} width="100%" height="100%" alt="music"/>
</LazyLoad>
```


还有我们在加载文本、图片的时候，经常出现“闪屏”的情况，比如图片或者文字还没有加载完毕，此时页面上对应的位置还是完全空着的，然后加载完毕，内容会突然撑开页面，导致“闪屏”的出现，造成不好的体验。
一些优秀的组件react-placeholder、react-hold 解决placeholder问题


### spa项目首屏加载缓慢过度办法

使用 html-webpack-plugin 自动插入 loading

创建loading.html
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
</head>
<body>
  <div class="pre-loading">
      loading...
  </div>
</body>
</html>

```

webpack配置
```
// 读取写好的 loading 态的 html 和 css
var loading = {
  html: fs.readFileSync(path.join(__dirname, './loading.html')),
  css: '<style>' + fs.readFileSync(path.join(__dirname, './loading.css')) + '</style>'
}

const htmlWebpackPluginArr = Array.of('a', 'b', 'c').map(p => {
  return new HtmlWebpackPlugin({
      filename: `${p}.html`,
      template: `sample${!args.dev ? '/cdn' : ''}/${p}.html`,
      inject: true,
      chunks: [p],
      minify: args.dev ? {} : minifyOb,
      cdnConfig: externalConfig,
      loading: loading
    })
})
```


### 长列表

对于较长的列表，比如1000个数组的数据结构，如果想要同时渲染这1000个数据，生成相应的1000个原生dom，我们知道原生的dom元素是很复杂的，同时渲染很多dom节点，也会造成一下几个问题：

- 容易失帧，因为渲染很慢，所以无法维持浏览器的帧率，主观上会显得页面卡顿
- 网页失去响应，事件等无法及时被触发

贯穿React核心的就是"virtual dom",我们同样的可以通过用虚拟列表的方式来优雅的优化长列表

通过react-virtualized 能够很好地处理大型列表（它只会渲染列表显示在屏幕上的一小部分，只有滚动列表时才会更新）

- 或者通过react-tiny-virtual-list来优化长列表

```
render(){
    return <div style={{width:"240px",height:"80px",padding:"15px",border:"4px double red",resize:"both",overflow:"auto"}}>
        <AutoSizer>
        {({ height, width }) => (
          <List
            height={height}
            rowCount={list.length}
            rowHeight={cache.rowHeight}
            deferredMeasurementCache={cache}
            rowRenderer={cellRenderer}
            width={width}
          />
        )}
      </AutoSizer>
    </div>
  }
```

### 动画

react-transition-group

```
import { TransitionGroup, CSSTransition } from 'react-transition-group'

...

render() {
  return (
    <TransitionGroup className={'router-wrapper'}>
      <CSSTransition timeout={300} classNames={'fade'} key={location.hash} unmountOnExit={true}>
        <Switch>
          <Route path="/schedule" exact component={Schedule} />
          <Route path="/set" exact component={Set} />
          <Route path="/personal" exact component={Personal} />
          <Route path="/faq" exact component={Faq} />
          <Route path="/classHour" exact component={ClassHour} />
          <Redirect to="/schedule" />
        </Switch>
      </CSSTransition>
    </TransitionGroup>
  )
}
```

### 复杂运算引入webworker

对于高的计算，引入webworker。后台执行复杂运算

## 3.性能度量、性能检测

### 3.1 React DevTools

React 16.5 增加了对新的开发者工具 DevTools 性能分析插件的支持。 此插件使用 React 实验性的 Profiler API 来收集有关每个组件渲染的用时信息，以便识别 React 应用程序中的性能瓶颈。它将与我们即将推出的time slicing（时间分片） 和 suspense（悬停） 功能完全兼容。

DevTools 将为支持新的 Profiler API 的应用显示 “Profiler” 选项卡：

like this: 

![](/imgs/简单梳理react优化相关/reactdevtool.png)


可以： 浏览 commits、过滤 commits、看火焰图、排序图标、查看组件图标等

[官方中文](http://react.html.cn/blog/2018/09/10/introducing-the-react-profiler.html#flame-chart)

- 功能很强大

#### 3.2 Profiler

Profiler 测量渲染一个 React 应用多久渲染一次以及渲染一次的“代价”。 它的目的是识别出应用中渲染较慢的部分，或是可以使用类似 memoization 优化的部分，并从相关优化中获益。


[官方中文](https://reactjs.bootcss.com/docs/profiler.html)

```
render(
  <App>
    <Profiler id="Navigation" onRender={onRenderCallback}>
      <Navigation {...props} />
    </Profiler>
    <Main {...props} />
  </App>
);


function onRenderCallback(
  id, // 发生提交的 Profiler 树的 “id”
  phase, // "mount" （如果组件树刚加载） 或者 "update" （如果它重渲染了）之一
  actualDuration, // 本次更新 committed 花费的渲染时间
  baseDuration, // 估计不使用 memoization 的情况下渲染整颗子树需要的时间
  startTime, // 本次更新中 React 开始渲染的时间
  commitTime, // 本次更新中 React committed 的时间
  interactions // 属于本次更新的 interactions 的集合
) {
  // 合计或记录渲染时间。。。
}
```

### 3.3 @welldone-software/why-did-you-render 组件

可以使用@welldone-software/why-did-you-render在控制台查看触发 View 更新的原因

- 由于why-did-you-update 已经弃用了，现转为@welldone-software/why-did-you-render 

用法：

```
// 创建wdyr.js 然后在入口导入
import React from 'react';
if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    trackHooks: true
  });
}

// 导入
import '@common/wdyr'


// 在你想分析的组件添加静态属性whyDidYouRender
class RootComp extends Component {
  // 静态属性
  static whyDidYouRender = true

  constructor(props) {
    super(props)
    this.state = {
    }
  }
  ....
}
```

出现不必要的render会有类似如下的提示：
![](/imgs/简单梳理react优化相关/wdyrrerender.png)


### 3.4 Performance api

performance.mark：

```
// 以一个标志开始。
performance.mark("mySetTimeout-start");
// 等待一些时间。
setTimeout(function() {
  // 标志时间的结束。
  performance.mark("mySetTimeout-end");

  // 测量两个不同的标志。
  performance.measure(
    "mySetTimeout",
    "mySetTimeout-start",
    "mySetTimeout-end"
  );

  // 获取所有的测量输出。
  // 在这个例子中只有一个。
  var measures = performance.getEntriesByName("mySetTimeout");

  var measure = measures[0];
  console.log("setTimeout milliseconds:", measure.duration)

  // 清除存储的标志位
  // performance.clearMarks();
  // performance.clearMeasures();
}, 1000);
```

- 你可以react工程的任意位置用performance.mark进行统计数据分析


当然performance还有其他的api, 控制台打印下看看就知道了
performance.memory
performance.navigation
performance.timeOrigin
performance.timing
...

### 当然我们可以用chrome浏览器自带的工具 performance 面板、Audits面版、memory面板、Porformance monitor等进行分析

不一一列举分析了

***最好的优化就是不碰到瓶颈前别瞎优化***



