title: 从setTimeout聊起
date: 2017-08-27 14:55:00
description:
categories: 前端开发
tags:
  - JavaScript
toc:
feature:
---

### 测试 JavaScript 环境
| 宿主环境    |    版本   |
| :-------- | --------:|
| Chrome    |  60.0.3112.113 |
| Safari    |   10.1.2 |
| Firefox   |     55.0.3|
| Node.js   |    7.10.0 |

当然高精度计时也是必须的:
```javascript
function getTime(start) {
    var time;
    if (typeof window === 'undefined') {
        time = process.hrtime();
        if (start) {
            var diff = process.hrtime(start);
            time = diff[0] * 1e3 + diff[1] / 1e6;
        }
    } else {
        if (typeof window.performance === 'undefined') {
            time = +new Date();
        } else {
            time = performance.now();
        }
        if (start) {
            time -= start;
        }
    }
    return time;
}
```
### 定时器(Timer)

>提问：是否存在低于1ms延迟的定时器？

最有可能被首先提出的方案有:
```
var now = getTime();setTimeout(_=>{var diff = getTime(now);console.log(diff)});
```
测试结果:

| 宿主环境    |    ms   |
| :-------- | --------:|
| Chrome    |  1.303 |
| Safari    |   1.232 |
| Firefox   |     0.465|
| Node.js   |    1.643 |

显然除了Firefox,其他环境下,setTimeout(fn)或者setTimeout(fn,0)都是超过1ms延迟。

