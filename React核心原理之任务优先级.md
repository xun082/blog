`React` 是一个声明式,高效且灵活的用于构建用户界面的 `JavaScript` 库,`React` 团队一直致力于实现高效渲染。

其中 `可中断渲染`、`时间切片`、`异步渲染(Suspence)`等特性是 `React` 中很突出的特点,它们的具体实现都依赖于 `优先级管理`,那么今天我们来聊聊这个话题。

# 为什么需要优先级

不同优先级的任务间,会存在一种现象,当执行低优先级任务时,突然插入一个高优先级任务,那么会中断低优先级的任务,先执行高优先级的任务,我们可以将这种现象称为任务插队。

当高优先级任务执行完,准备执行低优先级任务时,又插入一个高优先级任务,那么又会执行高优先级任务,如果不断有高优先级任务插队执行,那么低优先级任务便一直得不到执行,我们称这种现象为任务饥饿问题。

当用户操作界面时,例如搜索、点击、下拉等操作,为了避免页面卡顿,需要让出线程的执行权,先执行用户触发的事件,这个我们称之为高优先级任务,其它不那么重要的事件我们称之为低优先级任务。

# 优先级分类

在整个 `React` 内部中对于优先级的管理,根据其功能的不同分为了三种优先级:

- `lane` 优先级;
- `React` 事件优先级;
- `Schedule` 优先级;

# lane 优先级

`lane` 对其定义使用了二进制变量,利用了位掩码的特性,在频繁运算的时候占用内存少,计算速度快。

回想一下我们学校的操场,是不是分为了多个跑道,跑道越往内,距离越近,`lane` 模型借助了这个概念,将其分为了 `31` 个赛道,其中位数越少的赛道,也就是 `1` 越往右的赛道有`优先级` 就越高,某些相邻的赛道拥有相同的 `优先级`,称为赛道组。

| 优先级                       | 赛道注释       | 二进制值                          | 赛道位置 |
| ---------------------------- | -------------- | --------------------------------- | -------- |
| NoLane                       | 没有赛道       | 0b0000000000000000000000000000000 | 0        |
| SyncLane                     | 同步赛道       | 0b0000000000000000000000000000001 | 0        |
| InputContinuousHydrationLane | 连续注水赛道   | 0b0000000000000000000000000000010 | 1        |
| InputContinuousLane          | 连续注水赛道   | 0b0000000000000000000000000000100 | 2        |
| DefaultHydrationLane         | 默认注水赛道   | 0b0000000000000000000000000001000 | 3        |
| DefaultLane                  | 默认赛道       | 0b0000000000000000000000000010000 | 4        |
| TransitionHydrationLane      | 过渡注水赛道   | 0b0000000000000000000000000100000 | 5        |
| TransitionLane1              | 过渡赛道       | 0b0000000000000000000000001000000 | 6        |
| TransitionLane2              | 过渡赛道       | 0b0000000000000000000000010000000 | 7        |
| TransitionLane3              | 过渡赛道       | 0b0000000000000000000000100000000 | 8        |
| TransitionLane4              | 过渡赛道       | 0b0000000000000000000001000000000 | 9        |
| TransitionLane5              | 过渡赛道       | 0b0000000000000000000010000000000 | 10       |
| TransitionLane6              | 过渡赛道       | 0b0000000000000000000100000000000 | 11       |
| TransitionLane7              | 过渡赛道       | 0b0000000000000000001000000000000 | 12       |
| TransitionLane8              | 过渡赛道       | 0b0000000000000000010000000000000 | 13       |
| TransitionLane9              | 过渡赛道       | 0b0000000000000000100000000000000 | 14       |
| TransitionLane10             | 过渡赛道       | 0b0000000000000001000000000000000 | 15       |
| TransitionLane11             | 过渡赛道       | 0b0000000000000010000000000000000 | 16       |
| TransitionLane12             | 过渡赛道       | 0b0000000000000100000000000000000 | 17       |
| TransitionLane13             | 过渡赛道       | 0b0000000000001000000000000000000 | 18       |
| TransitionLane14             | 过渡赛道       | 0b0000000000010000000000000000000 | 19       |
| TransitionLane15             | 过渡赛道       | 0b0000000000100000000000000000000 | 20       |
| TransitionLane16             | 过渡赛道       | 0b0000000001000000000000000000000 | 21       |
| RetryLane1                   | 重试赛道       | 0b0000000010000000000000000000000 | 22       |
| RetryLane2                   | 重试赛道       | 0b0000000100000000000000000000000 | 23       |
| RetryLane3                   | 重试赛道       | 0b0000001000000000000000000000000 | 24       |
| RetryLane4                   | 重试赛道       | 0b0000010000000000000000000000000 | 25       |
| RetryLane5                   | 重试赛道       | 0b0000100000000000000000000000000 | 26       |
| SelectiveHydrationLane       | 选择性注水赛道 | 0b0001000000000000000000000000000 | 27       |
| IdleHydrationLane            | 空闲注水赛道   | 0b0010000000000000000000000000000 | 28       |
| IdleLane                     | 非空闲赛道     | 0b0100000000000000000000000000000 | 29       |
| OffscreenLane                | 离屏渲染赛道   | 0b1000000000000000000000000000000 | 30       |

