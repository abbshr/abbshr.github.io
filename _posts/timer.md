[提纲] - 深度剖析JavaScript异步函数

# Chapter 1：Node.js环境下的异步函数

## Timer

Node对两个Timer的超时时间做了个小trick, 任何大于`TIMEOUT_MAX`小于1ms的超时都被视为1ms.

`setTimeout`和`setInterval`在Node下的封装基本上一样, 这里单拿前者举例.

```js
exports.setTimeout = function(callback, after) {
  after *= 1; // coalesce to number or NaN
  // 保证setTimeout永远会延时执行
  if (!(after >= 1 && after <= TIMEOUT_MAX)) {
    after = 1; // schedule on next tick, follows browser behaviour
  }
  // ...
  var ontimeout = callback;
  timer._onTimeout = ontimeout;
  // ...
  exports.active(timer);

  return timer;
};
```

## setImmediate

官方文档对`setImmediate`并不准确, 基本上让所有没读过源码(包括读得不仔细)的人对其产生极大误解.

主要来自`setImmediate`和`setTimeout(0)`谁先谁后的问题,

```js
exports.setImmediate = function(callback, arg1, arg2, arg3) {
  var i, args;
  var len = arguments.length;
  var immediate = new Immediate();

  L.init(immediate);
  // ...
  // 这里是setImmediate能执行的关键, c++作用域里会检查`_needImmediateCallback`
  // c++那部分代码太长我就不贴了, 全在src/node.cc里, 自己去看.
  if (!process._needImmediateCallback) {
    process._needImmediateCallback = true;
    process._immediateCallback = processImmediate;
  }
  // ...
  return immediate;
};
```

## process.nextTick

这个在异步函数里优先级最高大家都知道, 属于`idle`观察者也清. 代码在`src/node.js`里实现.

代码很长, 不多说, 所有`process.nextTick`堆积的任务都会在事件循环的next tick(后面讲)里一口气执行.

这里还有一个重点: `_tickCallback`函数是`idle`观察者在next tick里的主回调函数:

```js
function _tickCallback() {
  var callback, args, tock;

  do {
    while (tickInfo[kIndex] < tickInfo[kLength]) {
      tock = nextTickQueue[tickInfo[kIndex]++];
      callback = tock.callback;
      args = tock.args;
      // Using separate callback execution functions helps to limit the
      // scope of DEOPTs caused by using try blocks and allows direct
      // callback invocation with small numbers of arguments to avoid the
      // performance hit associated with using `fn.apply()`
      if (args === undefined) {
        doNTCallback0(callback);
      } else {
        switch (args.length) {
          case 1:
            doNTCallback1(callback, args[0]);
            break;
          case 2:
            doNTCallback2(callback, args[0], args[1]);
            break;
          case 3:
            doNTCallback3(callback, args[0], args[1], args[2]);
            break;
          default:
            doNTCallbackMany(callback, args);
        }
      }
      if (1e4 < tickInfo[kIndex])
        tickDone();
    }
    tickDone();
    _runMicrotasks();
    emitPendingUnhandledRejections();
  } while (tickInfo[kLength] !== 0);
}
```

当整个队列清空后, 继续向下执行`_runMicrotasks`, **Microtask是什么?**,
我们暂且不理会它, 先来看与其相关的代码:

```js
function scheduleMicrotasks() {
  if (microtasksScheduled)
    return;

  nextTickQueue.push({
    callback: runMicrotasksCallback,
    domain: null
  });

  tickInfo[kLength]++;
  microtasksScheduled = true;
}

function runMicrotasksCallback() {
  microtasksScheduled = false;
  _runMicrotasks();

  if (tickInfo[kIndex] < tickInfo[kLength] ||
      emitPendingUnhandledRejections())
    scheduleMicrotasks();
}
```

注意到microtask在调度时跟process.nextTick的行为差不多, 也是将任务压到`nextTickQueue`里, 然后在next tick里一次性处理掉.

process.nextTick初始化过程的其他部分代码:
```js
startup.processNextTick = function() {
    // 保存nextTick任务的队列
    var nextTickQueue = [];
    var pendingUnhandledRejections = [];
    var microtasksScheduled = false;

    // ...
    function emitPendingUnhandledRejections() {
      var hadListeners = false;
      while (pendingUnhandledRejections.length > 0) {
        var promise = pendingUnhandledRejections.shift();
        var reason = pendingUnhandledRejections.shift();
        if (hasBeenNotifiedProperty.get(promise) === false) {
          hasBeenNotifiedProperty.set(promise, true);
          if (!process.emit('unhandledRejection', reason, promise)) {
            // Nobody is listening.
            // TODO(petkaantonov) Take some default action, see #830
          } else {
            hadListeners = true;
          }
        }
      }
      return hadListeners;
    }

    addPendingUnhandledRejection = function(promise, reason) {
      pendingUnhandledRejections.push(promise, reason);
      scheduleMicrotasks();
    };
  };
```

