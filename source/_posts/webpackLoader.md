---
title: webpackLoader
date: 2019-03-29 20:47:50
tags: 技术
---
### 什么是 Loader ？
本质上来说，loader 就是一个 node 模块，这很符合 webpack 中“万物皆模块”的思路。既然是 node 模块，那就一定会导出点什么。在 webpack 的定义中，loader 导出一个函数，loader 会在转换源模块（resource）的时候调用该函数。在这个函数内部，我们可以通过传入 this 上下文给 Loader API 来使用它们。

- 我们也可以概括一下 loader 的功能：把源模块转换成通用模块。


我们常常见到的，webpack 中的配置

```
let webpackConfig = {
    //...
    module: {
        rules: [{
            test: /\.js$/,
            use: [{
                loader: 'x-loader', 
                options: {/* ... */}
            }, {
                loader: 'y-loader', 
                options: {/* ... */}
            }]
        }]
    },
    resolveLoader: {
        modules: ['node_modules', path.resolve(__dirname, 'loaders')]
    }
}
```

### 好处
loader为 JavaScript 生态系统提供了更多能力。 用户现在可以更加灵活地引入细粒度逻辑，例如压缩、打包、语言翻译和其他更多。  

### 常用loader

style-loader——将处理结束的CSS代码存储在js中，运行时嵌入&lt;style&gt;后挂载至html页面上  
css-loader——加载器，使webpack可以识别css模块  
sass-loader——加载器，使webpack可以识别scss/sass文件，默认使用node-sass进行编译  
less-loader  
url-loader   
babel-loader  
cache-loader  
...

一堆~~

### loader 特性

支持链式传递。一组链式的 loader 将按照相反的顺序执行。loader 链中的第一个 loader 返回值给下一个 loader。在最后一个 loader，返回 webpack 所预期的 JavaScript。 
可以是同步的，也可以是异步的。  

运行在 Node.js 中，并且能够执行任何可能的操作。 

接收查询参数。用于对 loader 传递配置。  

也能够使用 options 对象进行配置。  

除了使用 package.json 常见的 main 属性，还可以将普通的 npm 模块导出为 loader，做法是在 package.json 里定义一个 loader 字段。  

loader 能够产生额外的任意文件。


### webpack 中如何工作的

当你在 webpack 项目中引入模块时，匹配到 rule （例如下面的 /\.js$/）就会启用对应的 loader。这时loader 会导出一个函数，这个函数接受的唯一参数是一个包含源文件内容的字符串。也就是source

接着我们在函数中处理 source 的转化，最终返回处理好的值。当然返回值的数量和返回方式依据 loader 的需求来定。一般情况下可以通过 return 返回一个值，也就是转化后的值。如果需要返回多个参数，则须调用 this.callback(err, values...) 来返回。在异步 loader 中你可以通过抛错来处理异常情况。Webpack 建议我们返回 1 至 2 个参数，第一个参数是转化后的 source，可以是 string 或 buffer。第二个参数可选，是用来当作 SourceMap 的对象。

#### 这些loader如何一起工作的

归根结底是将某中标准的文件转化为另一种标准的文件，也可以理解为字符串

以处理 scss 文件为例：

* 1.scss 源代码会先交给 sass-loader 把 scss 转换成 css；  
* 2.把 sass-loader 输出的 CSS 交给 css-loader 处理，找出 CSS 中依赖的资源、压缩 CSS 等；  
* 3.把 css-loader 输出的 CSS 交给 style-loader 处理，转换成通过脚本加载的 JavaScript 代码；



```
webpack.config.js

    {
        test: /\.js/,
        use: [
            'style-loader',
            'css-loader',
            'sass-loader'
        ]
    }
```

loader 的调用顺序是 sass-loader -> css-loader -> style-loader。
sass-loader 拿到 source，处理后把 JS 代码传递给 css-loader，css-loader 拿到 sass-loader 处理过的 “source” ，再处理之后给 style-loader，style-loader 处理完后再交给 webpack。
style-loader 最终把返回值和 source map 传给 webpack。


#### 开发 Loader
Loader 就像是一个翻译员，能把源文件经过转化后输出新的结果，并且一个文件还可以链式的经过多个翻译员翻译。

首先我们要知道：

#### 分类
按照loader的返回值可以分为两种：

