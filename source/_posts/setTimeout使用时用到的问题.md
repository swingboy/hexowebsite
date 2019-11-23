---
title: setTimeout使用时用到的问题
date: 2016-05-30 10:52:41
tags: 技术
---

setTimeout() 方法用于在指定的毫秒数后调用函数或计算表达式。而且方法只执行一次。但是我们在用的时候会有一些问题甚至是误区。
1.try/catch捕捉不到它的错误
看下面：
```
try{
    setTimeout(function(){
        throw new Error("我是异常！”)
    }, 1000);
} catch(e){
    console.log(e);
}
```

这里的try/catch 语句块只捕获setTimeout函数自身内部发生的那些错误。因为setTimeout 异步地运行其回调，所以即使延时设置为0，回调抛出的错误也会直接流向应用程序。

2.执行的时候会被阻塞
看下面：
```
var date = new Date;
setTimeout(function(){
   console.log("时间差：" + (new Date - date));
}, 1000);
while(true) {
if(new Date - date > 3000) break;
}
```
我们期望 console 在 1s 之后出结果，可事实上他却是在 3008ms 之后运行的。这就是 JavaScript 单线程给我们带来的小问题，while循环阻塞了setTimeout 函数的执行。至于说为什么不执行setTimeout，是因为js的工作机制就是是：当线程中没有执行任何同步代码的前提下才会执行异步代码(这里的setTimeout是异步代码)，所以setTimeout的代码只能等js空闲才会执行。所以得等后面while执行完。

3.setTimeout 方法的及时性问题
看下面：
```
var date = new Date, count = 0, timer;
timer = setInterval(function (){
   if(new Date - date > 1000) {
       clearInterval(timer), console.log('执行次数:'  + count);
   }
   count++;
}, 0);
```
测试结果可以看出来 1s 中运行的次数大概在 200次多次。因为什么呢，因为setInterval 和 setTimeout 函数运转的最短周期大概是5ms 左右（并非因为函数作用域的转换消耗了时间，）
这个数值在 HTML规范 中也是有提到的:

```
5. Let timeout be the second method argument, or zero if the argument was omitted.
如果 timeout 参数没有写，默认为 0

7. If nesting level is greater than 5, and timeout is less than 4, then increase timeout to 4.
如果嵌套的层次大于 5 ，并且 timeout 设置的数值小于 4 则直接取 4.
```