## 执行顺序?

上面三个是Node下很常用的用来实现异步函数的工具, 可惜官网并没有对他们的差别给出易懂的解释(我猜测原因是他们默认你充分了解event loop并且熟读libuv源码...).

好了, 这里必须要说的一个例子:

`setImmediate`的表现在新旧版本中的差异:

```coffee
setImmediate ->
  console.log 'immediate-1'
  process.nextTick ->
    console.log 'nextTick-1'

setImmediate ->
  console.log 'immediate-2'
  process.nextTick ->
    console.log 'nextTick-2'
```

我在Node-v0.11.14中的测试结果符合以往的认知:

```coffee
# immediate-1
# nextTick-1
# immediate-2
# nextTick-2
```

按照正常的认知, 因为`process.nextTick`属于idle, 而`setImmediate`属于check, 它积压的任务在每个tick里执行一个.
所以setImmediate的第一个任务执行完后并不会接着执行第二个任务, 而是进入next tick, 执行process.nextTick的任务.

然而在iojs-v3.2中出现了颠覆世界观的一幕:

```coffee
# immediate-1
# immediate-2
# nextTick-1
# nextTick-2
```

输出结果很让人惊讶, 这种情况应该是setImmediate和process.nextTick一样把所有任务在next tick里一气呵成了.
不过并没有见官方文档做出改动, 也没给出合理解释.

## Node context event loop

这里要举另外一个例子, setImmediate和setTimeout的表现.

```js
setImmediate(function() { console.log('immediate'); });
setTimeout(function() { console.log('timeout'); }, 0);
```
哪个先打印出来? 'immediate'? 'timeout'? 可以尝试一下, 实际结果可能是这样的:

```coffee
# 第N次尝试 =>
# timeout
# immediate

# ...

# 第M次尝试 =>
# immediate
# timeout
```

如果`process.nextTick`乱入:

```coffee
process.nextTick ->
  console.log 'nexttick'
setImmediate ->
  console.log 'immediate'
setTimeout ->
  console.log 'timeout'
```

结果又是另一番景象:

```coffee
# nexttick
# timeout
# immediate
```

先从解释第一种情况. 在libuv的event loop代码中有这么一段核心代码. 上面没少提到next tick, event loop的每一次loop被称作一个tick:

```cpp
while (r != 0 && loop->stop_flag == 0) {
    // tick开始时更新当前时间
    uv__update_time(loop);
    // 1. 执行超时的计时器任务
    uv__run_timers(loop);
    // 2. 执行上次推迟执行的I/O回调函数
    ran_pending = uv__run_pending(loop);
    // 3. 执行idle任务
    uv__run_idle(loop);
    // 4. 执行prepare任务
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
    // 5. 开始I/O poll, (网络I/O为epoll(Linux为例), 文件系统I/O等其他异步任务为thread pool).
    // 得到底层通知后, 通常会在本次tick执行I/O回调函数
    uv__io_poll(loop, timeout);
    // 6. 执行check任务
    uv__run_check(loop);
    uv__run_closing_handles(loop);
    // ...
    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }
```

我们着重关注的是步骤`1, 3, 5, 6`, 暂且看做`setTimeout的任务`,`process.nextTick的任务`,`I/O任务`,`setImmediate的任务`.

于是我们得到了观察者的执行顺序: `timer > I/O > check`.

即然这样, 为什么第一个例子结果会不确定呢?

这时Node对setTimeout改动的结果, 记得最开始提到的setTimeout源码中的重点吗?

setTimeout的任务无论如何都会延时入队, 最早也是在1ms之后.
所以对于第一个例子, 由于首次call stack的占用时间没法确定, 在next tick时可能已经过了1ms, 也有可能小于1ms,
所以check和timer的执行先后没法确定.

当我们把process.nextTick加入call stack时, next tick一定最先执行它的任务,
结束后耗时基本上已经超过1ms了, 于是timer的任务先被执行.
最后check的任务得到执行, 这也是为什么会有确定的输出结果.


# Chapter 1.5: macroTask & microTask

紧接上文, 在`process.nextTick`的源码里发现了`MicroTask`这个东西, 当时并没有解释此为何物,
只描述了行为. 因为提起`microTask`就不得不回到Browser环境来.

## Browser context event loop

> 参见 https://html.spec.whatwg.org/multipage/webappapis.html#event-loops

异步任务都需要依赖一个宿主环境, 由它提供一个称之为"事件循环"的机制, 正如Node一样, 浏览器里的async也依赖这么一个event loop.

