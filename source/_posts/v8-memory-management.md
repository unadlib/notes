title: V8 内存管理
date: 2016-09-14 09:45:31
description: 
categories: 前端开发
tags: 
  - 浏览器引擎
toc: 
feature: 
---

## 垃圾回收(GC)

### 全局变量
全局对象只会在页面生命周期(页面跳转、页面关闭或浏览器关闭)结束后才会被清理。

### 闭包作用域
函数作用域的变量将在超出作用域时被清理(即退出函数时),对于已经没有任何引用的变量就将被清理。

## 堆栈
基本类型存放在栈内存,基本包装型与引用类型等对象引用均在堆内存.

![image](https://raw.githubusercontent.com/unadlib/notes/master/upload/v8-memory-management0.gif)
## 变量存放
### handle
handle是指向对象的指针，在V8中，所有对象都是通过handle来引用，handle主要用于V8的垃圾回收机制。进一步的，handle分为两种：
* 持久化(Persistent handle)，存放在堆上
* 本地化(Local handle)，存放在栈上
### scope
scope是handle的集合，可以包含若干个handle，这样就无需将每个handle逐次释放，而是直接释放整个scope。
### context
context是一个执行器环境，使用context可以将相互分离的JavaScript脚本在同一个V8实例中运行，而不互相干涉。在运行JavaScript脚本时，需要显示的指定context对象。

![image](https://raw.githubusercontent.com/unadlib/notes/master/upload/v8-memory-management1.gif)
### 分代策略
**新生区，老生区，大对象区，Map区，Code区**

64位环境下的V8引擎的新生代内存大小32MB、老生代内存大小为1400MB，而32位则减半，分别为16MB和700MB

#### 新生区
新生区采用半空间分配策略

![image](https://raw.githubusercontent.com/unadlib/notes/master/upload/v8-memory-management2.gif)

### 标记和清除
* dom节点删除后，dom引用 内存无法释放
* 意外全局（'use strict'）
* gc卡顿
* 内存占用过多
* 匿名函数可以访问父级作用域的变量
* 定时器