* 最左loader：这种loader会返回字符串描述的js模块代码，已经是loader的最终处理结果了，这样的字符串会被添加到webpack的模块函数中
* 非最左loader：返回值不是js模块代码，而仅仅是对资源的中间处理结果，这样的字符串需要被后续的loader处理

 （一般情况下，在loader的链式调用中，一般是这样：最左loader！非最左loader！非最左loader ....）
 
 
 ```
module.exports = function loader(source) {
	//返回的值是js模块代码，这个loader属于最左loader  
    return `module.exports = {fn: ${source}}`;
};

 ```
 
 
##### aync loader
把以上例子中的loader定义为

 ```
 module.exports = function(source) {
    var callback = this.async();
    setTimeout(function(){
       callback(null,`module.exports = {fn: ${source}}`)
    },5000);
};
 ```
 
##### pitching loader
 
在loader函数对象上添加一个pitch属性，这个pitch所执行的函数称为pitching loader。在pitching loader中可以通过data把数据传递给对应的loader，而不能传递给其他loader。

在链式调用中，pitching loader 与 loader的执行次序（以 a!b!c!resource 为例）

pitch a   
	pitch b  
		pitch c  
			read file resource (adds resource to dependencies)  
		run c  
	run b  
run a  

- style-loader 为例
 

[官方翻译文档编写一个loader](https://www.webpackjs.com/contribute/writing-a-loader/)

#### 开始写吧 ~~

* 创建 loader 的目录及模块文件
* 在 webpack 中配置 rule 及 loader 的解析路径，并且要注意 loader 的顺序，这样在 require 指定类型文件时，我们能让处理流经过指定 laoder。
* 遵循原则设计和开发 loader。

```
//解析路径  
resolveLoader: {
        modules: ['node_modules', path.resolve(__dirname, 'loaders')]
    }
```


最简单的loader,source 中加一行代码

```
var loaderUtils = require('loader-utils');

module.exports = function(source) {
    var opts = loaderUtils.getOptions(this) || {};

    console.log(this.context, );
    console.log(this.resource);
    // console.info(opts);
    return `//i am the test line;\n${source}`;
};
```

实现的一个简易的babel-loader

```
var babel = require("@babel/core")
//https://webpack.js.org/api/loaders/#this-callback
module.exports = function (source, inputSourceMap) {
  var babelOptions = {
    presets: ['@babel/preset-env'],
    inputSourceMap: inputSourceMap,
    filename: this.request.split('!')[1].split('/').pop(),
    sourceMaps: true
  }
  var result = babel.transform(source, babelOptions)
  	this.callback(null, result.code, result.map)

	return ;//// always return undefined when calling 	callback()

}
```

- 串联组合中的 loader 并不一定要返回 JS 代码。只要下游的 loader 能有效处理上游 loader 的输出，那么上游的 loader 可以返回任意类型的模块。



#### 还有一些常用 API：

```
this.context：当前处理文件的所在目录，假如当前 Loader 处理的文件是 /src/main.js，则 this.context 就等于 /src。
this.resource：当前处理文件的完整请求路径，包括 querystring，例如 /src/main.js?name=1。
this.resourcePath：当前处理文件的路径，例如 /src/main.js。
this.resourceQuery：当前处理文件的 querystring。
this.target：等于 Webpack 配置中的 Target。
this.loadModule：但 Loader 在处理一个文件时，如果依赖其它文件的处理结果才能得出当前文件的结果时， 就可以通过 this.loadModule(request: string, callback: function(err, source, sourceMap, module)) 去获得 request 对应文件的处理结果。
this.resolve：像 require 语句一样获得指定文件的完整路径，使用方法为 resolve(context: string, request: string, callback: function(err, result: string))。
this.addDependency：给当前处理文件添加其依赖的文件，以便再其依赖的文件发生变化时，会重新调用 Loader 处理该文件。使用方法为 addDependency(file: string)。
this.addContextDependency：和 addDependency 类似，但 addContextDependency 是把整个目录加入到当前正在处理文件的依赖中。使用方法为 addContextDependency(directory: string)。
this.clearDependencies：清除当前正在处理文件的所有依赖，使用方法为 clearDependencies()。
this.emitFile：输出一个文件，使用方法为 emitFile(name: string, content: Buffer|string, sourceMap: {...})。

```


[写一个loader](https://segmentfault.com/a/1190000012990122)