除了表中显示的之外,可以看到有几个变量占用了几条赛道,比如:

```js
const TransitionLanes: Lanes = 0b0000000001111111111111111000000;
```

这就是 `批` 的概念其主要原因在于越低 `优先级` 的更新越容易被打断,导致积压下来,所以需要更多的位,相反最高优先级的同步更新 `SyncLane` 不需要多余的 `lanes`。

# 事件优先级

`React` 按照事件的紧急程度,将它们分成了三个等级。

| EventPriority           | 事件分类                                                                                             | Lane                | 数值      |
| ----------------------- | ---------------------------------------------------------------------------------------------------- | ------------------- | --------- |
| DiscreteEventPriority   | 离散事件: 例如 click、keydown、focusin 等,事件的触发不是连续,可以做到快速响应                        | SyncLane            | 1         |
| ContinuousEventPriority | 连续事件: drag、scroll、mouseover 等，事件的是连续触发的，快速响应可能会阻塞渲染，优先级较离散事件低 | InputContinuousLane | 4         |
| DefaultEventPriority    | 默认的事件优先级                                                                                     | DefaultLane         | 16        |
| IdleEventPriority       | 空闲的优先级                                                                                         | IdleLane            | 536870912 |

事件优先级的具体划分被定义在了 `react/packages/react-dom/src/events/ReactDOMEventListener.js` 目录下的 `getEventPriority` 函数。

# Schedule 优先级

`Schedule优先级`,属于 `schedule` 包,主要有五个:

```js
// react\packages\scheduler\src\SchedulerPriorities.js

export const NoPriority = 0;
export const ImmediatePriority = 1;
export const UserBlockingPriority = 2;
export const NormalPriority = 3;
export const LowPriority = 4;
export const IdlePriority = 5;
```

# 优先级之间计算

利用二进制位的特性,我们可以通过位运算来进行操作,例如合并赛道,删除赛道等等操作。

## 合并赛道

当我们执行一个低优先级的任务时,突然进来了一个高优先级的任务,那么这个低优先级的任务会被打断去执行高优先级的任务。

但是此时两个赛道已经被占用了,我们可以对其进行合并,具体代码如下所示:

```ts
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}
```

具体结果如下所示:

```js
console.log(mergeLanes(0b00000010, 0b00001000)); // 2 | 8 = 10
```

## 删除赛道

当高优先级的任务执行完毕,我们可以对其进行删除,具体代码如下所示:

```ts
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
```

具体结果如下所示:

```js
console.log(removeLanes(0b00000010, 0b00001000)); // 2 & ~8 = 2
```

还有很多优先级的计算方法,后面还会继续讲到,这里就先说说这两个。

# 优先级转换关系

在整个 `React` 应用当中,它们可分为四种优先级,它们分别有如下优先级:

1. 事件优先级: 按照用户事件的交互紧急程度,划分的优先级;
2. 更新优先级：事件导致 `React` 产生的更新对象 `update` 的优先级;
3. 任务优先级：产生更新对象之后,`React` 去执行一个更新任务,这个任务所持有的优先级;
4. 调度优先级: `Schedule` 依据 `React` 更新任务生成一个调度任务,这个调度任务所持有的优先级;

前三者属于 `React` 的优先级机制,第四个属于 `Scheduler` 的优先级机制,`Scheduler` 内部有自己的优先级机制,虽然与 `React` 有所区别,但等级的划分基本一致。