相比Node, 浏览器里的event loop就简单的多. 这个loop只检查两类queue: macroTaskQueue, microTaskQueue.
分别存放macroTask和microTask.

### macroTask

每个event loop可以有多个macroTask队列, 这类任务和setTimeout任务的行为类似,
都是在每个tick执行队列里的一个任务. 相似的特点还有他们都是在每个tick的开始执行的.

包括: timer task, I/O task

### microTask

每个event loop仅有一个microTask队列, 类似process.nextTick, 一旦执行就一气呵成.
正符合process.nextTick源码里的microtask的行为.

包括: Promise task

### next tick
每个tick的执行流程:

# TODO
> 参见 https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/

### 任务类型
这些任务可以是:

+ dispatch事件
+ 解析HTML
+ 执行回调
+ 非阻塞方式获取外部资源(典型的ajax fetch)
+ 响应DOM树的操作

# Chapter 2：Be Careful！

### `setInterval()`的问题
在定时器的选择上, 有经验的老手会告诉你别用`setInterval`. 为什么?  setInterval的功能是**周期性触发定时事件**, 而不是周期性执行任务. 比如我们想每隔50ms执行一次`syncTask`函数, 直觉上告诉我们这样写:

```coffee
setInterval -> syncTask(), 50
```

但是JavaScript单线程模型告诉你这样不行, 假如syncTask是一个计算量非常大的任务(执行时间远大于50ms), 那么主线程在syncTask结束调用之前是不会交出使用权的, 而计时器的tick是在另一个线程中执行的, 所以计时器依然会生效: 每隔50ms触发定时, 同时将每个任务压入队列等待主线程空闲, 而一旦主线程空闲下来, 队列里的任务将在next tick依次执行掉. 如此往复, 队列里的任务会越积越多, 而我们每隔50ms的预期也就没法实现了. 与其说是不能实现预期功能, 不如说这会带来严重后果: 随着任务的积压, 主线程将马不停蹄的执行syncTask, 这会导致主线程负载过高, 随之而来的就是严重的性能损耗, 无法及时(甚至根本不能)做出响应.

### 无精确计时
这里要说的不包含上面讲的任务积压导致执行周期不精确问题, 而是计时器本身的*bug*. 众所周知CPU都有个时钟周期, 亦作'tick', 基本上计算机的所有工作都依赖时钟, 它也代表一台计算机所能计量的最小时间单位, 一般计算机大概在3~4ms左右.

所以你把代码写成这个样子然后指望它立马执行是不可能的:

```coffee
setTimeout -> console.log 'haha', 0
```

#### 如何证明?

为了证明所言不虚, 我们可以写一个这样的代码, 利用累计任务计算一个平均值:

```coffee
diff = 0
i = 0
test = ->
	if i < 10000
		i++
		initTime = Date.now()
		setTimeout ->
			startTime = Date.now()
			diff += startTime - finishTime
			test()
		, 0
test()

# 计算平均延时
console.log diff / i
```

# Chapter 3：Tick与JavaScript线程

### 单线程的JavaScript
单线程事件驱动确实为JavaScript带来了莫大的优势, 然而不能充分利于CPU也常被人诟病.

在浏览器里, js和UI渲染是互斥的线程, 一旦执行点耗时的逻辑, 页面渲染就卡住了, 必须要等到js跑完. 而这种设计是为了保证交互上逻辑正确性: js触发的UI重绘一定要等到js执行完, 一旦js线程与UI线程并行执行可能导致不可预期的后果.

# Chapter 4: 同步事件与异步事件

### 异步事件

异步事件又称"任务", 即前文提到的macrotask/microtask.

### 同步事件

同步事件即"事件".

没有异步事件那么复杂, 他们的执行会在this tick而不用等到next tick, 就是触发即执行. 同步事件是Event Driven Programming(Pub/Sub设计模式)的一个实现, 不需要额外线程或其他底层机制的辅助便可实现, 如Node里的EventEmitter, 触发的都是同步事件, 触发原理就是普通的JavaScript代码遍历一次事件名字对应的数组中保存的回调函数, 很显然这和其他代码并没有什么两样, 所以是同步的.

除了EventEmitter, 浏览器中的大多数DOM事件(比如click事件)都是同步的, 为此可以在浏览器里做个试验:

```coffee
a = document.querySelector "#a"
a.onclick = -> console.log "SYNC"

for i in [0..100]
	setTimeout ->
		console.log "ASYNC"
	, 0
	a.click()
	a.click()
	console.log i
```

所有的ASYNC会在整个循环结束之后打印, 而SYNC会在每次循环最开始时打印两次, 然后打印当前迭代次数, 这就是同步事件与异步事件的区别.

简记为: **凡是用纯JavaScript实现的事件都是同步的, 只有底层实现的事件才有异步的可能**.

