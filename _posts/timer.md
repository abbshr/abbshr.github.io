[提纲] - 深度剖析JavaScript原生异步函数

# Chapter 1：Timer与process.nextTick的内部实现

Timers：
https://github.com/nodejs/node/blob/master/lib/timers.js
https://github.com/nodejs/node/blob/master/src/timer_wrap.cc

nextTick：
https://github.com/nodejs/node/blob/master/src/node.js


# Chapter 2：Be Careful！

+ 任务堆积(何时产生?)
+ 非精确计时(如何计量?)

# Chapter 3：Tick与JavaScript线程

+ 单线程的劣势(UI渲染,重绘)
+ 线程挂起(谁干的?)
+ event loop(tick unit)
+ 资源释放

# Chapter 4: 同步事件与异步事件

+ 大多数DOM事件(sync-event, trigger right now)
+ IO操作/消息通知/定时器(async-event, queued before next tick)

# Chapter 5：巧用setTimeout

+ 解决setInterval任务堆积问题
+ 实现尾递归优化
+ 封装异步事件
+ 调度程序执行流程
+ 改变事件回调的执行顺序
+ 解决密集任务下cpu负载过高问题
+ 解决密集任务下的页面渲染/重绘问题