它们之间是相互递进的关系,主要关系图如下所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65330dba078b4c979d6f9cafd4f85226~tplv-k3u1fbpfcp-watermark.image?)

## 事件优先级

在这四个优先级当中,事件优先级作为整个优先级的起点,在上面的内容中又讲到了事件的优先级又分为三个等级,分别是 `DiscreteEvent`、`UserBlockingEvent`、`ContinuousEvent`,其优先级顺序从低到高。

那么事件的优先级是怎么来的呢?

事件的优先级是在注册阶段时就被确定了,在向 `root` 上注册事件时,会根据事件的类型类别创建不同优先级的事件监听,最终将其绑定到 `root` 上。

那么事件是怎么来的呢? 答案是通过 `createEventListenerWrapperWithPriority(...)` 函数并传入当前的目标 `container` 和当前事件名称 `domEventName`,该函数会根据事件的名称去找对应的事件优先级,然后依据优先级返回不同的事件监听函数,在最后通过 `bind` 绑定调用函数返回该事件类型的优先级。

具体代码如下所示:

```ts
export function createEventListenerWrapperWithPriority(
  targetContainer: EventTarget,
  domEventName: DOMEventName,
  eventSystemFlags: EventSystemFlags
): Function {
  const eventPriority = getEventPriority(domEventName);
  console.log(eventPriority);
  let listenerWrapper;
  switch (eventPriority) {
    case DiscreteEventPriority:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    case ContinuousEventPriority:
      listenerWrapper = dispatchContinuousEvent;
      break;
    case DefaultEventPriority:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }

  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer
  );
}
```

而设置时间的优先级主要调用的是 `setCurrentUpdatePriority(...)` 函数,它的代码实现很简单,就两行:

```ts
export function setCurrentUpdatePriority(newPriority: EventPriority) {
  currentUpdatePriority = newPriority;
}
```

而更新优先级是通过 `getCurrentUpdatePriority(...)` 函数来获取的,它的实现也和上面的一样,只有两行代码:

```ts
export function getCurrentUpdatePriority(): EventPriority {
  return currentUpdatePriority;
}
```

## 任务优先级

任务优先级用来区分多个更新任务的紧急程度,谁的优先级高,就先处理谁。

任务优先级通过调用 `getHighestPriorityLanes(...)`函数来传入更新优先级,由更新优先级计算而来,通过该函数来获取最高优先级任务,具体代码忽略了很多,主要都是一些令人眼花缭乱的 `switch...case` 语句,代码如下:

```ts
function getHighestPriorityLanes(lanes: Lanes | Lane): Lanes {
  switch (getHighestPriorityLane(lanes)) {
    case SyncLane:
      return SyncLane;

    // ...

    case InputContinuousHydrationLane:
      return InputContinuousHydrationLane;
    case InputContinuousLane:
      return InputContinuousLane;
    case OffscreenLane:
      return OffscreenLane;
    default:
      return lanes;
  }
}
```

## Schedule 优先级

要想得到 `Schedule` 优先级,那么你首先需要将将 `lane` 模型的优先级转换为 `事件` 优先级,因为他们的值都是 `lane` 值,具体代码如下所示:

```ts
// a 比 b 的优先级高 返回 true
export function isHigherEventPriority(
  a: EventPriority,
  b: EventPriority
): boolean {
  return a !== 0 && a < b;
}

// lanes模型优先级转换为事件优先级
export function lanesToEventPriority(lanes: Lanes): EventPriority {
  const lane = getHighestPriorityLane(lanes);
  if (!isHigherEventPriority(DiscreteEventPriority, lane)) {
    return DiscreteEventPriority;
  }
  if (!isHigherEventPriority(ContinuousEventPriority, lane)) {
    return ContinuousEventPriority;
  }
  if (includesNonIdleWork(lane)) {
    return DefaultEventPriority;
  }
  return IdleEventPriority;
}
```

任务一旦被调度,那么它就会进入 `Schedule`,在 `Schedule` 中,每一个任务都会被 `Schedule` 包装,生成一个属于 `Schedule` 自己的任务队列,它内部维护了两个队列,分别是 `taskQueue`、`timerQueue`。

