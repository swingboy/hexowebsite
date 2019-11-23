---
title: js随机数与简单抽奖
date: 2017-12-10 10:23:57
tags: 技术
---
#####
随机数分为三种：真随机数、准随机数、伪随机数。
真随机数是属于大自然的，真随机数只能通过随机的物理过程来获取，实际中是很难的。
准随机数来自于严格的数学方法，理论上可行，但实际上不可行。

伪随机数，就是假的随机数。因为它是根据某种已知的方法获取随机数，本质上已经去除了真正随机的可能。这种方法一旦重复，则获取到的随机数也会重复。

为什么是伪随机数呢？因为通过这种方式得到的“随机数”都是系统根据公式算出来的！既然是算出来的，那么只要输入的参数一致，那么每次获得的随机数就是同一个。输入的参数我们叫做seed。也就是说，如果程序设计者使用了一个预先确定好的seed来生成随机数，如果抽100次奖，那么这100次中奖的都会是同一个人，或者同一批人。更有甚者，虽然每次使用的seed看似不同，但这批seed会让特定的一部分人中奖概率更大！

真随机数是电脑通过对一些没有规律的自然现象量化后得到的，比如热力学噪声、量子现象等。获取这些真随机数的获取代价显然比用时间做个seed高多了。如果大家觉得年会抽奖的算法有必要使用真随机数，也有办法做到，random.org这个网站就提供了一系列通过“大气噪声”获取随机数序列的在线API，通过这些API获取的随机数序列都是真随机数。

如何产生随机数：产生随机数的方法有很多种，什么斐波那契法、线性同余法、梅森旋转算法Mersenne twister等。

下面简单介绍下。

说到js 的随机数，不得不提的就是Math.random这个方法了。
该方法可返回介于 [0 ~ 1) 之间的一个随机数。
```
var r = Math.random(); //生成0-1 的随机数。
```
还有其他简单的方法，简单回顾一下。
Math.ceil  //对一个数进行上取整 返回值：返回大于或等于x，并且与之最接近的整数。
注：如果x是正数，则把小数“入”；如果x是负数，则把小数“舍”。

Math.floor
对一个数进行下取整。 返回值：返回小于或等于x，并且与之最接近的整数。
注：如果x是正数，则把小数“舍”；如果x是负数，则把小数“入”。

Math.round  四舍五入取整。

问题: 想求解1-x的随机数
var x = 100;
// 1.0-x之间的随机数：
Math.round(Math.random()*x);


问题：x至y之间的随机数
var x = 100;
var y = 200;
Math.round(Math.random()*(y-x)+x);

还可以通过下面这种方式产生：
var r = +new Date();
var r = (new Date()-0) % 15;



问题：随机产生六位数字
```
var Num = ""; 
for(var i=0;i<6;i++){ 
    Num+= Math.floor(Math.random()*10); 
}
console.log(Num);
```


问题：随机的token 生成方式
```
var r = Math.random().toString(36).slice(2); 
//或者
var r = Math.random().toString(36).slice(7);
console.log(r);
```

but，这样有个问题。
for(var i = 0; i< 20; i++){
    console.log(Math.random().toString(36).slice(2));
}

for(var i = 0; i< 20; i++){
    console.log((+new Date()).toString(36));
}


问题：如果想不要数字和字母呢？
当然是正则了！

```
var r = Math.random().toString(36).replace(/[^a-z]+/g, '');
//或者这样
var r = Math.random().toString(36).slice(2).replace(/\d/g, '');
```

问题：从指定的字符中，抽取某一部分当做随机

```
Array(20).fill().map(function () {
    return (function (str) {
        return str.charAt(Math.floor(Math.random() * str.length));
    }('ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'));

}).join('');
```

或者;
```
function rand(length, current) {
    var str = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz";
    current = current ? current : '';

    return length ? rand(--length, str.charAt(Math.floor(Math.random() * 60)) + current) : current;
}
```
当然上面的办法bijiaolow

还有下面这种办法：
```
/*
** randomWord 产生任意长度随机字母数字组合
** randomFlag-是否任意长度 min-任意长度最小位[固定位数] max-任意长度最大位
*/
 
function randomWord(randomFlag, min, max){
    var str = "",
        range = min,
        arr = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z'];
 
    // 随机产生
    if(randomFlag){
        range = Math.round(Math.random() * (max-min)) + min;
    }

    for(var i=0; i<range; i++){
        pos = Math.round(Math.random() * (arr.length-1));
        str += arr[pos];
    }
    return str;
}

var str = randomWord(true, 3, 32); //生成3-32位随机串
console.log(str)
randomWord(false, 43); //生成43位随机串

```


问题：产生len长度的字符串
```
function generateRandomAlphaNum(len) {
    var rdmString = "";
    for (; rdmString.length < len; rdmString += Math.random().toString(36).substr(2));

    return rdmString.substr(0, len);
}
var str = generateRandomAlphaNum(5);
// console.log(str);
```

看下生成GUID的函数
什么是GUID：
全局唯一标识符（GUID，Globally Unique Identifier）也称作 UUID(Universally Unique IDentifier) 。

