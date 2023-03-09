`React` 采用了 `fiber` 架构来管理所有组件的渲染任务,并且可以做到暂停某一个组件的渲染。

在 `react-reconciler` 中通过函数 `scheduleUpdateOnFiber(...)` 将 `fiber` 树生成逻辑封装到一个回调函数中,然后通过 `performSyncWorkOnRoot(...)` 或 `performConcurrentWorkOnRoot(...)` 送入 `scheduler` 进行调度。

在 `React` 应用运行中,`scheduler` 是整个 `React` 运行时的大心脏,那么在接下来这篇文章中我们来学习一下 `scheduler`。

# reconciler 运作流程

在开始讲解 `scheduler` 内容之前,我们先来了解一下 `reconciler` 的主要作用,它可以分为一下四个方面:

1. 任务输出: `schedulerUpdateOnfiber(...)` 函数是处理更新任务的入口;
2. 注册调度任务: 与调度中心 `scheduler` 交互,注册调度任务 `task`,等待任务回调;
3. 执行任务回调: 在内存中构造出 `fiber` 树,同时与与渲染器 `react-dom` 交互,在内存中创建出与 `fiber` 对应的 `DOM` 节点;
4. 输出: 与渲染器 `react-dom` 交互,渲染 `DOM` 节点;

`reconciler` 的整个运作流程可以是一个固定的流程,主要运行流程有以下流程所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/838bf957c3f94c36b6ea146bad33bf2c~tplv-k3u1fbpfcp-watermark.image?)

好了,在这里就讲个大概流程,如果想详细了解 `reconciler`,可以查看 `react\packages\react-reconciler\src\ReactFiberWorkLoop.old.js` 源码。

# React 与 Scheduler

在上面说到,在整个 `React` 应用中,`shceduler` 是整个 `React` 运行时的大心脏,那么 `React` 是怎么把任务交给 `scheduler` 调度和控制执行呢?

接下来我们来用一个例子引开权威,先来看以下代码:

```jsx
import React, { Component } from "react";

class App extends Component {
  constructor() {
    super();
    this.state = {
      count: 1,
    };
  }
  render() {
    return (
      <h1 onClick={() => this.setState({ count: this.state.count + 1 })}>
        {this.state.count}
      </h1>
    );
  }
}

export default App;
```

在上面的代码中我们定义了一个类组件,并为 `h1` 元素绑定了点击事件,当点击该元素时会触发该事件,`count` 会加一,触发函数状态更新。

而当我们通过点击的时候,调用的 `this.setState(...)` 方法,实际上是调用的 `Component` 构造函数上的原型方法 `setState(...)`,忽略一些错误边界处理的代码如下所示:

```js
Component.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState, callback, "setState");
};
```

在这里的 `this.updater.enqueueSetState(...)` 方法的调用,实际上调用的是 `classComponentUpdater` 对象上的 `enqueueSetState(...)` 方法,具体流程如下图所示:

![动画.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3792b94dac0149e7a0d42f1966f62030~tplv-k3u1fbpfcp-watermark.image?)

在这里,这个方法的操作有以下几个事情:

1. 获取当前组件对应的 `fiber` 节点;
2. 获取当前事件触发的时间;
3. 获取到当前事件对应的 `Lane` 优先级;
4. 创建一个 `update` 对象,并将传进来的参数添加到 `payload` 属性上,并通过判断在调用 `setState(...)` 时是否传入回调函数,有则将其绑定在 `update.callback` 字段上;
5. 将 `fiber`、`update`、`lane` 传入 `enqueueUpdate(...)` 函数返回一个新的 `fiberRoot`;
6. 最后将 `fiberRoot` 交给 `scheduleUpdateOnFiber(...)` 函数处理;

以下画圈这些值是不是好熟悉,懂的都懂 😉😉😉

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57b138808db9418590ce574040c19ae0~tplv-k3u1fbpfcp-watermark.image?)

# scheduleUpdateOnFiber