这两个任务队列分成了过期任务队列和未过期任务的队列去管理它内部的任务,这个我们后面讲到饥饿任务的时候会讲到。

`Schedule` 优先级由 事件优先级转换而来,那么他是怎么来的呢?

在整个 `React` 应用当中,在多个任务的情况下,相对于新与任务,会对现有的任务进行服用或者取消的操作,单个任务的情况,对任务进行同步、异步或者批处理同步调度的决策,这种行为可以看成是一种任务调度协调机制,而 `ensureRootIsScheduled(...)` 函数是  `react`  把任务交由  `scheduler`  调度的最后一步。

该函数的主要执行步骤是有以下几个方面:

1. 获取 root.callbackNode，即旧任务;
2. 检查任务是否过期,将过期任务放入 root.expiredLanes,目的是让过期任务能够以同步优先级去进入调度;
3. `newCallbackPriority`  获取最高 `lane` 等级的优先级;
4. 根据 `newCallbackPriority` 优先级进入不同的调度入口;
5. 并讲其挂载到 `root` 对象上;

```
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );

   // 根据 newCallbackPriority 保存调度最高 lane 等级的优先级
  root.callbackPriority = newCallbackPriority;
  // 保存调度单元 scheduler 所创建的task对象
  root.callbackNode = newCallbackNode;
```

# 插队机制

在 `lane` 优先级中,如果一个低优先级的任务执行,并且在调度的时候触发了一个高优先级的任务,则高优先的任务打断低优先级任务,此时应该取消低优先级的任务,因为此时低优先级的任务可能已经进行了一段时间,`Fiber` 树已经构造了一部分,但这并不会完全取消。

# 饥饿问题

什么是饥饿呢?在维基百科中是这样定义的:

> 在计算机科学中,饥饿是指在并发计算中,进程一直无法获得运行所需的必要资源而发生的问题。调度、互斥锁算法、资源泄漏等都可能导致饥饿,或者在被 DoS 攻击时主动产生饥饿。在并发计算中，如果饥饿不可能发生,这个算法就被称为是无饥饿、无闭锁或者称其拥有有限旁路。这一属性是存活的例子,也是互斥锁算法的两个条件之一。

那么在 `React` 应用中是一个怎么样的形式呢?

在上面的内容中讲到,在高优先级任务执行完毕之后,低优先级任务就会被重启,但假设如果持续有高优先级任务持续进来,那么低优先级岂不是永远不会执行。

为了解决这个问题,`React` 的解决办法一旦优先级任务过期了,那么它就会被提升到同步优先级去立即执行。

## 任务过期

对于过期的处理,在 `React` 应用中有两套逻辑,一套在 `react` 模块里,一套在 `Schedule` 模块里。

在 `schedule` 模块中,有这样的定义,具体实现如下:

```ts
// react\packages\scheduler\src\forks\Scheduler.js
function unstable_scheduleCallback(priorityLevel, callback, options) {
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
}
```

在上面的代码中根据不同的优先级设置过期时间,而通过计算开始时间和过期时间相加,那么该任务就被标记为过期任务,开始时间代码如下所示:

```ts
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
```

通过 `expirationTime` 标记一个任务的具体过期时间:

```js
var expirationTime = startTime + timeout;
```

并通过判断开始时间是否大于当前时间,如果大于,那么该任务会添加到 `timerQueue` 过期队列里面,否则会被添加到 `taskQueue` 待调度任务。

## 解决饥饿问题

现在已经区分了过期任务了,那么在一次更新中,`React` 是怎么处理这些情况的,首先我们应该知道一点,`React` 应用的每一次更新都是从 `fiberRoot` 开始的,页面上需要更新的都是它的子节点,,因此 `ensureRootIsScheduled(...)` 函数便成了每一次更新的必经之路。

在该函数中开头调用 `markStarvedLanesAsExpired(...)` 函数,找出是否有 `lane` 任务是否达到过期时间,如果达到过期时间则将其标记为过期,该函数的具体代码如下所示:

