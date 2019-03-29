---
title: webpack原理
date: 2019-03-29 20:49:07
tags:
---
# webpack 原理

## webpack核心概念

entry 一个可执行模块或库的入口文件。

chunk 多个文件组成的一个代码块，例如把一个可执行模块和它所有依赖的模块组合和一个 chunk. 

loader 文件转换器，例如把es6转换为es5，scss转换为css。

plugin 插件，用于扩展webpack的功能，在webpack构建生命周期的节点上加入扩展hook为webpack添加功能。  

### webpack 模块？

对比 Node.js 模块，webpack 模块能够以各种方式表达它们的依赖关系，几个例子如下：

ES2015 import 语句 
CommonJS require() 语句 
AMD define 和 require 语句 
css/sass/less 文件中的 @import 语句 
样式(url(...))或 HTML 文件(\<img src=...\>)中的图片链接(image url) 

### 支持的模块类型

webpack 通过 loader 可以支持各种语言和预处理器编写模块。loader 描述了 webpack 如何处理 非 JavaScript(non-JavaScript) 模块，并且在bundle中引入这些依赖。 webpack 社区已经为各种流行语言和语言处理器构建了 loader，包括：

- CoffeeScript
- TypeScript
- ESNext (Babel)
- Sass
- Less
- Stylus

### 事件系统
Webpack的基础组件之一Tapable是为其量身定做的“EventEmitter”，但它不只是单纯的事件中枢，还相应补充了对事件流程的控制能力，增加了如waterfall/series/parallel系列方法，实现了同步/异步、顺序/并行等事件流的控制能力。

- 发布/订阅模式

## webpack构建流程 总括

解析webpack配置参数，合并从shell传入和webpack.config.js文件里配置的参数，生产最后的配置结果。

注册所有配置的插件，好让插件监听webpack构建生命周期的事件节点，以做出对应的反应。 

从配置的entry入口文件开始解析文件构建AST语法树，找出每个文件所依赖的文件，递归下去。 
 
在解析文件递归的过程中根据文件类型和loader配置找出合适的loader用来对文件进行转换。  

递归完后得到每个文件的最终结果，根据entry配置生成代码块chunk。  

输出所有chunk到文件系统。  

- 在构建生命周期中有一系列插件在合适的时机做了合适的事情，比如一些插件的操作。
- Webpack 会在特定的时间点广播出特定的事件


### 1. webpack入口（webpack.config.js和shell options）

从配置文件package.json 和 Shell 语句中读取与合并参数，得出最终的参数；

每次在命令行输入 webpack 后，操作系统都会去调用 ./node_modules/.bin/webpack 这个 shell 脚本。这个脚本会去调用 ./node_modules/webpack/bin/webpack.js 

- 开一个子进程处理 cli

### 2. 用yargs参数解析

```
// webpack/bin/webpack.js => webpack-cli/bin/cli.js
yargs.parse(process.argv.slice(2), (err, argv, output) => {

```

### 3.webpack初始化

（1）构建compiler对象

```
// webpack-cli/bin/cli.js
let lastHash = null;
let compiler;
try {
    compiler = webpack(options);

...
compiler = new Compiler(options.context);

```

- 还会某些阶段注册一些钩子，compiler.hooks.watchRun、beforeRun、done

（2）注册NodeEnvironmentPlugin插件

```
// 在webpack/lib/webpack.js 
new NodeEnvironmentPlugin().apply(compiler);
```

```
//NodeEnvironmentPlugin.js
apply(compiler) {
    compiler.inputFileSystem = new CachedInputFileSystem(
        new NodeJsInputFileSystem(),
        60000
    );
    const inputFileSystem = compiler.inputFileSystem;
    compiler.outputFileSystem = new NodeOutputFileSystem();
    compiler.watchFileSystem = new NodeWatchFileSystem(
        compiler.inputFileSystem
    );
    // 注册beforeRun
    compiler.hooks.beforeRun.tap("NodeEnvironmentPlugin", compiler => {
        if (compiler.inputFileSystem === inputFileSystem) inputFileSystem.purge();
    });
}
```

（3）挂载options中的基础插件，调用WebpackOptionsApply库初始化基础插件。

