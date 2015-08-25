深度剖析JavaScript原生异步函数

# Chapter 1：Timer与process.nextTick的内部实现

Timers：
https://github.com/nodejs/node/blob/master/lib/timers.js
https://github.com/nodejs/node/blob/master/src/timer_wrap.cc

nextTick：
https://github.com/nodejs/node/blob/master/src/node.js


# Chapter 2：Be Careful！

+ 任务堆积
+ 非精确计时
+ 线程挂起

# Chapter 3：Tick与JavaScript线程

# Chapter 4：巧用setTimeout

+ 解决setInterval任务堆积问题
+ 实现尾递归优化
+ 模拟异步事件