在前面的内容中讲到 `scheduleUpdateOnFiber(...)` 函数是 `React` 应用处理更新的入口,无论是首次渲染,还是后续更新操作,都会进入到该函数,而首次渲染是经过 `updateContainer(...)` 函数然后在该函数体中调用 `scheduleUpdateOnFiber(...)`,接下来我们看看这个函数的关键代码:

```ts
function ensureRootIsScheduled(root: FiberRoot, currentTime: number) {
  // 保存调度单元，scheduler所创建的task对象，与并发模式的调度有关。
  const existingCallbackNode = root.callbackNode;
  // 是否有 lane 饿死，标记为过期。
  markStarvedLanesAsExpired(root, currentTime);
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  // 无任务
  if (nextLanes === NoLanes) {
    // Special case: There's nothing to work on.
    if (existingCallbackNode !== null) {
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }
  // 根据lane，获取优先级
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;
  if (
    // 检查是否有一个正在执行的的任务，优先级相同复用它。
    existingCallbackPriority === newCallbackPriority &&
    !(
      __DEV__ &&
      ReactCurrentActQueue.current !== null &&
      existingCallbackNode !== fakeActCallbackNode
    )
  ) {
    return;
  }
  if (existingCallbackNode != null) {
    //取消现有的回调。我们将在下面安排一个新的。
    cancelCallback(existingCallbackNode);
  }
  // 开启一个新的调度
  let newCallbackNode;

  // 同步优先级
  if (newCallbackPriority === SyncLane) {
    // Special case: Sync React callbacks are scheduled on a special
    // internal queue
    if (root.tag === LegacyRoot) {
      if (__DEV__ && ReactCurrentActQueue.isBatchingLegacy !== null) {
        ReactCurrentActQueue.didScheduleLegacyUpdate = true;
      }
      // legacy模式，调度渲染performSyncWorkOnRoot
      scheduleLegacySyncCallback(performSyncWorkOnRoot.bind(null, root));
    } else {
      scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    }

    // 是否支持微任务
    // 当前是同步优先级，立即执行渲染
    if (supportsMicrotasks) {
      // Flush the queue in a microtask.
      if (__DEV__ && ReactCurrentActQueue.current !== null) {
        ReactCurrentActQueue.current.push(flushSyncCallbacks);
      } else {
        scheduleMicrotask(() => {
          if (
            (executionContext & (RenderContext | CommitContext)) ===
            NoContext
          ) {
            // flushSyncCallbacks，即立刻执行performSyncWorkOnRoot
            flushSyncCallbacks();
          }
        });
      }
    } else {
      // flushSyncCallbacks，即立刻执行performSyncWorkOnRoot
      scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);
    }
    newCallbackNode = null;
  } else {
    // 并发模式 异步优先级。
    let schedulerPriorityLevel;
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    // 保存调度单元 scheduler 所创建的task对象
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
  // 根据 newCallbackPriority 保存调度最高 lane 等级的优先级
  root.callbackPriority = newCallbackPriority;
  // 保存调度单元 scheduler 所创建的task对象
  root.callbackNode = newCallbackNode;
}
```

该函数的主要作用还是确保在 `root` 在调度区分类型是同步任务还是并发任务,如果是同步任务,则调用 `scheduleCallback(ImmediateSchedulerPriority, flushSyncCallbacks);` 函数。

那么什么时候同步任务,什么是异步任务呢,据我所知道的,首次渲染是同步任务和过期任务会被当做同步任务执行,而 `onclick` 等操作的优先级为 `SyncLane`,所以它是属于同步任务,接下来我们来看一个例子:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d68b2cff13b48dd839dbe793bd2f2f0~tplv-k3u1fbpfcp-watermark.image?)

我在同步任务和异步任务执行前的代码中添加了 `console.log(...)` 函数,接下来请看下面的动图演示:

![动画.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b286551ea3124cf4bcc5e2ce86995bc4~tplv-k3u1fbpfcp-watermark.image?)

验证结果正是我前面说到的那样,首次渲染会是并发任务,`onClick` 是同步任务。

## scheduleSyncCallback