```
//webpack/lib/webpack.js
compiler.options = options;
new NodeEnvironmentPlugin().apply(compiler);
if (options.plugins && Array.isArray(options.plugins)) {
    for (const plugin of options.plugins) {
        if (typeof plugin === "function") {
            plugin.apply(compiler);
        } else {
            plugin.apply(compiler);
        }
    }
}
compiler.hooks.environment.call();
compiler.hooks.afterEnvironment.call();
//调用WebpackOptionsApply库初始化基础插件。
compiler.options = new WebpackOptionsApply().process(options, compiler);



//  WebpackOptionsApply
class WebpackOptionsApply extends OptionsApply {
	constructor() {
		super();
	}

	/**
	 * @param {WebpackOptions} options options object
	 * @param {Compiler} compiler compiler object
	 * @returns {WebpackOptions} options object
	 */
	process(options, compiler) {
        // 一堆基础插件
        ...
		let ExternalsPlugin;
        new JsonpTemplatePlugin().apply(compiler);
        new FetchCompileWasmTemplatePlugin({
            mangleImports: options.optimization.mangleWasmImports
        }).apply(compiler);
        new FunctionModulePlugin().apply(compiler);
        new NodeSourcePlugin(options.node).apply(compiler);
        new LoaderTargetPlugin(options.target).apply(compiler);
        ....
    }
```

（4） run 开始编译(compiler.run)

```
// webpack/lib/webpack.js
if (
    options.watch === true ||
    (Array.isArray(options) && options.some(o => o.watch))
) {
    const watchOptions = Array.isArray(options)
        ? options.map(o => o.watchOptions || {})
        : options.watchOptions || {};
    return compiler.watch(watchOptions, callback);
}
compiler.run(callback);

```

实例compiler后根据options的watch判断是否启动了watch，如果启动watch了就调用compiler.watch来监控构建文件

```
if (
    options.watch === true ||
    (Array.isArray(options) && options.some(o => o.watch))
) {
    const watchOptions = Array.isArray(options)
        ? options.map(o => o.watchOptions || {})
        : options.watchOptions || {};
    return compiler.watch(watchOptions, callback);
}
```

- 这么分析来，webpack的实际入口是Compiler类的run方法

- 注： 这些事情都是在 webpack/lib/webpack.js做掉


在 run 方法里做了哪些事情了 ？

```
//compiler.js 的 run 方法
...
this.hooks.beforeRun.callAsync(this, err => {
    if (err) return finalCallback(err);

    this.hooks.run.callAsync(this, err => {
        if (err) return finalCallback(err);

        this.readRecords(err => {
            if (err) return finalCallback(err);

            this.compile(onCompiled);
        });
    });
});
...
后面会执行、触发 this.compile (见下面分析)
```

### 5.触发compile

（1）在run的过程中，已经触发了一些钩子：beforeRun->run->beforeCompile（在complie方法中this.hooks.beforeCompile）->compile(this.hooks.beforeCompile回调中)->make->seal(compilation.seal)

- 其中很多钩子已经注册。beforeCompile 在构建complier 时已经完成

（2）构建了Compilation对象

在run()方法中，执行了this.compile（）

this.compile()中创建了compilation 见如下代码：

``` 
// complier.js 的 compile
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
        ...
        //回到hook 里面，可以看看实现
        this.hooks.compile.call(params);
        const compilation = this.newCompilation(params);
        // 这里触发make 分析入口等（见下面）
        this.hooks.make.callAsync(compilation, err => {
            ...
            compilation.finish();
            compilation.seal(err => {
                ...
                this.hooks.afterCompile.callAsync(compilation, err 
                    ...
                    return callback(null, compilation);
                });
            });
        });
    });
}
```

- Compilation负责整个编译过程，包含了每个构建环节所对应的方法。对象内部保留了对compiler的引用。

- 当 Webpack 以开发模式运行时，每当检测到文件变化，一次新的 Compilation 将被创建。

- Compilation很重要！编译生产资源变换文件都靠它。


### 6.addEntry() make 分析入口文件创建模块对象

```
//complier 中的 make钩子
this.hooks.make = new AsyncParallelHook(["compilation"])
```


compile中触发hooks.make(异步)事件并调用addEntry


webpack的make钩子中, tapAsync注册了一个DllEntryPlugin, 就是将入口模块通过调用compilation。（下面有注册代码）

这一注册在Compiler.compile()方法中被执行。

addEntry方法将所有的入口模块添加到编译构建队列中，开启编译流程。

```
//DllEntryPlugin.js
compiler.hooks.make.tapAsync("DllEntryPlugin", (compilation, callback) => {
    compilation.addEntry(
        this.context,
        new DllEntryDependency(
            this.entries.map((e, idx) => {
                const dep = new SingleEntryDependency(e);
                dep.loc = {
                    name: this.name,
                    index: idx
                };
                return dep;
            }),
            this.name
        ),
        this.name,
        callback
    );
});
```

解释下为什么流程走到这里

之前WebpackOptionsApply.process()初始化插件的时候，执行了compiler.hooks.entryOption.call(options.context, options.entry);