### 异步与同步协同实现Event Driven模型

但你可能会说在一个循环执行完之前单击几次鼠标, 难道会在循环中间打印SYNC ? 也就是这样:

```coffee
a = document.querySelector "#a"
a.onclick = -> console.log "SYNC"

for i in [0..10000]
	# 在循环结束之间单击鼠标
	console.log i
```

你会发现你的单机并没有生效, 而是在整个循环结束之后打印响应次数的SYNC. 不是说click是同步吗, 为什么会出现这种情况?

看了前面的**任务**自然会明白.

我们谈到同步事件时讲的是**DOM**上的事件, 第一个`a.click()`是DOM API提供的, 或者说是和Node EventEmitter的emit()一样的方式实现的, 由于和普通代码没区别, 肯定在循环中被处理了.

然而真正的单机事件是由操作系统触发, 然后传递给浏览器的, 这属于系统级的事件(task), 所以是异步的. 当浏览器拿到单击事件通知, 会在底层的回调里调用DOM API的click()方法来实现DOM click. 这和Node里的异步/同步事件协作方式别无二致, 所有的异步函数都是靠底层异步事件通知上层的同步事件, 再由同步事件触发当前环境的回调函数.

# Chapter 5：巧用setTimeout

### 解决setInterval任务堆积

该怎么写? 首先我们要明确问题出现的根源: 定时器不管当前任务是否执行完毕都会周期性添加任务到队列. 若要解决问题, 就该让定时器在任务执行完毕后在开始计时, 这里使用了另一个计时器setTimeout, 以形似递归的手段解决了问题:

```coffee
task = =>
	setTimeout ->
		syncTask()
		task()
	, 50
task()
```

### 给JavaScript实现尾递归优化

JavaScript的一个遗憾就是不支持尾递归优化, 这给有情怀的函数式追随者泼了一头冷水. 不过好消息是我们可以同过一个小小的hack手段实现"尾递归":`setTimeout`.

因为setTimeout的异步性, 当调用结束时(调用会立即结束), 整个函数会释放掉当前堆栈(但是依情况可能保留context对象), 因此无论"递归"调用多少次都不会出现`'RangeError: Maximum call stack size exceeded'`的错误:

```coffee
# 普通无限递归调用
recu = ->
	console.log 1
	recu()
# => RangeError: Maximum call stack size exceeded

# 使用setTimeout的模拟"尾递归"
recu = ->
	console.log 1
	setTimeout recu, 0
# => 1 1 1 1 .....
```

### 执行流程的调度

因为timer触发的回调永远是在nextTick里执行的, 所以这就给了程序流程重新调度的机会.

比如在页面的某个div上添加了click监听器执行taskA, 然后因为某个需求又在其子元素(比如一个按钮)上添加了一个click监听器执行taskB. 要求在click按钮时先执行taskA, 再执行taskB.

> DOM上的事件模型有一个冒泡过程, 一般都是先从最深层的子节点触发事件, 事件逐层冒泡, 最后才触发最上层的回调, 冒泡是同步的.

```coffee
btn.onclick = (e) ->
	taskB()
div.onclick = (e) ->
	taskA()

# 永远都是先taskB()后taskA()
```

用setTimeout把taskB放到next Tick从而改变了执行流程, 所以先执行taskA了:
```coffee
btn.onclick = (e) ->
	setTimeout taskB, 0
div.onclick = (e) ->
	taskA()
```

再举一个实际的例子. 把输入框里输入的英文实时地变成大写.

```coffee
# 貌似该这么写...
input.onchange = (e) ->
	@value = @value.toUpperCase()
```

可是并没有达到预期的效果: 总是在输入下一个字符时才会把前一个转大写. 这是因为onchange事件总是在输入框的value赋值前触发. 知道了原理我们就可以搞定了:
```coffee
input.onchange = (e) ->
	setTimeout =>
		@value = @value.toUpperCase()
	, 0
```

### 解决密集任务下页面渲染/重绘问题

前面提到的JavaScript单线程带来的问题: 跑脚本时DOM无法及时渲染. 这里有个活生生的例子:

实现背景色随时间的灰度渐变. 还是先来一段理想中的代码:

```coffee
div.background = "rgb(#{i},#{i},#{i})" for i in [0...255]
```

但是循环导致没空渲染UI, 看到的结果就是开始是白色, 突然变成黑色. 这时next tick就能大展身手了:

```coffee
# 把js放到next tick里,先让UI渲染占用线程
liner = (i) ->
	div.background = "rgb(#{i},#{i},#{i})"
	setTimeout liner, 0, ++i
liner 0
```

现在就可以看到灰度的渐变过程了, 这也是Worker出现前早先解决密集任务下cpu负载过高问题的思路: 分割任务.