`scheduleSyncCallback` 是 `react` 自己维护的一个同步任务数组,然后将所有同步任务推入 `syncQueue` 队列保存起来,在后续执行 `flushSyncCallbacks` 时使用,那么我们就看看 `scheduleSyncCallback(...)` 是怎么定义的:

```ts
export function scheduleSyncCallback(callback: SchedulerCallback) {
  if (syncQueue === null) {
    syncQueue = [callback];
  } else {
    syncQueue.push(callback);
  }
}
```

那么接下来我们看看 `flushSyncCallbacks(...)` 函数是在干嘛?

## flushSyncCallbacks

`syncQueue` 会通过 `flushSyncCallbacks(...)` 遍历,然后逐个执行其中的任务,这个流程是在 `react` 内部中执行的,这个实际上是一个同步任务调度器,其函数体代码如下所示:

```js
export function flushSyncCallbacks() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    // Prevent re-entrance.
    isFlushingSyncQueue = true;
    let i = 0;
    const previousUpdatePriority = getCurrentUpdatePriority();
    try {
      const isSync = true;
      const queue = syncQueue;
      setCurrentUpdatePriority(DiscreteEventPriority);
      for (; i < queue.length; i++) {
        let callback = queue[i];
        do {
          callback = callback(isSync);
        } while (callback !== null);
      }
      syncQueue = null;
      includesLegacySyncCallbacks = false;
    } catch (error) {
      // 发生报错时，保留剩余的任务队列
      if (syncQueue !== null) {
        syncQueue = syncQueue.slice(i + 1);
      }
      // 通过 scheduler 进行任务恢复
      scheduleCallback(ImmediatePriority, flushSyncCallbacks);
      // 同时抛出错误
      throw error;
    } finally {
      setCurrentUpdatePriority(previousUpdatePriority);
      isFlushingSyncQueue = false;
    }
  }
  return null;
}
```

如果是并发任务,则有如下的操作:

```js
// 保存调度单元 scheduler 所创建的task对象
newCallbackNode = scheduleCallback(
  schedulerPriorityLevel,
  performConcurrentWorkOnRoot.bind(null, root)
);

// 根据 newCallbackPriority 保存调度最高 lane 等级的优先级
root.callbackPriority = newCallbackPriority;
// 保存调度单元 scheduler 所创建的task对象
root.callbackNode = newCallbackNode;
```

`performConcurrentWorkOnRoot` 函数具体做了什么,现在我们还不需要关心,我们只要知道这就是任务队列里的任务的执行函数就可以了。

再最后会把 `scheduleCallback(...)` 函数的返回结果挂载到 `root.callbackNode`。

这个时候,我们才真正进入到了 `scheduler` 模块,因为 `Scheduler` 的主入口正是 `scheduleCallback(...)` 函数。

# scheduleCallback

`scheduleCallback`  的本质其实是在调用  `Scheduler_scheduleCallback`  函数,而  `Scheduler_scheduleCallback`  函数是  `unstable_scheduleCallback`  函数的别名。

我们先来看看这个函数是怎么定义的,具体有如下代码所示:

```js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 这个 currentTime 获取的是 performance.now() 时间戳
  var currentTime = getCurrentTime();

  // 任务开始的时间
  var startTime;
  if (typeof options === "object" && options !== null) {
    var delay = options.delay;
    if (typeof delay === "number" && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
  } else {
    startTime = currentTime;
  }

  // 根据调度优先级设置相应的超时时间
  var timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT; // -1
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT; // 250
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT; // 1073741823
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT; // 10000
      break;
    case NormalPriority:
    default: // 5 000
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
  }

  // expirationTime 用于标识一个任务具体的过期时间
  var expirationTime = startTime + timeout;
  // 属于 Scheduler 自己的 task
  var newTask = {
    id: taskIdCounter++, // 一个自增编号
    callback, // 这里的 callback 是 performConcurrentWorkOnRoot 函数
    priorityLevel, // 调度优先级
    startTime, // 任务开始时间
    expirationTime, // 任务过期时间
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }

  // 表示这个任务将会延迟执行
  if (startTime > currentTime) {
    // 当前任务已超时，插入超时队列
    newTask.sortIndex = startTime;
    // 超时任务
    push(timerQueue, newTask);

    // peek 查看堆的顶点, 也就是优先级最高的`task`
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // 这个任务是最早延迟执行的
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        // 取消现有的定时器
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 安排一个定时器
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 任务未超时，插入调度任务队列
    newTask.sortIndex = expirationTime;
    // 调度中的队列
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // 符合更新调度执行的标志
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true; // 是否有回调函数
      requestHostCallback(flushWork); // 调度任务
    }
  }
  return newTask;
}
```