首先先来看看W3C关于定时器的API [HTML5 Web application APIs - Timer](https://www.w3.org/TR/2011/WD-html5-20110525/timers.html)运行setTimeout函数的标准步骤:

---
1. Let handle be a user-agent-defined integer that is greater than zero that will identify the timeout to be set by this call.
2. Add an entry to the list of active timeouts for handle.
3. Get the timed task handle in the list of active timeouts, and let task be the result.
4. Get the timeout, and let timeout be the result.
5. If the currently running task is a task that was created by the setTimeout() method, and timeout is less than 4, then increase timeout to 4.
6. Return handle, and then continue running this algorithm asynchronously.
7. If the method context is a Window object, wait until the Document associated with the method context has been fully active for a further timeout milliseconds (not necessarily consecutively).
    Otherwise, if the method context is a WorkerUtils object, wait until timeout milliseconds have passed with the worker not suspended (not necessarily consecutively).
    Otherwise, act as described in the specification that defines that the WindowTimers interface is implemented by some other object.
8. Wait until any invocations of this algorithm started before this one whose timeout is equal to or less than this one's have completed.
9. Optionally, wait a further user-agent defined length of time.
10. Queue the task task.
---
    (9).This is intended to allow user agents to pad timeouts as needed to optimise the power usage of the device. For example, some processors have a low-power mode where the granularity of timers is reduced; on such platforms, user agents can slow timers down to fit this schedule instead of requiring the processor to use the more accurate mode with its associated higher power usage.


#### 1. Promise替代方案:
```javascript
new function(){
    var setPromiseTimeout = function(handle){
        new Promise(resolve=>resolve())
        .then(_=>handle())
    };
    if (typeof window === 'undefined') {
        global.setPromiseTimeout = setPromiseTimeout;
    }else {
        window.setPromiseTimeout = setPromiseTimeout;
    }
}
````
```javascript
var now = getTime();setPromiseTimeout(_=>{var time = getTime(now);console.log(time)});
```
测试结果:

| 宿主环境    |    ms   |
| :-------- | --------:|
| Chrome    |  0.160 |
| Safari    |   0.147 |
| Firefox   |     0.545|
| Node.js   |    0.510 |

#### 2. postMessage替代方案:
不过早在2010年，David Baron提出利用postMessage来实现[setTimeout with a shorter delay](https://dbaron.org/log/20100309-faster-timeouts)
```javascript
    // Only add setZeroTimeout to the window object, and hide everything
    // else in a closure.
    (function() {
        var timeouts = [];
        var messageName = "zero-timeout-message";

        // Like setTimeout, but only takes a function argument.  There's
        // no time argument (always zero) and no arguments (you have to
        // use a closure).
        function setZeroTimeout(fn) {
            timeouts.push(fn);
            window.postMessage(messageName, "*");
        }

        function handleMessage(event) {
            if (event.source == window && event.data == messageName) {
                event.stopPropagation();
                if (timeouts.length > 0) {
                    var fn = timeouts.shift();
                    fn();
                }
            }
        }

        window.addEventListener("message", handleMessage, true);

        // Add the one thing we want added to the window object.
        window.setZeroTimeout = setZeroTimeout;
    })();
```
```javascript
var now = getTime();setZeroTimeout(_=>{var time = getTime(now);console.log(time)});
```

测试结果:

| 宿主环境    |    ms   |
| :-------- | --------:|
| Chrome    |  0.562 |
| Safari    |   0.552 |
| Firefox   |     0.549|
| Node.js   |    N/A |

从实际实现效果上看，`Promise`方案似乎会比`postMessage`方案更理想一些。

### setTimeout执行顺序
根据标准,setTimeout定时时间参数应该是整数，当然若参数非整数则将自动转为整数，当小于4ms时，根据标准应该是定时时间为4ms，实际情况如何？

接下来通过这个函数来测试执行无其他同步代码顺序阀值:
```javascript
function test(time){
    setTimeout(function(){
        console.log(1)
    },time);
    setTimeout(function(){
        console.log(-1)
    });
}
```

测试结果:

| 宿主环境    |    ms   |
| :-------- | --------:|
| Chrome    |  2 |
| Safari    |   2 |
| Firefox   |     1|
| Node.js   |    N/A |

在实际测试中Node.js处于一个随机分配情况。

#### setTimeout执行顺序的坑点

```javascript
setTimeout(function(){
    console.log(100)
},100);
for(var i=0,arr=[];i<10e6;i++){arr.push(i)}
setTimeout(function(){
    console.log(1)
})
```
由于Javascript执行是单线程，因此，当代码同步执行时阻塞耗时超过之前已队列的定时器的定时时间则优先执行。


```javascript
var time1=performance.now();
setTimeout(function(){
    var time4=performance.now()-time1;
    console.log(2000,time4);
    for(var a=0;a<10e9;a++){a=a+1;}
    var time2=performance.now()-time1;
    console.log('slow run',time2);
},2000);
setTimeout(function(){
    var time3=performance.now()-time1;
    console.log(1000,time3);
},1000);
setTimeout(function(){
    var time5=performance.now()-time1;
    console.log(4000,time5);
},4000);
setTimeout(function(){
    var time6=performance.now()-time1;
    console.log(10000,time6);
},10000);
setTimeout(function(){
    var time7=performance.now()-time1;
    console.log(15000,time7);
},15000);
```
当定时器队列中代码被运行时，耗时超过队列中的任何定时任务时间则全部执行，但进入队列其他未超时的队列中定时任务则保持正常。


### setTimeout一些特点

#### 函数带参数
```
var arg=['foo','bar'];
setTimeout(function(...arg){console.log(arg)},0,...arg);//IE9及更早版本不支持
setTimeout(function(...arg){console.log(arg)}.bind(this,...arg),0);
```
#### 自调用函数
```javascript
setTimeout(function test(){console.log(test)});
setTimeout(function(){console.log(arguments.callee)});
```
#### 代码字符串
```javascript
new function(){
    setTimeout("console.log(this)");//this指向全局
}
```
#### setTimeout()返回值
```javascript
console.log(toString.call(setTimeout(function(){},0))); // Node.js:[object Object]
console.log(typeof setTimeout(function(){},0)); // 浏览器: number/返回timeoutID是一个正整数，表示定时器的编号
```
需要注意的是setTimeout()和setInterval()共用一个编号池,且定时器编号在一个沙盒环境下运行时为一个独立的编号池，编号整数自动递增。

如果需要一个稳定的重复定时器:
```javascript
function setNewInterval(fn,timer){
    setTimeout(function() {
        fn();
        setTimeout(arguments.callee, timer)
    }, timer);
}
```

一个纳秒定时器(性能不考虑的情况下):
```javascript
function setNewTimeout(fn,timer){
    var start = getTime();
    setTimeout(function(){
        var time = getTime(start);
        fn();
        console.log(time);
    }, 0);
    while (getTime(start) < timer) {};
}
```


延伸阅读：
1. [Tasks, microtasks, 队列和执行顺序](https://shijianan.com/2017/03/18/tasks-microtasks-queue-schedule/)
2. [Promise的队列与setTimeout的队列有何关联](https://www.zhihu.com/question/36972010)
3. [细说setTimeout/setImmediate/process.nextTick的区别](http://zyy1217.com/2017/03/22/%20细说setTimeout:setImmediate:process.nextTick的区别/)

