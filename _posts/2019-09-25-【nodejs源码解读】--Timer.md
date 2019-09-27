---
layout:     post   				    # 使用的布局（不需要改）
title:      【nodejs源码解读】--Timer 				# 标题 
subtitle:   Timer源码javascript代码 #副标题
date:       2019-09-25 				# 时间
author:     daydreamgirl 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - nodejs源码
---

# Timer源码解读顺序
---
layout:     post   				    # 使用的布局（不需要改）
title:      【nodejs源码解读】--Timer 				# 标题 
subtitle:   Timer源码javascript代码 #副标题
date:       2019-09-25 				# 时间
author:     daydreamgirl 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - nodejs源码
---

## 依赖的数据结构模块
### 双向循环链表
在NodeJs中几乎所有的网络I/O请求都会设置timeout来控制socket连接超时的情况,这里会大量使用到setTimeout接口，会频繁的有增删操作，链表插入和删除元素的时间复杂度是O(1)，所以这块用双向循环链表来提高Timer模块的性能。timeout计时器插入计时器列表用的是时间轮算法，给相同 ms 级的 timeout 任务共用了一个 timeWrap，相同时间的任务分配在同一个链表，使计时任务的调度和新增的复杂度都是 O(1)， 也达到高效复用了同一个 timeWrap。

链表源码在lib/internal/linkedlist.js
```
'use strict';

function init(list) {
  list._idleNext = list;
  list._idlePrev = list;
}

// Show the most idle item.
function peek(list) {
  if (list._idlePrev === list) return null; //如果该节点的前驱指针指向本身，说明没有前置节点，返回null
  return list._idlePrev; //返回该节点的前一个节点
}

// Remove an item from its list.
function remove(item) {
  if (item._idleNext) {
    // 如果item存在后继元素，修改item的后继元素的前驱元素为itme的前驱元素
    item._idleNext._idlePrev = item._idlePrev;
  }

  if (item._idlePrev) {
    // 如果item存在前驱元素，修改item的前驱元素的后继元素为item的后继元素
    item._idlePrev._idleNext = item._idleNext;
  }
  // 把item从原来的链表中孤立
  item._idleNext = null;
  item._idlePrev = null;
}

// Remove an item from its list and place at the end.
function append(list, item) {
  // 如果item还存在其他的链表中，从原来的链表中删除item
  if (item._idleNext || item._idlePrev) {
    remove(item);
  }

  // Items are linked  with _idleNext -> (older) and _idlePrev -> (newer).
  // Note: This linkage (next being older) may seem counter-intuitive at first.
  item._idleNext = list._idleNext; // item的后继元素设置为list的后继元素
  item._idlePrev = list; // item的前驱元素设置为list

  // The list _idleNext points to tail (newest) and _idlePrev to head (oldest).
  list._idleNext._idlePrev = item; // list原本的后继元素的前驱元素设置为item
  list._idleNext = item; // list新的后继元素设置为item
}

function isEmpty(list) {
  return list._idleNext === list;
}

module.exports = {
  init,
  peek,
  remove,
  append,
  isEmpty
};

```
### 最小堆
最小堆：父结点的键值总是小于或等于任何一个子节点的键值。

最小堆源码在lib/internal/priority_queue.js
## setTimeout
1.timer.js中的定义

timer模块不需要手动引入，它的源码在/lib/timer.js中
```
function setTimeout(callback, after, arg1, arg2, arg3) {
  if (typeof callback !== 'function') {
    throw new ERR_INVALID_CALLBACK(callback);
  }

  var i, args;
  //将第3个以后的参数包装成数组
  switch (arguments.length) {
    // fast cases
    case 1:
    case 2:
      break;
    case 3:
      args = [arg1];
      break;
    case 4:
      args = [arg1, arg2];
      break;
    default:
      args = [arg1, arg2, arg3];
      for (i = 5; i < arguments.length; i++) {
        // Extend array dynamically, makes .apply run much faster in v6.0.0
        args[i - 2] = arguments[i];
      }
      break;
  }
    const timeout = new Timeout(callback, after, args, false);
  active(timeout);

  return timeout;
}
```
其中调用了Timeout这个对象，在这个文件中找到定义Timeout的地方，可以发现是require('internal/timers')的，所以继续去/lib/internal/timers.js

2.Timeout类定义

