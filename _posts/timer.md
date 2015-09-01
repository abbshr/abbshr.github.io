[提纲] - 深度剖析JavaScript原生异步函数

# Chapter 1：Timer与process.nextTick的内部实现

Timers：
https://github.com/nodejs/node/blob/master/lib/timers.js
https://github.com/nodejs/node/blob/master/src/timer_wrap.cc

nextTick：
https://github.com/nodejs/node/blob/master/src/node.js


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

### 事件循环
无论是Node环境还是浏览器环境, 都存在这么个事件循环, 每次循环被称作一个**Tick**, 跑在主线程里, 当代码都执行完之后, 会去异步事件队列取任务执行, 并且每次取一个.

# Chapter 4: 同步事件与异步事件

### 异步事件

上面提到的计时器触发的事件就属于异步事件. 所有的异步事件触发后都会放入异步事件的队列里, 等到next tick里执行. 所有的I/O操作、消息通知、定时器等系统级别的都是异步事件.

### 同步事件

同步事件没有异步事件那么复杂, 他们的执行会在this tick而不用等到next tick, 就是触发即执行. 同步事件是Event Driven Programming(Pub/Sub设计模式)的一个实现, 不需要额外线程或其他底层机制的辅助便可实现, 如Node里的EventEmitter, 触发的都是同步事件, 触发原理就是普通的JavaScript代码遍历一次事件名字对应的数组中保存的回调函数, 很显然这和其他代码并没有什么两样, 所以是同步的.

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

你会发现你的单机并没有生效, 而是在整个循环结束之后打印响应次数的SYNC. 不是说click是同步吗, 为什么会出现这种情况? 其实第一个实验我们忽略了一点: DOM.

我们谈到同步事件时讲的是**DOM**上的事件, 第一个`a.click()`是DOM API提供的, 或者说是和Node EventEmitter的emit()一样的方式实现的, 由于和普通代码没区别, 肯定在循环中被处理了.

然而真正的单机事件是由操作系统触发, 然后传递给浏览器的, 这属于系统级的事件, 所以是异步的. 当浏览器拿到单击事件通知, 会在底层的回调里调用DOM API的click()方法来实现DOM click. 这和Node里的异步/同步事件协作方式别无二致, 所有的异步函数都是靠底层异步事件通知上层的同步事件, 再由同步事件触发当前环境的回调函数.

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

现在就可以看到灰度的渐变过程了, 这也是Worker出现前早先解决密集任务下cpu负载过高问题的手段.
