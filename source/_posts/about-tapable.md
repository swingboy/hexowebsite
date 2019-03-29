---
title: about tapable
date: 2019-03-29 20:46:15
tags:
---


## tapable

Webpack 本质上是一种事件流的机制，它的工作流程就是将各个插件串联起来，而实现这一切的核心就是 tapable ，Webpack 中最核心的，负责编译的 Compiler 和负责创建 bundles 的 Compilation 都是 tapable 构造函数的实例。

都是继承自Tapable

```
class Compiler extends Tapable {
    // ...
}
```

webpack的灵魂Tapable，有点类似于nodejs的Events，都是注册一个事件，然后到了适当的时候触发。这里的事件触发是这样绑定触发的，通过on方法，绑定一个事件，emit方法出发一个事件。Tapable的机制和这类似，也是tap注册一个事件，然后call执行这个事件。


```
const EventEmitter = require('events');
const myEmitter = new EventEmitter();
myEmitter.on('newListener', (param1,param2) => {
	console.log("newListener",param1,param2)
});
myEmitter.emit('newListener', 1, 2);
```

- 想想ttm ~~~


### Tabable是什么

tapable库暴露了很多Hook（钩子）类，为插件提供挂载的钩子。

```
const {
    SyncHook,
    SyncBailHook,
    SyncWaterfallHook,
    SyncLoopHook,
    AsyncParallelHook,
    AsyncParallelBailHook,
    AsyncSeriesHook,
    AsyncSeriesBailHook,
    AsyncSeriesWaterfallHook
 } = require("tapable");
```

- tabpack提供了同步&异步绑定钩子的方法，并且他们都有绑定事件和执行事件对应的方法。

Async*	                        Sync*
绑定：tapAsync/tapPromise/tap	 绑定：tap
执行：callAsync/promise	        执行：call


*** Tabable的其他方法 *** 
type	function  
Hook	所有钩子的后缀  
Waterfall	同步方法，但是它会传值给下一个函数  
Bail	熔断：当函数有任何返回值，就会在当前执行函数停止  
Loop	监听函数返回true表示继续循环，返回undefine表示结束循环  
Sync	同步方法  
AsyncSeries	异步串行钩子  
AsyncParallel	异步并行执行钩子  

#### 用法：

```
const { SyncHook } = require('tapable');
const mySyncHook = new SyncHook(['name', 'age']);
mySyncHook.tap('1', function (name, age) {
    console.log(name, age, 1)
    return 'wrong' // 不关心返回值 这里写返回值对结果没有任何影响
});

mySyncHook.tap('2', function (name, age) {
    console.log(name, age, 2)
});

mySyncHook.tap('3', function (name, age) {
    console.log(name, age, 3)
});

mySyncHook.call('liushiyu', '18');
```

- tap接收两个参数，第一个参数是名称（没有任何意义）第二个参数是一个函数 接收一个参数.


and ~~~~

```
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable");
const hook = new SyncHook(["arg1", "arg2", "arg3"]);
//The best practice is to expose all hooks of a class in a hooks property:

class Car {
	constructor() {
		this.hooks = {
			accelerate: new SyncHook(["newSpeed"]),
			brake: new SyncHook(),
			calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
		};
	}

	/* ... */
}
//Other people can now use these hooks:

const myCar = new Car();

// Use the tap method to add a consument
myCar.hooks.brake.tap("WarningLampPlugin", () =>  {console.log(`it's work`)});

myCar.hooks.brake.call()
```

### 流程

### 订阅  
SyncHook.tap('xx') 
（其中tap 继承自Hook）
this.taps = [];
-> this._insert(options);

### 发布

SyncHook.call   
Hook.prototype 上的_call 方法。

```
Object.defineProperties(Hook.prototype, {
	_call: {
		value: createCompileDelegate("call", "sync"),
		configurable: true,
		writable: true
	},
	
});
```

createCompileDelegate -> _createCall -> HookCodeFactory ->  create  返回fn 。然后执行封装过的工厂函数
返回对应的的value。


#### SyncHook
同步串行不关心订阅函数执行后的返回值是什么。其原理是将监听(订阅)的函数存放到一个数组中, 发布时遍历数组中的监听函数并且将发布时的 arguments传递给监听函数

```
class SyncHook {
  constructor(options) {
    this.options = options
    this.hooks = []  // 缓存订阅的事件
  }

  /**
   * 订阅事件
   * @param name
   * @param callback
   */
  tap(name, callback) {
    this.hooks.push(callback)
  }

  /**
   * 发布事件
   * @param args
   */
  call(...args) {
    this.hooks.forEach(task => task(...args))
  }
}
```

#### SyncWaterfallHook

同步串行瀑布流, 瀑布流指的是第一个监听函数的返回值,做为第二个监听函数的参数。第二个函数的返回值作为第三个监听函数的参数,依次类推...

```
const syncWaterfallHook = new SyncWaterfallHook('name')

syncWaterfallHook.tap('name', data => {
	console.log('name', data)
	return 23
})
syncWaterfallHook.tap('age', data => {
	console.log('age', data)
})

syncWaterfallHook.call('qiqingfu')
```


伪实现

```
class SyncWaterfallHook {
    constructor () {
        this.hooks = [];
    }
    tap (name, fn) {
        this.hooks.push(fn)
    }
    call () {
        let result = null
        for(let i=0; i< this.hooks.length; i++) {
            let hook = this.hooks[i];
            if (!i) {
                result = hook(...arguments)
            } else {
                result = hook(result)
            }
        }
    }
}
```

- 实际的比这个复杂一些。
- 奠定了webpack插件体系的基础
- 看明白这个就可以读webpack代码了