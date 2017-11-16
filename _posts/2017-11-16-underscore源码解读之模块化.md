---
layout: post
title: 'underscore源码解读之模块化'
date: '2017-11-16 19:23:32'
# categories: ['underscore源码解读']
description: '什么是模块化？封装的插件如何兼容不同的模块环境？'
#avatarimg: '/assets/images/firebird.png'
showimg: '/assets/images/2017-11-16.png'
tags: ['underscore']
---

**大家在阅读一些源码的过程中，经常会遇到类似这样的结构代码**
```JavaScript
;(function(){
   // ...
   function MyModule() {
      // ...
  }
  // ...
  if(typeof module !== `undefined` && typeof exports === `object`) {
      module.exports = MyModule;
  }
  // ...
  if(typeof define === `function` && define.amd) {
      define(function() { return MyModule; });
  } 
   // ...
})()
```
**想必大家或多或少，无非就是一些什么闭包、模块化之类的作用，下面就此展开解读。**
## 首先介绍什么是模块化
**在封装插件之前，需要考虑什么问题？**
1. 如何安全的包装一个模块的代码？（不污染模块外的任何代码）
2. 如何唯一标识一个模块？
3. 如何优雅的把模块的API暴漏出去？（不能增加全局变量）
4. 如何方便的使用所依赖的模块？

  解决这些问题的同时，产生了CommonJS和AMD、CMD等，其中规范以前两种为主。下面简单介绍用法及区别。
### CommonJS
Node中采用了CommonJS，模块引用使用require，如下：
```
var http = require('http');
```
### AMD
AMD 即Asynchronous Module Definition，中文名是“异步模块定义”的意思。它是一个在浏览器端模块化开发的规范，服务器端的规范是CommonJS。

  模块将被异步加载，模块加载不影响后面语句的运行。所有依赖某些模块的语句均放置在回调函数中。
AMD 是 RequireJS 在推广过程中对模块定义的规范化的产出。

其模块引用方式如下：
```
define(id?,dependencies?,factory);
```
其中，id及依赖是可选的。其与CommonJS方式相似的地方在于factory的内容就是实际代码的内容，下面是一个简单的例子：
```
define(function(){
  var exports = {};
  exports.say = function(){
    alert('hello');
  };
  return exports;
});
```
### CMD
CMD规范，与AMD类似，区别主要在于定义模块和依赖引入的地方。主要实现比如： SeaJS。

```
define(['dep1','dep2'],function(dep1,dep2){
  return function(){};
});
```
require、exports、module通过形参传递给模块，在需要依赖模块时，随时调用require引入即可，实现按需加载。

**AMD:提前执行（异步加载：依赖先执行）+延迟执行
CMD:延迟执行（运行到需加载，根据顺序执行）**

CMD 推崇依赖就近，AMD 推崇依赖前置。看如下代码：
```
// CMD
define(function(require, exports, module) {
  var a = require('./a')
  a.doSomething()
  // ...
  var b = require('./b') // 依赖可以就近书写
  b.doSomething()
  // ... 
})
 
// AMD 
define(['./a', './b'], function(a, b) { // 依赖必须一开始就写好
  a.doSomething()
  // ...
  b.doSomething()
  // ...
})
```

## 下面分析underscore源码中加载模块化相关部分
```
(function() {
  var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
  };
  // ...
  if (typeof exports !== 'undefined') {
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {
    root._ = _;
  }
  // ...
  if (typeof define === 'function' && define.amd) {
    define('underscore', [], function() {
        return _;
    });
  }
}).call(this);
```
**1.  通过function定义构建了一个闭包，call(this)是将function在this对象下调用，以避免内部变量污染到全局作用域。浏览器中，this指向的是全局对象（window对象），将“_”变量赋在全局对象上“root._”，以供外部调用。**

**2. 另外还有一个常用知识点，在这里没有出现。**
```
;(function (){}())
!(function (){})()
```
**立即执行函数前面放一个";"或者"!"是干嘛用的？看到下面这个例子你就知道了**
```
var aa = function (){console.log('我是无辜的')}
(function(){console.log("立即输出")}())

// 看下面代码与上面代码不同之处
var aa = function (){console.log('我是无辜的')};
(function(){console.log("立即输出")}())
```
看到运行结果你有什么想法？发现，上面的代码被当做了一条语句运行了，也就是
`var aa = function (){console.log('我是无辜的')}(function(){console.log("立即输出")}())`

  这就是为什么要加";"或者"!"的原因，**防止上条执行语句没有分号影响下面语句执行**

**3. 如下代码：**
```
if (typeof exports !== 'undefined') {
    if (typeof module !== 'undefined' && module.exports) {
      exports = module.exports = _;
    }
    exports._ = _;
  } else {
    root._ = _;
  }
```
**首先判断是否支持exports导入方式，`typeof module !== 'undefined' && module.exports`这句代码判断CommentJS与CMD，如果加上`define.cmd`则只支持CMD**。
对于`exports = module.exports = _;`与`exports._ = _;`这两句我是这么理解的。`exports._ = _;`是为了兼容老的写法。对于那些进入if条件已经执行完`exports = module.exports = _;`的情况，当执行到`exports._ = _;`时，并不会生效。

  下面介绍下`exports`与`module.exports`的区别：
**1. exports是module.exports的辅助方法。模块最终返回module.exports给调用者，而不是exports。**
**2. exports所做的事情是收集属性，如果module.exports当前没有任何属性的话，exports会把这些属性赋予module.exports。**
**3. 如果module.exports已经存在一些属性的话，那么exports中所用的东西都会被忽略。**

```
// aa.js
module.exports = '测试';
exports.name = function() {
    console.log('My name is XX');
};

// 下面引入aa.js文件
var aa= require('./aa.js');
aa.name(); // TypeError: Object 测试 has no method 'name'
```
**aa模块完全忽略了exports.name，然后返回了一个字符串'测试'。通过上面的例子，你可能认识到你的模块不一定非得是模块实例（module instances）。可以是任何合法的JavaScript对象 - boolean，number，date，JSON， string，function，array和其他。模块可以是任何你赋予module.exports的值。如果你没有明确的给module.exports设置任何值，那么exports中的属性会被赋给module.exports中，然后并返回它。**

**4. 如下代码**
```
if (typeof define === 'function' && define.amd) {
    define('underscore', [], function() {
      return _;
    });
  }
```
**只是为了兼容加载AMD 规范，几乎每个插件源码都是这样引入，固定格式**

### 结尾
**以上只是我对对underscore源码阅读的一点见解，或许也会有理解错误的地方，欢迎指正。在doT.js的源码中同样的模块功能中发现了一些不同之处，很有意思，大家可以思考下这种使用方式。代码如下：**

```
(function(){
   // ...
   (function(){ return this || (0,eval)('this'); }()).doT = doT;

}())
```
**1. 这里的`(0,eval)("this")`什么意思？为什么不用`window.doT`或者`eval('this')`呢**
**2. 另外`(function(){}())`与`(function(){})()`又有什么区别呢**


