---
title: webpackPlugin
date: 2019-03-29 20:48:37
tags: 技术
---

### plugin 

先把[中文](https://www.webpackjs.com/concepts/) 或[英文](https://webpack.js.org/contribute/writing-a-plugin/)文档看一下。

### 实际应用

如下方代码

```
const ProgressBarPlugin = require('progress-bar-webpack-plugin');
module.exports = {
    mode: 'development',
    plugins:[
        new ProgressBarPlugin({
            width: 5,
        }),
    ],  
};
```


### 什么是webpack 插件

webpack plugin是一个具有 apply 属性的 JavaScript 对象。apply 属性会被 webpack compiler 调用，并且 compiler 对象可在整个编译生命周期访问。

- webpack 插件机制是整个 webpack 工具的骨架，而 webpack 本身也是利用这套插件机制构建出来的。

一个简单的插件结构如下：

```
const pluginName = 'ConsoleLogOnBuildWebpackPlugin';

class ConsoleLogOnBuildWebpackPlugin {
    apply(compiler) {
        compiler.hooks.run.tap(pluginName, compilation => {
            console.log("webpack 构建过程开始！");
        });
    }
}
```

再看一个

```
function HelloCompilationPlugin(options) {}

HelloCompilationPlugin.prototype.apply = function(compiler) {

  // 设置回调来访问 compilation 对象：
  compiler.plugin("compilation", function(compilation) {

    // 现在，设置回调来访问 compilation 中的步骤：
    compilation.plugin("optimize", function() {
      console.log("Assets are being optimized.");
    });
  });
};

module.exports = HelloCompilationPlugin;
```

有什么特点呢？ 

我们看到了都有个Compiler 和 Compilation

### 不得不说的 Compiler 和 Compilation

在插件开发中最重要的两个资源就是 compiler 和 compilation 对象。

#### 区别与联系
compiler 对象代表了完整的 webpack 环境配置。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。

而compilation 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用。

- 这两个组件是任何 webpack 插件不可或缺的部分（特别是 compilation）

看看他们的代码会有很多收获  

[Compilation](https://github.com/webpack/webpack/blob/master/lib/Compilation.js)

[Compiler](https://github.com/webpack/webpack/blob/master/lib/Compiler.js)


#### Compiler

Compiler暴露了和webpack整个生命周期相关的钩子，通过如下的方式访问:

```
//基本写法
compiler.hooks.someHook.tap(...)
//如果希望在entry配置完毕后执行某个功能
compiler.hooks.entryOption.tap(...)
//如果希望在生成的资源输出到output指定目录之前执行某个功能
compiler.hooks.emit.tap(...)
```

- webpack中有complier对象，代表webpack实例。有编译的具体参数。
- complation代表本次编译，有打包的文件等信息


#### complication

[对应钩子](https://www.webpackjs.com/api/compilation-hooks/)


buildModule -- 在模块构建开始之前触发

failedModule -- 模块构建失败时执行

seal -- 编译(compilation)停止接收新模块时触发。

optimize -- 优化阶段开始时触发。

...


### Tapable

说到 compiler 和 compilation 对象更不得不提 Tapable

tapable是webpack的核心框架，是一个基于事件流的框架，或者叫做发布订阅模式，或观察者模式。webpack 中许多对象扩展自 Tapable 类。

tapable 这个类暴露 tap, tapAsync 和 tapPromise 方法，可以使用这些方法，注入自定义的构建步骤，这些步骤将在整个编译过程中不同时机触发。

[先看下看另一篇]()


### 插件类型(plugin types)

根据所使用的 钩子(hook) 和 tap 方法，插件可以以多种不同的方式运行。这个工作方式与 Tapable 提供的 hooks 密切相关。compiler hooks 分别记录了 Tapable 内在的钩子，指出哪些 tap 方法可用。

因此，根据你触发到 tap 事件，插件可能会以不同的方式运行。例如，当钩入 compile 阶段时，只能使用同步的 tap 方法：

```
compiler.hooks.compile.tap('MyPlugin', params => {
  console.log('以同步方式触及 compile 钩子。')
})
```

然而，对于能够使用了 AsyncHook(异步钩子) 的 run，我们可以使用 tapAsync 或 tapPromise（以及 tap）：

```
compiler.hooks.run.tapAsync('MyPlugin', (compiler, callback) => {
  console.log('以异步方式触及 run 钩子。')
  callback()
})

compiler.hooks.run.tapPromise('MyPlugin', compiler => {
  return new Promise(resolve => setTimeout(resolve, 1000)).then(() => {
    console.log('以具有延迟的异步方式触及 run 钩子')
  })
})
```

- 可以有多种方式将 hook 钩入到 compiler 中，可以让各种插件都以合适的方式去运行。


```
class Compiler extends Tapable {
	constructor(context) {
		super();
		this.hooks = {
			/** @type {SyncBailHook<Compilation>} */
			shouldEmit: new SyncBailHook(["compilation"]),
			/** @type {AsyncSeriesHook<Stats>} */
			done: new AsyncSeriesHook(["stats"]),
			/** @type {AsyncSeriesHook<>} */
			additionalPass: new AsyncSeriesHook([]),
			/** @type {AsyncSeriesHook<Compiler>} */
			beforeRun: new AsyncSeriesHook(["compiler"]),
			/** @type {AsyncSeriesHook<Compiler>} */
			run: new AsyncSeriesHook(["compiler"]),
			/** @type {AsyncSeriesHook<Compilation>} */
			emit: new AsyncSeriesHook(["compilation"]),
			/** @type {AsyncSeriesHook<Compilation>} */
			afterEmit: new AsyncSeriesHook(["compilation"]),
```



#### 自定义的钩子函数(custom hooks)

为了给其他插件的编译添加一个新的钩子，来 tap(触及) 到这些插件的内部，直接从 tapable 中 require 所需的钩子类(hook class)，然后创建：

```
const SyncHook = require('tapable').SyncHook;

// 具有 `apply` 方法……
if (compiler.hooks.myCustomHook) throw new Error('Already in use');
compiler.hooks.myCustomHook = new SyncHook(['a', 'b', 'c'])

// 在你想要触发钩子的位置/时机下调用……
compiler.hooks.myCustomHook.call(a, b, c);
```

### 如何写一个plugin？

看完上面了解了一些基本内容，以及plugin的样子。怎么写一定要知道怎么样的一个东西可以称之为 webpack plugin呢？

一个完整的 webpack 插件需要满足以下几点规则和特征：

- 是一个独立的模块。
- 模块对外暴露一个 js 函数。
- 函数的原型 (prototype) 上定义了一个注入 compiler 对象的 apply 方法。
- apply 函数中需要有通过 compiler 对象挂载的 webpack 事件钩子，钩子的回调中能拿到当前编译的 compilation 对象，如果是异步编译插件的话可以拿到回调 callback。
- 完成自定义子编译流程并处理 complition 对象的内部数据。
- 如果异步编译插件的话，数据处理完成后执行 callback 回调。


### 两个plugin例子
（1）在每个chunk 中添加 个banner

```
const { ConcatSource } = require("webpack-sources");
let pluginOptions = {
    name: 'myName',
    stage: Infinity
}

class BannerPlugin {
  constructor(doneF, failF) {
    this.doneF = doneF;
    this.failF = failF;
  }
  
  apply(compiler) {
      compiler.hooks.compilation.tap("BannerTest", compilation => {
    compilation.hooks.optimizeChunkAssets.tap("BannerTest", (chunks, callbak)  => {
      chunks.forEach(chunk => {
                  chunk.files.forEach(file => {
                      compilation.assets[file] = new ConcatSource(
                      '\/**i am the banner-  ' + new Date() + '-**\/',
                      '\n',
                      compilation.assets[file]
                      );
                  });
              });
    });
  });   
  }
}

module.exports = BannerPlugin;
```

（2）编译成功和失败回调

```
const { ConcatSource } = require("webpack-sources");
let pluginOptions = {
    name: 'myName',
    stage: Infinity
}

class ResultPlugin {
  constructor(doneF, failF) {
    this.doneF = doneF;
    this.failF = failF;
  }
  
  apply(compiler) {
      compiler.hooks.done.tap(pluginOptions,  (stats) => {
          this.doneF('我成功了');
      });

      compiler.hooks.failed.tap(pluginOptions,   (err) => {
          this.failF(err);
      });

      compiler.hooks.compilation.tap("BannerTest", compilation => {
    compilation.hooks.optimizeChunkAssets.tap("BannerTest", (chunks, callbak)  => {
      chunks.forEach(chunk => {
                  chunk.files.forEach(file => {
                      compilation.assets[file] = new ConcatSource(
                      '\/**Sweet Banner**\/',
                      '\n',
                      compilation.assets[file]
                      );
                  });
              });
    });
  }); 
  }
}

module.exports = ResultPlugin;
```

- compiler hook 的 tap 方法的第一个参数，应该是驼峰式命名的插件名称。建议为此使用一个常量，以便它可以在所有 hook 中复用。


### 总结

webpack plugin的本质，它实际上和webpack loader一样简单，其实它只是一个带有apply方法的class。

插件目的在于解决 loader 无法实现的其他事。
