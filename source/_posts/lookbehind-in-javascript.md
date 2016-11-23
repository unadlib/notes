title: JavaScript的后行断言
date: 2016-11-20 13:50:31
description: 
categories: 前端开发
tags:
toc: 
feature: 
---

* (?<=pattern)	零宽正向后行断言(zero-width positive lookbehind assertion)
* (?<!pattern)	零宽负向后行断言(zero-width negative lookbehind assertion)

## 后行断言
JavaScript的正则表达式并不支持后行断言（也有叫后顾或反向预查），一直以来有个问题，就是没有正常版的正则规则之一：后行断言(RegExp Lookbehind Assertions)，毕竟在ECMA 262 [5.1](http://www.ecma-international.org/ecma-262/5.1/#sec-15.10.2.8)/[6.0](http://www.ecma-international.org/ecma-262/6.0/#sec-runtime-semantics-canonicalize-ch)/[7.0](http://www.ecma-international.org/ecma-262/7.0/index.html#sec-runtime-semantics-canonicalize-ch)中完全没有提及，在2015年底至2016年初，也有提议在TC39讨论加入ECMA-262草案中，其[原因](http://stackoverflow.com/questions/12273112/will-js-regex-ever-get-lookbehind)`据说是早起ECMAScript3引入正则的时候，ECMAScript3已经相对稳定，参考Perl的正则，当时Perl的也正在实验`。虽然原因有点含糊其辞，但不可能是有人以讹传讹说后行断言（任意模式）性能问题过于占用CPU资源。虽然连ECMA-262 7.0都没有加上标准，但是在2016年年初，[V8引擎](http://v8project.blogspot.com/2016/02/regexp-lookbehind-assertions.html)在4.9(--harmony)以及之后版本均已先行实现，当然也包括从Chrome 49开始在开启about:flags中【实验性JavaScript】即可。

Mac:
```
新建一个 Automater 应用， 然后选择 Run Shell Script 使用open命令并编辑所需要的参数:
open /Applications/Google\ Chrome.app --args --js-flags="--harmony-regexp-lookbehind"
```
Windows:
```
chrome.exe --js-flags="--harmony-regexp-lookbehind"
```
node:
```
node --harmony_regexp_lookbehind
```
直到ES5.1，依然不支持的正则特性（JavaScript高级程序设计）：
 
> 1. 匹配字符串开始的结尾的A和Z锚(但支持以^和$来匹配字符串的开始的结尾)
> 2. 后行断言(但完全支持先行断言)
> 3. 并集和交集类
> 4. 原子组(atomic grouping)
> 5. Unicode支持(单个字符除外，如\uFFFF)
> 6. 命名的捕获组(但支持编号的捕获组)
> 7. s(single,单行)和x(free-spacing,无间隔)匹配模式
> 8. 条件匹配
> 9. 正则表达式注释

## 实现后行断言
[有人用正则匹配实现了一个初级的方法](https://gist.github.com/slevithan/2387872),并可以[合并到xregexp](https://github.com/beaugunderson/xregexp-lookbehind),它利用`?:`规则简单实现了compile/exec/test，以及search/match/replace/split，且不支持非后行断言起始或多组后行断言。

XRegExp的创建者Steven Levithan在[2007年的Blog中提到几个实现方法](http://blog.stevenlevithan.com/archives/mimic-lookbehind-javascript/)，其中捕获组包括`?:`和`reverse->先行断言->reverse`之类的。

ps.在测试过程中偶然发现一个有趣的对比(较早来源[Laruence](http://www.laruence.com/2009/09/27/1123.html)):
```
console.time();/^(t+)+\1$/.exec("tttttttttttttttttttttttttttest");console.timeEnd();
Chrome/54.0.2840.98 default: 1.36e+03ms
Safari/10.0.1       default: 48.115ms
FireFox/49.0        default: 1451ms
```
同样采用NFA类型的正则引擎，JavaScriptCore与V8在这点优化上，差别那么大，有点意外；看来在，backtracking和lazy上应该有区别，有时间好看看它们之间算法；并且如果测试的字符串再长点，Chrome 就直接卡住，并且不会内存溢出，CPU温度直接飙到85摄氏度，Chrome任务管理器的当前页面进程直接100%。