```ts
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number
): void {
  // 获取当前有更新的赛道
  const pendingLanes = root.pendingLanes;
  const suspendedLanes = root.suspendedLanes;
  const pingedLanes = root.pingedLanes;
  // 记录每个赛道上的过期时间
  const expirationTimes = root.expirationTimes;

  // 检查所有车道,看他们是否超过过期时间
  // 如果超过,处理为饥饿车道
  let lanes = pendingLanes;
  while (lanes > 0) {
    // 返回最左侧的1的索引 例如 00100000 索引为 7-2=5
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;

    // 获取当前赛道的过期时间
    const expirationTime = expirationTimes[index];

    // 如果当前车道没有过期时间,若车道没有被挂起或发送
    // 每一个车道默认值为 -1 NoTimestamp值为-1
    if (expirationTime === NoTimestamp) {
      // 计算一个基于当前时间戳的新的过期时间
      if (
        (lane & suspendedLanes) === NoLanes ||
        (lane & pingedLanes) !== NoLanes
      ) {
        // 计算过期时间
        expirationTimes[index] = computeExpirationTime(lane, currentTime);
      }
    } else if (expirationTime <= currentTime) {
      // 把次车道添加到过期车道里
      root.expiredLanes |= lane;
    }
    // 将检查过的车道剔除
    lanes &= ~lane;
  }
}
```

在上面的代码中又通过 `computeExpirationTime(...)` 为每一个任务计算过期时间,代码如下所示,省略了部分代码:

```js
// 计算过期时间
function computeExpirationTime(lane: Lane, currentTime: number) {
  switch (lane) {
    case SyncLane:
    case InputContinuousHydrationLane:
    case InputContinuousLane:
      return currentTime + 250;
    case DefaultHydrationLane:
    case DefaultLane:
    case TransitionHydrationLane:
      return currentTime + 5000;
    case RetryLane5:
      return NoTimestamp;
    case SelectiveHydrationLane:
    case IdleHydrationLane:
    case IdleLane:
    case OffscreenLane:
      return NoTimestamp;
    default:
      return NoTimestamp;
  }
}
```

我们把重点再次放回 `ensureRootIsScheduled(...)` 函数这里,通过 `markStarvedLanesAsExpired(...)` 的标记,过期任务会被挂载到 `root.expiredLanes` 中,会在下一次调度的时候优先从 `root.expiredLanes` 中取值去计算,这个时候会将过期任务并入同步任务一起执行,你可以理解为将其优先级升级为同步优先级。

`Concurrent` 模式下的任务执行会有时间片的体现,也就是当一个任务执行完成之后,先判断这一帧是否还有空闲时间,没有就挂起下一个任务的调度并记住当前被挂起的节点,让出控制权给浏览器执行更高优先级的任务。

在浏览器渲染完成一帧后,并判断当前帧是否还有剩余时间,如果有就恢复执行之前挂起的任务。

执行任务的函数是 `performConcurrentWorkOnRoot(...)`,一旦因为时间片中断了任务,就又会调用 `ensureRootIsScheduled(...)`函数,直到渲染完成。

在 `performConcurrentWorkOnRoot(...)` 函数中主要这样的定义:

```js
const shouldTimeSlice =
  !includesBlockingLane(root, lanes) &&
  !includesExpiredLane(root, lanes) &&
  (disableSchedulerTimeoutInWorkLoop || !didTimeout);

let exitStatus = shouldTimeSlice
  ? renderRootConcurrent(root, lanes)
  : renderRootSync(root, lanes);
```

在 `shouldTimeSlice` 中会检查当前任务的 `lane` 是否在已过期的 `expiredLanes`中或者`didTimeout` 表示当前任务是否过期,为了防止饥饿问题,最终结果则会返回 `false`,如果过期了就进入同步模式执行,也就是执行 `renderRootSync(...)` 函数,如果不是那么就会执行并发模式 `renderRootConcurrent(...)`。

`React` 应用解决任务饥饿问题就是通过上面的办法来解决的。

# 参考文章

- [React 中的优先级](https://juejin.cn/post/6916790300853665800#heading-4)
- [一眼看穿 react 源码(3)：不再神秘的优先级机制](https://juejin.cn/post/7039961115803009055#heading-25)
- [React 中的任务饥饿行为](https://juejin.cn/post/6923835053029982221)
- [图解 React](https://7kms.github.io/react-illustration-series/main/priority)
- [React 技术揭秘](https://react.iamkasong.com/concurrent/lane.html#%E8%A1%A8%E7%A4%BA%E4%BC%98%E5%85%88%E7%BA%A7%E7%9A%84%E4%B8%8D%E5%90%8C)