```
//WebpackOptionsApply.js
class WebpackOptionsApply extends OptionsApply {
    process(options, compiler) {
        ...
        //这句是关键，注册entryOption钩子
        new EntryOptionPlugin().apply(compiler);
		compiler.hooks.entryOption.call(options.context, options.entry);
    }
}
```

但是entryOption钩子 又是在哪里注册的呢？   
答：在EntryOptionPlugin中

```
compiler.hooks.entryOption.tap("EntryOptionPlugin", (context, entry) => {
    if (typeof entry === "string" || Array.isArray(entry)) {
        itemToPlugin(context, entry, "main").apply(compiler);
    } else if (typeof entry === "object") {
        for (const name of Object.keys(entry)) {
            itemToPlugin(context, entry[name], name).apply(compiler);
        }
    } else if (typeof entry === "function") {
        new DynamicEntryPlugin(context, entry).apply(compiler);
    }
    return true;
});
```

itemToPlugin => new SingleEntryPlugin(context, item, name) 
 => 

```
//SingleEntryPlugin
compilation.dependencyFactories
```

=> compilation.addEntry(context, dep, name, callback);(这就到了构建莫板块)

流弊~~~~

各种插件穿梭~~~


### 7. 构建模块

compilation.addEntry中执行 _addModuleChain() (下面有代码)这个方法主要做了两件事情。

一是根据模块的类型获取对应的模块工厂并创建模块，二是构建模块。

```
//Compilation.js
addEntry(context, entry, name, callback) {
    const slot = {
        name: name,
        request: entry.request,
        module: null
    };
    this._preparedEntrypoints.push(slot);
    this._addModuleChain(
        context,
        entry,
        module => {
            this.entries.push(module);
        },
        (err, module) => {
            if (err) {
                return callback(err);
            }

            if (module) {
                slot.module = module;
            } else {
                const idx = this._preparedEntrypoints.indexOf(slot);
                this._preparedEntrypoints.splice(idx, 1);
            }
            return callback(null, module);
        }
    );
}

=> addModule(module, cacheGroup) => compilation.addModuleDependencies => 
后面是对于各个模块的解析等
```

然后通过 ModuleFactory.create方法创建模块，比如创建了NormalModual(里面有loaders等)

然后调用里面的doBuild方法=》 runLoaders() (在这里面有个而loader-runner。这是webpack的loader运行器。) 

打个断点

插播一下loader的运行总体流程

- 使用loaderResolver解析loader模块路径
- 根据rule.modules创建RulesSet规则集
- 使用loader-runner运行loader

继续断点~

然后对模块使用的loader进行加载。调用 acorn 解析经 loader 处理后的源文件生成抽象语法树 AST。遍历 AST，构建该模块所依赖的模块

```
//NormalModual.js
//产生source，生成语法树
this._source = this.createSource(
    this.binary ? asBuffer(source) : asString(source),
    resourceBuffer,
    sourceMap
);
this._ast =
    typeof extraInfo === "object" &&
    extraInfo !== null &&
    extraInfo.webpackAST !== undefined
        ? extraInfo.webpackAST
        : null;
```



```
//NormalModual.js
//调用相应的loader对resource进行加工，生成一段js代码后交给acorn解析生成AST.所以不管是css文件，还是jpg文件，还是html模版，最终经过loader处理会变成一个module：一段js代码。
return this.doBuild(options, compilation, resolver, fs, err => {
    //回调里面解析ast
    try {
        // 有个Parse类，本质还是acorn
        const result = this.parser.parse(
            this._ast || this._source.source(),
            {
                current: this,
                module: this,
                compilation: compilation,
                options: options
            },
            (err, result) => {
                if (err) {
                    handleParseError(err);
                } else {
                    handleParseResult(result);
                }
            }
        );
        if (result !== undefined) {
            // parse is sync
            handleParseResult(result);
        }
    } catch (e) {
        handleParseError(e);
    }
});
```


#### 使用acorn生成AST，并遍历AST收集依赖

webpack使用acorn解析每一个经loader处理过的source，并且成AST

```
const ast = acornParser.parse(source, {
    ranges: true,
    locations: true,
    ecmaVersion: 2019,
    sourceType: "module",
    onComment: comments
});

```

- 调用 loaders 对模块的原始代码进行编译，转换成标准的JS代码
- 调用 acorn 对JS代码进行语法分析，然后收集其中的依赖关系。每个模块都会记录自己的依赖关系，从而形成一颗关系树


### 8. 封装构建结果（seal）