在开始讲这个函数之前,我们先来了解一些相关的概念。

## 任务队列

在 `unstable_scheduleCallback` 函数之外,`scheduler` 维护着两个全局变量：`taskQueue`  和  `timerQueue`,它们用来表示任务队列。

其中 `timerQueue` 表示要延迟执行的任务,`taskQueue` 表示现在就要执行的任务,这两个数组命名为 `Queue`,但是它实际上是一个 `二叉堆`。

`二叉堆` 是一种特殊的堆,`二叉堆` 是 `完全二叉树` 或者是 `近似完全二叉树`。

`二叉堆 满足堆特性:

- 父节点的键值总是保持固定的序关系于任何一个子节点的键值,且每个节点的左子树和右子树都是一个二叉堆;
- 当父节点的键值总是大于或等于任何一个子节点的键值时为 `最大堆`;
- 当父节点的键值总是小于或等于任何一个子节点的键值时为 `最小堆`;

二叉堆一般用数组来表示,如果根节点在数组中的位置是 1,那么它的子节点分别在 `2*n`和 `2*n+1` 位置上。因此,第 1 个位置的子节点在 2 和 3,第 2 个位置的子节点在 4 和 5。以此类推,这种基于 1 的数组存储方式便于寻找父节点和子节点。

那么它判断大小的依据是什么呢?答案就是它会根据 `newTask` 的 `sortIndex` 字段,它的初始值是 `-1`。

如果是普通任务,也或者说是未超时,使用 `expirationTime` 作为 `sortIndex` 字段的值:

```js
// 任务未超时，插入调度任务队列
newTask.sortIndex = expirationTime;
// 调度中的队列
push(taskQueue, newTask);
```

如果是延迟任务,则使用 `startTime` 作为 `sortIndex` 字段的值:

```js
// 当前任务已超时，插入超时队列
newTask.sortIndex = startTime;
// 超时任务
push(timerQueue, newTask);
```

如果任务为过期,使用当前时间作为排序顺序,过期时间越早,说明离现在越远,任务优先级越高。

如果任务为立即要执行的任务,使用过期时间 `expirationTime` 作为排序顺序,过期时间越小,说明离现在越近,优先级越高。

我们通过浏览器控制台来看看 `newTask` 的值是怎么样的:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a5262b5a89d4781909716f3cb7adadd~tplv-k3u1fbpfcp-watermark.image?)

这个时候我们把关注点回到 `Scheduler_scheduleCallback(...)` 这个函数里,这个函数主要做了以下这几件事情:

1. 获取开始时间 `startTime`;
2. 根据不同的优先级设置超时时间 `timeout`;
3. 使用 `expirationTime` 给一个任务标记一个具体的过期时间;
4. 创建一个 `newTask` 对象;
5. 如果任务已过期,把任务推入 `timerQueue`,检查是否有执行中的 `isHostTimeoutScheduled`,有的话使用 `cancelHostTimeout()` 取消掉,再重新开启一个;
6. 如果任务未过期,则把任务推入 `taskQueue`,开始调度 `requestHostCallback(flushWork)`,- `flushWork`  是真正需要执行的函数;

## requestHostCallback

`requestHostCallback(...)` 函数的主要作用是保存回调函数到 `scheduledHostCallback` 变量上,把 `isMessageLoopRunning` 设置为 `true`,执行 `schedulePerformWorkUntilDeadline(...)` 函数。

## requestHostTimeout 与 cancelHostTimeout

在未过期的任务中出现了两个函数,它们分别是 `requestHostTimeout(...)` 与 `cancelHostTimeout(...)`,它们的目的是为了让一个未过期的任务能够达到刚好过期的状态,那么只需要延迟 `startTime-currentTime` 就可以了,前者是用来开启定时器的,后者是用来清除定时器的:

```js
// 安排一个定时器
requestHostTimeout(handleTimeout, startTime - currentTime);

