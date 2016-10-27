title: 从IIFE讨论表达式和语句
date: 2016-10-26 23:14:00
description: 
categories: 前端开发
tags: [] 
toc: 
feature: 
---
从Immediately Invoked Function Expression(中文:立即调用的函数表达式)说起,主流写法:
```
;(function(){
    //something
})();

;(function(){
    //something
}());
```
两个分号都属于防御型.函数声明和函数表达式真正的区别是什么呢?
```
function fn(){
    //This is FunctionDeclaration(函数声明)
};

var fn = function(){
   //This is FunctionExpression(函数表达式)
};//这个是完整的比表达式语句

```
我们认知的函数表达式,并不是真正的函数表达式,而是函数表达式语句.从[ECMA-262 5.1](http://www.ecma-international.org/ecma-262/5.1/#sec-13)中是这样表述:
```
FunctionDeclaration :
    function Identifier ( FormalParameterList`opt` ) { FunctionBody }
FunctionExpression :
    function Identifier`opt` ( FormalParameterList`opt` ) { FunctionBody }
```
它们的区别并不是在于是否有标示符(Identifier),而是上下文,我们传统看到的`var name=`只不过是其中一种方式,例如:
```
null,function fn(){
    //something
};
console.log(fn);
//Uncaught ReferenceError: fn is not defined
```
这里我们还要分清,语句(Statements),表达式(Expressions),表达式语句(Expression Statement).通俗点说,语句是语法,表达式是由运算符(Operators)和标识符(Identifiers)组成可计算的最终值(注意`=`并不属于表达式,而属于语句).

而表达式语句是不能以`{`和`function`开始的语句带表达式.详细查看[Expression Statement](http://www.ecma-international.org/ecma-262/5.1/#sec-12.4)
```
ExpressionStatement :
    [lookahead ∉ {{, function}] Expression ;
```
既然弄清楚三者的关系,回头过再来看看IIFE的两个种方式,有什么区别,首先
```
function(){}()
//Uncaught SyntaxError: Unexpected token (
```
这是不被允许的表达式,因为自动分号规则,将自动解析成:
```
function(){};()
```
按照规则来完善表达式,那么可以有很多种方式,例如:
```
null,function(){}();

(function(){}());

!function(){}();
//等等之类.
```
当然,我们需要提醒的是,并不存在真正的实名IIFE,不少文章都在误导:
```
(function test(){
    //test并没有在同层的闭包内被声明
}(),console.log(test));
```
当然,回到题目一开始的两种IIFE的区别从语意理解上的区别是什么呢?
```
;(function(){})();
//等价于;(FunctionExpression)();
;(function(){}());
//;(FunctionExpression());
```
就调用后开始执行而言,并无区别.当然,在知道了,函数表达式和表达式语句的区别后,我们也可以这样写
```
;null,function(){}();//表达式
;var test = (function(){})();//表达式语句
```
理论上,只要函数表达式不以`{`和`function`开始,且符合表达式规则均可[Expressions](http://www.ecma-international.org/ecma-262/5.1/#sec-A.3)且是自运行都是属于IIFE.

需要注意的是像`new`表达式的IIFE是:`new fuction(){}`,后面的`()`如果需要带参也可加.

**而IIFE是仅仅只是表达式而已,并不是表达式语句,更不存在某些文章提到的实名IIFE.类似各种花式IIFE的[测试](https://jsperf.com/iife-different-operator-efficiency),核心原理不过是自运行函数的return 值再处理,也能解释大样本下为什么`+function(){}()`较慢.**

顺带提一句:`eval('({test:0})')`的内括号只是为了变成表达式,而不让解释器识别成`{}`语法块[Block](http://www.ecma-international.org/ecma-262/5.1/#sec-12.1);因此,事实上,`eval('0,{test:0}')`的逗号`,`表达式规则都是同理的.