webpack 会监听 seal事件调用各插件对构建后的结果进行封装，要逐次对每个 module 和 chunk 进行整理，生成编译后的源码，合并，拆分，生成 hash。webpack会根据不同的插件，如MinChunkSizePlugin,LimitChunkCountPlugin 将不同的module整理到不同的chunk里，每个chunk最终对应一个输出文件。 同时这是我们在开发时进行代码优化和功能添加的关键环节。

然后通过Template生成结果代码

插播一下Template

- Template是用来生成结果代码的。webpack中Template有四个子类：
- MainTemplate.js 用于生成项目入口文件  
- ChunkTemplate.js 用于生成异步加载的js代码  
- ModuleTemplate.js 用于生成某个模块的代码  
- HotUpdateChunkTemplate.js


*Template 也是 继承自Tapable，有render方法等*

然后通过这些模板把chunk生产__webpack_require__的格式。

### 9. 输出资源（emit）

webpack会在Compiler的emitAssets方法里把compilation.assets里的结果写到输出文件里，在此前会先创建输出目录(把Assets输出到output的path中)。


```
emitAssets(compilation, callback) {
    let outputPath;
    const emitFiles = err => {
        if (err) return callback(err);

        asyncLib.forEachLimit(
            compilation.assets,
            15,
            (source, file, callback) => {
                ....
                if (targetFile.match(/\/|\\/)) {
                    const dir = path.dirname(targetFile);
                    this.outputFileSystem.mkdirp(
                        this.outputFileSystem.join(outputPath, dir),
                        writeOut
                    );
                } else {
                    writeOut();
                }
            },
            err => {
                if (err) return callback(err);

                this.hooks.afterEmit.callAsync(compilation, err => {
                    if (err) return callback(err);

                    return callback();
                });
            }
        );
    };

    this.hooks.emit.callAsync(compilation, err => {
        if (err) return callback(err);
        outputPath = compilation.getPath(this.outputPath);
        this.outputFileSystem.mkdirp(outputPath, emitFiles);
    });
}
```

- 当要开发一些自定义的 插件要输出一些结果时，把文件放入compilation.assets里即可。

### and more

#### memory-fs 内存文件系统
是node原生fs模块内存版(in-memory)的完整功能实现。相比于从磁盘读写数据，memory-fs是内存缓存和快速数据处理的完美替代方案。

- webpack 通过自己实现的memory-fs将 bundle.js 文件打包到了内存中，访问内存中的代码文件也就更快，也减少了代码写入文件的开销。

- memory-fs 是 webpack-dev-middleware 的一个依赖库，webpack-dev-middleware 将 webpack 原本的outputFileSystem 替换成了MemoryFileSystem实例，这样代码就将输出到内存中。
 

```
//webpack-dev-middleware 中该部分代码
//webpack-dev-middleware/lib/Shared.js
var isMemoryFs = !compiler.compilers && compiler.outputFileSystem instanceof MemoryFileSystem;
if(isMemoryFs) {
    fs = compiler.outputFileSystem;
} else {
    fs = compiler.outputFileSystem = new MemoryFileSystem();
}
```

- 首先判断当前 fileSystem 是否已经是 MemoryFileSystem 的实例，如果不是，用 MemoryFileSystem 的实例替换 compiler 之前的 outputFileSystem。这样 bundle.js 文件代码就作为一个简单 javascript 对象保存在了内存中，当浏览器请求 bundle.js 文件时，devServer就直接去内存中找到上面保存的 javascript 对象返回给浏览器端。


关于webpack-dev-server 的核心内容

- webpack, 负责编译代码
- webpack-dev-server，主要提供了 in-memory 内存文件系统，他会把webpack的outputFileSystem 替换成一个 inMemoryFileSystem，并且拦截全部的浏览器请求，从这个文件系统中把结果取出来返回。
- express，作为服务器

#### 总结

webpack的主要编译都按照下面的钩子调用顺序执行。

- Compiler:beforeRun 清除缓存
- Compiler:run 注册缓存数据钩子
- Compiler:beforeCompile
- Compiler:compile 开始编译
- Compiler:make 从入口分析依赖以及间接依赖模块，创建模块对象
- Compilation:buildModule 模块构建
- Compiler:normalModuleFactory 构建
- Compilation:seal 构建结果封装， 不可再更改
- Compiler:afterCompile 完成构建，缓存数据
- Compiler:emit 输出到dist目录
- 一个 Compilation 对象包含了当前的模块资源、编译生成资源、变化的文件等。


***

and~~~

- Compilation 对象也提供了很多事件回调供插件做扩展。

- Compilation中比较重要的部分是assets,如果我们要借助webpack来生成文件,就要在assets上添加对应的文件信息。


参考：[干货！撸一个webpack插件(内含tapable详解+webpack流程)](https://segmentfault.com/a/1190000017013855?utm_source=tag-newest)