源码在/lib/internal/timers.js
```
function Timeout(callback, after, args, isRepeat) {
  // 把延迟时间转换成数字类型，如果不能转换成数字则会变成NaN
  after *= 1; // Coalesce to number or NaN
  if (!(after >= 1 && after <= TIMEOUT_MAX)) {  // 最大时间为2147483647
    if (after > TIMEOUT_MAX) {
      process.emitWarning(`${after} does not fit into` +
                          ' a 32-bit signed integer.' +
                          '\nTimeout duration was set to 1.',
                          'TimeoutOverflowWarning');
    }
    // 小于1，大于最大限制，非法参数都会被置为1
    after = 1; // Schedule on next tick, follows browser behavior
  }

  this._idleTimeout = after; //延迟时间
  this._idlePrev = this; //前驱指针
  this._idleNext = this; //后继指针
  this._idleStart = null; //该定时器超时时间
  // This must be set to null first to avoid function tracking
  // on the hidden class, revisit in V8 versions after 6.2
  this._onTimeout = null;
  this._onTimeout = callback; //回调函数
  this._timerArgs = args; //参数
  this._repeat = isRepeat ? after : null; //setInterval的参数
  this._destroyed = false; //摧毁标记

  this[kRefed] = null;
  // 实验中的API,作为异步资源添加钩子函数
  initAsyncResource(this, 'Timeout');
}
```
由此可见必传参数是callback,如果延迟时间没传，则默认是1,initAsyncResource是在async_hooks注册h回调，以跟踪在Nodejs应用程序内部创建的异步资源的寿命

3.active(timeout)

继续在/lib/internal/timers.js找active
```
function active(item) {
  insert(item, true, getLibuvNow());
}
```
getLibuvNow是通过internalBinding('timers')引入的，在src/timer.cc中找到这个方法,大概可以看出来这个接口就是为了获取准确的当前时间
```
void GetLibuvNow(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  args.GetReturnValue().Set(env->GetNow());
}
```

再看一下insert接口
```
function insert(item, refed, start) {
  let msecs = item._idleTimeout; // 延迟时间
  if (msecs < 0 || msecs === undefined)
    return;

  // Truncate so that accuracy of sub-milisecond timers is not assumed.
  msecs = Math.trunc(msecs); //将数字的小数部分去掉，只保留整数

  item._idleStart = start; // 当前时间

  // Use an existing list if there is one, otherwise we need to make a new one.
  var list = timerListMap[msecs]; // timerListMap就是个对象
  if (list === undefined) { // 如果链表不存在就去创建一个
    debug('no %d list was found in insert, creating a new one', msecs);
    const expiry = start + msecs; // 过期时间
    timerListMap[msecs] = list = new TimersList(expiry, msecs); // 赋予该条链表所需要的一些属性
    timerListQueue.insert(list); // 把该链表加到最小堆

    if (nextExpiry > expiry) { // 如果旧的定时器比新生成的定时器的到期时间还要晚，那需要重新设置一下nextExpiry
      scheduleTimer(msecs);
      nextExpiry = expiry;
    }
  }

  if (!item[async_id_symbol] || item._destroyed) { //如果这个定时器的回调没有注册在async_hooks或者这个定时器检测到已经毁灭，则注册
    item._destroyed = false;
    initAsyncResource(item, 'Timeout');
  }

  if (refed === !item[kRefed]) { // 与libuv的handle有关，refed是true的情况下需要添加handle
    if (refed)
      incRefCount();
    else
      decRefCount();
  }
  item[kRefed] = refed;

  L.append(list, item); // 往双向循环链表里添加该项
}
```
这段代码其实就是往特定timeout的链表里插入这个定时器，如果这个特定timeout的链表不存在就创建一个新的链表，把新链表放进最小堆里面，入过该链表已经存在，直接吧这个定时器插进去即可。

到目前为止，定时器已经插进去了，那是怎么触发的呢？

4.定时器的执行逻辑