function requestHostTimeout(callback, ms) {
  taskTimeoutID = localSetTimeout(() => {
    callback(getCurrentTime());
  }, ms);
}

function cancelHostTimeout() {
  localClearTimeout(taskTimeoutID);
  taskTimeoutID = -1;
}
```

## handleTimeout

在调用 `requestHostTimeout` 函数的时候,传入的第一个参数就是 `handleTimeout` 函数,这个函数的主要作用就是有以下几个方面:

- 执行 `advanceTimers`,用来更新 `timerQueue` 和 `taskQueue` 两个队列;
- 判断是否已经启动调度任务;
- 如果当前 `taskQueue` 存在有任务,那么启动调度任务的回调 `requestHostCallback(flushWork)`;
- 如果当前 `taskQueue` 不存在任务,获取 `timerQueue` 队列中的第一个任务,如果不为 `null`,则递归执行;

该函数的具体代码如下所示:

```js
function handleTimeout(currentTime) {
  isHostTimeoutScheduled = false;
  advanceTimers(currentTime);

  // 判断是否已启动调度任务回调
  if (!isHostCallbackScheduled) {
    if (peek(taskQueue) !== null) {
      // 当前 调度中的队列 不为空
      isHostCallbackScheduled = true;
      // 启动调度任务回调
      requestHostCallback(flushWork);
    } else {
      // 如果 当前 调度中的队列 为空
      // 获取待调度的任务中的第一个
      const firstTimer = peek(timerQueue);
      if (firstTimer !== null) {
        // 递归执行 handleTimeout
        requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
      }
    }
  }
}
```

该函数的主要意图就是用来更新 `timerQueue` 中的任务转移到 `taskQueue` 中,这个方法是不负责执行任务的回调的,它做的就是优先保证过期任务的执行,对于可延迟的任务,下一帧执行。

所以在 `unstable_scheduleCallback` 函数中会有这样的判断,等到该函数要执行了,需要清除定时器:

```js
  // 表示这个任务将会延迟执行
  if (startTime > currentTime) {
   // 当前任务已超时，插入超时队列
    newTask.sortIndex = startTime;
    // 超时任务
    push(timerQueue, newTask);

     // peek 查看堆的顶点, 也就是优先级最高的`task`
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
        // 这个任务是最早延迟执行的
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
          // 取消现有的定时器
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // 安排一个定时器
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
```

## advanceTimers

这个函数做的事情很重要,但是代码简单,具体代码如下所示:

```js
function advanceTimers(currentTime) {
  // 获取第一个待调度的任务
  let timer = peek(timerQueue);
  while (timer !== null) {
    // 如果任务不为空
    if (timer.callback === null) {
      // 没有 callback 说明已经执行完了或者是个空任务，直接忽略
      pop(timerQueue);
    } else if (timer.startTime <= currentTime) {
      // 任务超时了，需要执行了
      pop(timerQueue);
      // 标记排序用的标识
      timer.sortIndex = timer.expirationTime;
      push(taskQueue, timer);
      if (enableProfiling) {
        markTaskStart(timer, currentTime);
        timer.isQueued = true;
      }
    } else {
      return;
    }
    // 如果出现有任务被移动或者移除的情况，检查下一个 timer
    timer = peek(timerQueue);
  }
}
```

这个函数具体做的事情有以下几个方面:

- 获取第一个 `timerQueue` 过期任务,并且任务不为空;
- 判断是否有 `callback`,没有 `callback` 说明已经执行完了或者是个空任务,直接忽略;
- 判断任务是否超时,超时的情况从  `timerQueue`  取出来,`push`  到  `taskQueue`，并更新排序标记;
- 如果出现有任务被移动或者移除的情况,检查下一个 timer;

一句话总结就是检查 `timerQueue` 中的过期任务放到 `taskQueue` 队列中。

## schedulePerformWorkUntilDeadline

创建一个 `MessageChannel` 实例添加一个宏任务,为 `port1.onmessage` 注册 `performWorkUntilDeadline` 使得最终的回调在下一个事件循环才会执行,具体代码如下所示:

```js
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

schedulePerformWorkUntilDeadline = () => {
  port.postMessage(null);
};
```

## performWorkUntilDeadline

`performWorkUntilDeadline` 是任务的执行者,它主要是用来在时间片内执行任务,如果没有执行完,用一个新的调度者继续调度,该函数具体代码如下所示:

```ts
const performWorkUntilDeadline = () => {
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    startTime = currentTime;
    const hasTimeRemaining = true;

    // 是否有更多的工作要做
    let hasMoreWork = true;
    try {
      // 调用了 scheduledHostCallback，并保存返回结果
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      if (hasMoreWork) {
        // 如果 scheduledHostCallback 返回不为 false，那么发送消息，重新调度执行
        schedulePerformWorkUntilDeadline();
      } else {
        // 如果 scheduledHostCallback 返回 false，那么任务结束
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
  needsPaint = false;
};
```

该函数主要做的事情主要有以下几个方面:

- 判断是否有  `scheduledHostCallback`,如果存在说明存在需要被调度的任务;
- 获取当前时间,首次将 `hasMoreWork` 设置为 `true`;
- 并且把 `scheduledHostCallback(hasTimeRemaining, currentTime)` 的返回值赋值给 `hasMoreWork`;
- 如果 `hashMoreWork` 的值为真值,继续调用 `schedulePerformWorkUntilDeadline` 函数,否则任务结束;

`scheduledHostCallback` 函数正是我们传进来的 `callback`,也就是 `flushWork` 函数。

## flushWork

在 `flushWork` 函数中,主要做的事情是对几个状态进行修改:

- 将 `isHostCallbackScheduled` 设置为 `false`,表示没有回调函数了;
- 如果 `isHostTimeoutScheduled` 为 `true`,将其设置为 `false`,并取消定时;
- 将 `isPerformingWork` 设置为 `true`,表示任务在执行中了;
- 调用 `workLoop(...)` 函数;

`flushWork(...)` 函数的定义如下代码所示:

```js
function flushWork(hasTimeRemaining, initialTime) {
  if (enableProfiling) {
    markSchedulerUnsuspended(initialTime);
  }
  isHostCallbackScheduled = false;
  if (isHostTimeoutScheduled) {
    // 要开始执行了，不需要再等待 待调度任务 进入调度队列了，直接取消掉
    isHostTimeoutScheduled = false;
    cancelHostTimeout();
  }

  // 标记在执行中了
  isPerformingWork = true;
  // 保存当前的优先级
  const previousPriorityLevel = currentPriorityLevel;
  try {
    if (enableProfiling) {
      try {
        // 交给小弟 workLoop 去做任务中断与恢复了
        return workLoop(hasTimeRemaining, initialTime);
      } catch (error) {
        if (currentTask !== null) {
          const currentTime = getCurrentTime();
          markTaskErrored(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        throw error;
      }
    } else {
      return workLoop(hasTimeRemaining, initialTime);
    }
  } finally {
    // 标记当前无任务执行
    currentTask = null;
    // 恢复优先级
    currentPriorityLevel = previousPriorityLevel;
    // 标记执行结束
    isPerformingWork = false;
    if (enableProfiling) {
      const currentTime = getCurrentTime();
      markSchedulerSuspended(currentTime);
    }
  }
}
```

# workLoop

`flushWork` 是大哥,它做了很多它自己能做的,但是把一些脏活累活交给它的小弟 `workLoop` 去做,例如任务中断和任务恢复,我们来看看它的实现是怎么样的,代码有点长,如果不想看代码可以直接看总结:

```js
function workLoop(hasTimeRemaining, initialTime) {
  debugger;
  let currentTime = initialTime;
  // 先把过期的任务从 timerQueue 捞出来丢到 taskQueue 打包一块执行了
  advanceTimers(currentTime);
  // 获取优先级最高的任务
  currentTask = peek(taskQueue);
  // 循环任务队列
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    if (
      currentTask.expirationTime > currentTime &&
      // shouldYieldToHost 这个函数用来判断是否需要等待
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // 如果没有剩余时间或者该任务停止了就退出循环
      break;
    }
    // 取出当前的任务中的回调函数 performConcurrentWorkOnRoot
    const callback = currentTask.callback;
    if (typeof callback === "function") {
      // 只有 callback 为函数时才会被识别为有效的任务
      currentTask.callback = null;
      // 设置执行任务的优先级，回想下 flushWork中的恢复优先级，关键就在这
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      if (enableProfiling) {
        markTaskRun(currentTask, currentTime);
      }

      // 如果返回新的函数,表示当前的工作还没有完成
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === "function") {
        // 这里是真正的恢复任务，等待下一轮循环时执行
        currentTask.callback = continuationCallback;

        if (enableProfiling) {
          markTaskYield(currentTask, currentTime);
        }
      } else {
        if (enableProfiling) {
          markTaskCompleted(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        if (currentTask === peek(taskQueue)) {
          // 不需要恢复任务了，标识当前任务已执行完，把任务从队列中移除掉
          pop(taskQueue);
        }
      }
      // 先把过期的任务从 timerQueue 捞出来丢到 taskQueue 打包一块执行了
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    // 获取最高优先级的任务（不一定是下一个任务）
    currentTask = peek(taskQueue);
  }
  // Return whether there's additional work
  if (currentTask !== null) {
    // 还有任务说明调度被暂停了，返回true标明需要恢复任务
    return true;
  } else {
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // 任务都跑完了
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    // 返回false意味着当前任务都执行完了，不需要恢复
    return false;
  }
}
```

这个函数的主要执行步骤有以下几个方面:

- 调用 `advanceTimers(...)` 函数把过期的任务从 `timerQueue(...)` 取出来转移到 `taskQueue` 里执行;
- 获取优先级最高的任务作为第一个处理的任务;
- 进入循环，在执行任务前，先看看还有没有时间;
- 如果没有时间跳出循环,返回 `true`,标明需要恢复任务;
- 如果有时间则正常执行任务,并保存任务的返回值;
- 如果返回值是函数,说明任务执行时长不够了,需要恢复;
- 如果返回值不是函数,说明已经执行完了,从队列中移除当前任务;
- 每次调度任务之后,都通过 `advanceTimers` 把过期的任务从  `timerQueue`  取出来转移  `taskQueue`，因为在执行过程中有可能部分任务也过期了;

在上面的代码中调用了 `shouldYieldToHost` 函数,该函数的主要作用是判断当前时间片是否到期。

`continuationCallback` 函数是任务的恢复和执行,不仅完成多个任务的时候会被打断,单个任务执行也会被打断,如果被打断了,那么该就返回 `continuationCallback` 并等待下次执行。

# 参考文章

- [React 之 Scheduler 源码解读（上）](https://juejin.cn/post/7171000978278187038#heading-12)
- [深入"时间管理大师" —— React Scheduler](https://juejin.cn/post/6969452281301467143#heading-21)
- [React 源码细读-深入了解 scheduler"时间管理大师"](https://zhuanlan.zhihu.com/p/384525799)
- [图解 React](https://7kms.github.io/react-illustration-series/)

# 总结

没有总结,`Scheduler` 作为 `React` 的大心脏,掌握了 `Scheduler`你就等于学会了 `Scheduler`。

**_本文正在参加[「金石计划」](https://juejin.cn/post/7207698564641996856/ "https://juejin.cn/post/7207698564641996856/")_**