GUID是一种由算法生成的二进制长度为128位的数字标识符。GUID 的格式为“xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx”，其中的 x 是 0-9 或 a-f 范围内的一个32位十六进制数。在理想情况下，任何计算机和计算机集群都不会生成两个相同的GUID。

代码：
```
function S4() {
    return (((1 + Math.random()) * 0x10000) | 0).toString(16).substring(1);
}
function NewGuid() {
    return (S4() + S4() + "-" + S4() + "-" + S4() + "-" + S4() + "-" + S4() + S4() + S4());
}
console.log(NewGuid());
```

问题：如果想随机中文呢？
我们知道，中文字符的范围是[\u4e00-\u9fa5]

也就是我们常常写的判断是否中文的方法：
```
if (/^[\u4E00-\u9FA5]+$/.test(content)) {
    console.log('is chinese');
}
```

```

我们转化一下为10进制数字
parseInt("4E00",16) //19968
parseInt("9FA5",16) //40869

40869-19968 = 20901 可以有这么多了！！！ 基本上够用

randomHz = function(){
    eval( "var word=" +  '"\\u' + (Math.round(Math.random() * 20901) + 19968).toString(16)+'"');
    return word;
}
for(i=0;i<100;i++){
    console.log(randomHz());
}
```


附：
下面上几个随机抽奖的小实例，
```
let arr  = [
    {
        name: '奖品1',
        per: 10,
        end: 0,
        times: 0
    },{
        name: '奖品2',
        per: 20,
        times: 0
    },{
        name: '奖品3',
        per: 30,
        times: 0
    },{
        name:'谢谢',
        per: 50,
        times: 0
    }
];

let len = 0;
arr.forEach((item, index)=>{
    len += item.per;
    item.end =len;
});

function draw(){
    let random = parseInt(Math.random() * len);
    for(let i = 0 ; i< arr.length; i++){
        if(random < arr[i].end){
            // console.log(arr[i].name);
            arr[i].times ++ ;
            break;
        }
    }
}

// draw();

for(var j = 0; j < 100000; j++){
    draw();
}

arr.forEach(function(item){
    console.log(item.name, item.times);
});
```


```

var gifts = [
    {
        "name":"奖品1",
        "prop":5, 
        "times": 0
    },{
        "name":"奖品2",
        "prop":10,
        "times": 0
    },{
        "name":"奖品3",
        "prop":40,
        "times": 0
    },{
        "name":"奖品4",
        "prop": 45,
        "times": 0
    }
];
//抽奖经典算法
function getResult(arr){
    var leng = 0;
    for(var i=0; i<arr.length; i++){
        leng+=arr[i];
    }
    
    for(var i=0; i<arr.length; i++){
        if(parseInt(Math.random()*leng)<arr[i]){
            gifts[i].times ++;
            return i;
        }else {
            leng -= arr[i];
        }
    }
}

var gArr = [];
for(var i=0; i< gifts.length; i++){
    gArr.push(gifts[i]['prop']);
}

for(var i =0 ; i< 100000; i++){
    getResult(gArr);
} 
gifts.forEach(function(item){
    console.log(item.times);
});

// console.log(gifts[getResult(gArr)]['name']);
```


```
function draw(n = 1){
    let cards = Array(20).fill().map((_, i)=>i+1);
    let ret = [];
    for(let i = 0; i < n; i++){
        let idx = Math.floor(cards.length * Math.random());
        ret.push(...cards.splice(idx, 1));
    }
    return ret;
}
console.log(draw(5));


function draw2(amount, n = 1){
    const cards = Array(amount).fill().map((_, i)=>i+1);

    for(let i = amount - 1; i >= 0; i--){
        let rand = Math.floor((i + 1) * Math.random());
        [cards[rand], cards[i]] = [cards[i], cards[rand]];
    }
    return cards.slice(0, n);
}
console.log(draw2(20, 5));


//-----------------------------------------------------------------------

function * draw3(amount){
    const cards = Array(amount).fill().map((_,i)=>i+1);
    for(let i = amount - 1; i >= 0; i--){
        let rand = Math.floor((i + 1) * Math.random());
        [cards[rand], cards[i]] =  [cards[i], cards[rand]];
        yield cards[i];
    }
}
var drawer = draw3(20);
console.log(Array(5).fill().map(()=>drawer.next().value));

//------------------------------------------------

//draw4
function draw4(list = [], number = 0) {
    list = list.concat()
    let result = []
    while (number--) {
        result.push(list.splice(Math.floor(Math.random()*list.length), 1)[0]);
    }
    return result;
}
console.log(draw4([1,2,3,4,5,6,7,8,9,10], 5));


//------------------------------------------------

var arr = Array(20).join(',').replace(/\,/g, function(a, b, arr){
    //a ：元素，b：下标，c:数组
    var n = +b + 1;
    var m = n + (n == arr.length ? '': '.');

    return m;
}).split('.');

function draw5(){
    var rst = function() {
        var r = Math.floor(Math.random() * arr.length);
        return arr.splice(r, 1);
    };
    return rst;
}
console.log(draw5()(), draw5()(), draw5()(), draw5()(), draw5()());
```



<div>
参见： <a href="https://aotu.io/notes/2016/04/14/math-random/">https://aotu.io/notes/2016/04/14/math-random/</>
</div>