触发逻辑在lib/internal/bootstrap/node.js的284-291行，
```
{
  const { nextTick, runNextTicks } = setupTaskQueue();
  process.nextTick = nextTick;
  // Used to emulate a tick manually in the JS land.
  // A better name for this function would be `runNextTicks` but
  // it has been exposed to the process object so we keep this legacy name
  // TODO(joyeecheung): either remove it or make it public
  process._tickCallback = runNextTicks;

  const { getTimerCallbacks } = require('internal/timers');
  const { setupTimers } = internalBinding('timers');
  const { processImmediate, processTimers } = getTimerCallbacks(runNextTicks);
  // Sets two per-Environment callbacks that will be run from libuv:
  // - processImmediate will be run in the callback of the per-Environment
  //   check handle.
  // - processTimers will be run in the callback of the per-Environment timer.
  setupTimers(processImmediate, processTimers);
  // Note: only after this point are the timers effective
}
```
Node.js通过调用setupTimers方法将定时器处理函数processTimers传递给了底层，它最终会被用来调度执行定时器，processTimers方法由lib/internal/timers.js中提供的getTimerCallbacks(runNextTicks)方法运行得到,那先看一下getTimerCallbacks这个方法,找到processTimers这个方法
```
  function processTimers(now) {
    debug('process timer lists %d', now); // now是当前时间
    nextExpiry = Infinity;

    let list;
    let ranAtLeastOneList = false;
    while (list = timerListQueue.peek()) {
      if (list.expiry > now) { 
        nextExpiry = list.expiry; //如果定时器链表的过期时间大于现在，那就把下一个要触发的过期时间设为最小堆里面最小的过期时间
        return refCount > 0 ? nextExpiry : -nextExpiry;
      }
      if (ranAtLeastOneList)
        runNextTicks(); // 执行nextTick回调
      else
        ranAtLeastOneList = true; 
      listOnTimeout(list, now); // 执行list里面所有到期的回调
    }
    return 0;
  }
```
```
function listOnTimeout(list, now) {
    const msecs = list.msecs; // 定时器过期时间

    debug('timeout callback %d', msecs);

    var diff, timer;
    let ranAtLeastOneTimer = false;
    while (timer = L.peek(list)) {
      diff = now - timer._idleStart;

      // 检查此循环迭代对于下一个计时器是否太早
      // 如果有更多的计时器排在列表的后面，就会发生这种情况
      if (diff < msecs) {
        list.expiry = Math.max(timer._idleStart + msecs, now + 1);
        list.id = timerListId++;
        timerListQueue.percolateDown(1);
        debug('%d list wait because diff is %d', msecs, diff);
        return;
      }

      if (ranAtLeastOneTimer)
        runNextTicks();
      else
        ranAtLeastOneTimer = true;

      // The actual logic for when a timeout happens.
      L.remove(timer); // 从链表中删除这一项

      const asyncId = timer[async_id_symbol];

      if (!timer._onTimeout) {
        if (timer[kRefed])
          refCount--;
        timer[kRefed] = null;

        if (destroyHooksExist() && !timer._destroyed) {
          emitDestroy(asyncId);
          timer._destroyed = true;
        }
        continue;
      }

      emitBefore(asyncId, timer[trigger_async_id_symbol]);

      let start;
      if (timer._repeat) // 如果是重复定时器
        start = getLibuvNow(); // 重新设置开始时间

      try {
        const args = timer._timerArgs;
        if (args === undefined)
          timer._onTimeout();
        else
          timer._onTimeout(...args); //执行回调函数
      } finally {
        if (timer._repeat && timer._idleTimeout !== -1) {
          timer._idleTimeout = timer._repeat;
          if (start === undefined)
            start = getLibuvNow();
          insert(timer, timer[kRefed], start);
        } else if (!timer._idleNext && !timer._idlePrev) {
          if (timer[kRefed])
            refCount--;
          timer[kRefed] = null;

          if (destroyHooksExist() && !timer._destroyed) {
            emitDestroy(timer[async_id_symbol]);
            timer._destroyed = true;
          }
        }
      }

      emitAfter(asyncId);
    }

    // If `L.peek(list)` returned nothing, the list was either empty or we have
    // called all of the timer timeouts.
    // As such, we can remove the list from the object map and
    // the PriorityQueue.
    debug('%d list empty', msecs);

    // The current list may have been removed and recreated since the reference
    // to `list` was created. Make sure they're the same instance of the list
    // before destroying.
    if (list === timerListMap[msecs]) {
      delete timerListMap[msecs];
      timerListQueue.shift(); // 从最小堆中删除该项链表
    }
  }
```
上面的接口就是按照情况执行定时器链表，执行完后就可以从最小堆中删掉了
