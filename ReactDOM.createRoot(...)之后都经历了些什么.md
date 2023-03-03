在日常的开发中,我们不禁会想,为什么别人写出来的代码这么优雅,而我写出来的代码就一行代码百行报错,并且整体代码像一堆屎山,报错的时候也难以找出报错原因,那么在接下来的文章中让我们来学习 React 源码来对其进一步学习吧。

# 什么是 JSX

什么是 `JSX`,在 [React 官方文档](https://react.docschina.org/docs/introducing-jsx.html) 中是这样定义的: 它是一个 `JavaScript` 的语法扩展,在 `React` 中配合使用 `JSX` 可以很好地描述 UI 应该呈现出他应该有的交互形式。

它像模板语言,但他具有 `JavaScript` 的全部功能。实际上,`JSX` 仅仅是 `React.createElement(component, props, ...children)` 函数的语法糖。

那么,`JSX` 的语法是怎么样的 `JavaScript` 中生效的呢,让我们看下面的章节。

# JSX 本质

在 `React` 中,`JSX` 会被 `Babel` 编译为普通的 `JavaScript` 函数,在 `Babel` 中我们输入以下代码:

```jsx
<div className="moment">hello</div>
```

通过 `Babel` 转译最终会生成以下代码:

```js
React.createElement(
  "div",
  {
    className: "moment",
  },
  "hello"
);
```

我们再看看更复杂的代码:

```jsx
<div className="moment">
  <div key="1" className="test">
    1111
  </div>
  <div key="2" className="niu">
    2222
  </div>
</div>
```

最终会被编译成这样:

```js
React.createElement(
  "div",
  {
    className: "moment",
  },
  React.createElement(
    "div",
    {
      key: "1",
      className: "test",
    },
    "1111"
  ),
  React.createElement(
    "div",
    {
      key: "2",
      className: "niu",
    },
    "2222"
  )
);
```

好了,现在我们了解了在 `React` 中编写的代码 `JSX` 最终会被编译成上面的样子,那么接下来我们该正式进入主题了。

# React 应用包结构

要想阅读 `React` 源码,那么有必要先了解一下 `React` 中的包结构,在 `@18.1.0` 版本中,与 web 相关的核心包共有 4 个,其中主要有以下包:

**react**: `React` 基础包,只提供定义 `React` 组件的必要函数,在编写`react`应用的代码时, 大部分都是调用此包的 `api`,例如 `useState`、`memo`、`useEffect` 等。

**react-dom**: `React` 渲染器之一,是 `React` 与 `Web` 平台连接的桥梁,将 `react-reconciler` 中的运行结果输出到 `web` 界面上,在编写 `React` 应用的代码时,大多数场景下,能用到此包的就是一个入口函数`ReactDOM.render(<App/>, document.getElementById('root'))`, 其余使用的 api, 基本是`react`包提供的。

**react-reconciler**: 该包的主要功能有以下 4 个方面:

- 输入: 暴露`api`函数(如: `scheduleUpdateOnFiber`), 供给其他包(如`react`包)调用;
- 注册调度任务: 与调度中心(`scheduler`包)交互,注册调度任务`task`,等待任务回调;
- 执行任务回调: 在内存中构造出 fiber 树, 同时与与渲染器(react-dom)交互, 在内存中创建出与 fiber 对应的 DOM 节点;
- 输出: 与渲染器(react-dom)交互, 渲染 DOM 节点;

**scheduler**: 时间管理大师,核心任务就是执行回调,该回调函数由 `react-reconciler` 提供,通过控制回调函数的执行时机,来达到任务分片的目的,实现可中断渲染。

> 在接下来的代码解析当中,为了代码的简洁性,会删除一下无关紧要的代码,完整代码可以可以查看文章底部的链接获取。

# createElement

在上面的内容中,`JSX` 会被 `Babel` 会被编译成一个 `React.createElement` 函数,该函数在 `React` 包中如下:

```js
// react\packages\react\src\ReactElement.js

function createElement(type, config, children) {
  // 属性名称,用于后面的 for 循环
  let propName;
  // 存储 React Element 中的普通元素属性,但是不包含 key ref self source
  const props = {};
  // 对 dom 存入的key 值
  let key = null;
  // 通过 ref 获取到的 dom 实例
  let ref = null;
  let self = null;
  let source = null;

  // 如果 config 不为空
  if (config != null) {
    if (hasValidRef(config)) {
      // 将 config.ref 属性提取到 ref 变量中
      ref = config.ref;
    }
    // 会将 key 转换为字符串
    if (hasValidKey(config)) {
      key = "" + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    // 通过遍历 config 中的属性并添加到 props
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

  /**
   * 处理子元素,将第三个及之后的参数挂载到 props.children 属性中
   * 如果子元素是多个 props.children 是数组对象
   * 如果子元素是一个 props.children 是对象
   */
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    props.children = childArray;
  }

  /**
   * 如果当前处理的是组件,看组件身上是否有 defaultProps 属性
   * 这个属性存储的是 props 对象中的默认值
   */
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }

  // 最后返回一个调用ReactElement执行方法，并传入刚才处理过的参数
  return ReactElement(
    type, // HTML 标签
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props
  );
}
```

在 `ReactElement` 函数中的主要作用是返回一个对象,并标记为 `React Element`,具体代码如下:

```js
const ReactElement = function (type, key, ref, self, source, owner, props) {
  const element = {
    // 标记这是个 React Element
    $$typeof: REACT_ELEMENT_TYPE,

    type: type,
    key: key,
    ref: ref,
    props: props,
    _owner: owner,
  };

  return element;
};
```

在我们的项目中有这样的组件,其代码如下:

```jsx
const App = () => {
  return (
    <div className="moment">
      <div key="1" className="test">
        1111
      </div>
      <div key="2" className="niu">
        2222
      </div>
    </div>
  );
};

console.log(App());
```

然后再通过查看控制台,有以下输出:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9d9c222b9b848a184be1d5e60212be9~tplv-k3u1fbpfcp-watermark.image?)

在上面的图片中可以看出,`ReactElement` 返回的东西正是我们前面中讲到的,而 `_source` 是  `babel-preset-react`注入的调试信息,可以提供更有用的错误信息,能具体到代码的所在文件及行数。

到这里,也许你对 `JSX` 会有一个清除的概念了。

# createRoot

在我们使用 `create-react-app` 创建的项目,整个项目的入口有这样的一段代码:

```js
const root = ReactDOM.createRoot(document.getElementById("root"));
```

函数 `createRoot` 的定义存放与 `react` 包中的 `react-test\src\react\packages\react-dom\src\client\ReactDOMRoot.js`,在该方法中,初始化了一系列变量并调用 `createContainer` 方法:

```js
createContainer(
  container,
  ConcurrentRoot,
  null,
  isStrictMode,
  concurrentUpdatesByDefaultOverride,
  identifierPrefix,
  onRecoverableError,
  transitionCallbacks
);
```

在上面的函数调用中,传入的参数如下图所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c92ece08b524ca5a22cd62eac9367be~tplv-k3u1fbpfcp-watermark.image?)

在上面传入的第二个参数 `ConcurrentRoot` 意为当前 `React` 应用中的模式为 `ConcurrentRoot`。

在 `createContainer` 方法中是直接返回 `createFiberRoot` 函数的调用:

```ts
// react\packages\react-reconciler\src\ReactFiberReconciler.old.js

export function createContainer(
  containerInfo: Container,
  tag: RootTag,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: (error: mixed) => void,
  transitionCallbacks: null | TransitionTracingCallbacks
): OpaqueRoot {
  const hydrate = false;
  const initialChildren = null;
  return createFiberRoot(
    containerInfo,
    tag,
    hydrate,
    initialChildren,
    hydrationCallbacks,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onRecoverableError,
    transitionCallbacks
  );
}
```

其实在上面这个方法也没干啥事,也只是把参数传进来又去调用 `createFiberRoot` 函数,真的是层层套娃,其实阅读源码难也就难在这里,好了,接下来我们看看 `createFiberRoot` 又干了些啥事,具体代码实现如下图所示:

```ts
// react\packages\react-reconciler\src\ReactFiberRoot.old.js

export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  initialChildren: ReactNodeList,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: null | ((error: mixed) => void),
  transitionCallbacks: null | TransitionTracingCallbacks,
): FiberRoot {
  /** 创建 FiberRoot */
  const root: FiberRoot = (new FiberRootNode(
    containerInfo,
    tag,
    hydrate,
    identifierPrefix,
    onRecoverableError,
  ): any);

  console.log(root);

   /** 设置服务端渲染回调 */
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  /** 设置过渡回调 */
  if (enableTransitionTracing) {
    root.transitionCallbacks = transitionCallbacks;
  }

  /** 创建 HostRootFiber */
  const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );

  /** 将 HostRootFiber 挂载到 FiberRoot 的 current 属性上 */
  root.current = uninitializedFiber;

  /** 将 HostRootFiber 的 stateNode 设置为 FiberRoot */
  uninitializedFiber.stateNode = root;

  /** 设置 HostRootFiber 的 memoizedState */
  if (enableCache) {
    const initialCache = createCache();
    console.log(initialCache);
    retainCache(initialCache);

    root.pooledCache = initialCache;
    retainCache(initialCache);
    const initialState: RootState = {
      element: initialChildren,
      isDehydrated: hydrate,
      cache: initialCache,
      transitions: null,
      pendingSuspenseBoundaries: null,
    };
    uninitializedFiber.memoizedState = initialState;
  } else {
    const initialState: RootState = {
      element: initialChildren,
      isDehydrated: hydrate,
      cache: (null: any), // not enabled yet
      transitions: null,
      pendingSuspenseBoundaries: null,
    };
    uninitializedFiber.memoizedState = initialState;
  }

  // 初始化updateQueue，对于RootFiber，queue.share.pending上面存储着element
  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

在上面的代码中,通过 `new FiberRootNode` 返回一个 `fiberRoot` 实例,在这个构造函数中定义了很多很好玩的东西,具体可以自己去看,这里就不详细解答了。

在这个时候,我们打印一下 `new FiberRootNode` 返回的实例是一个什么样的值,详情请看下图:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9abb4ee4c77d40ae8bc2657d420af675~tplv-k3u1fbpfcp-watermark.image?)

现在我们只需关心这两个玩意,`containerInfo` 是容器的意思,也是整个项目的入口,为真实 `DOM`,那另外一个是什么呢,我们等下再看,代码我们继续往下看:

```js
// 创建 HostRootFiber
const uninitializedFiber = createHostRootFiber(
  tag,
  isStrictMode,
  concurrentUpdatesByDefaultOverride
);
```

这里又调用了 `createHostRootFiber` 函数,该函数的主要作用有以下几个方面:

- 设置 `React Fiber` 的工作模式,分别有:`Concurrent`模式、严格模式和 `createRootStrictEffectsByDefault` 模式；
- 创建 `Fiber`;

紧接着,我们再看下面的代码:

```js
// 将 HostRootFiber 挂载到 FiberRoot 的 current 属性上
root.current = uninitializedFiber;

// 将 HostRootFiber 的 stateNode 设置为 FiberRoot
uninitializedFiber.stateNode = root;
```

在上面的这个代码就像一个相互引用,我们再次通过上面的打印查看,发现有这样的输出:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d77b2f2abba412a9c99e14d8ad8ddf3~tplv-k3u1fbpfcp-watermark.image?)

它们的关系图请看下图所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef56743d80ea4c31b562f0e099a18d7b~tplv-k3u1fbpfcp-watermark.image?)

在控制台中还有这样一个信息:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/331e2069c64e4c97bcca734092f7949f~tplv-k3u1fbpfcp-watermark.image?)

`child` 指向的是子元素,而 `reture` 指向的父元素,两者相互指向,而 `FiberNode` 为当前的 `Fiber` 树的根节点,它没有父节点,它的 `return` 值为空也就讲得通了。

那么问题来了,`fiberRoot` 和 `rootiber` 的区别又是什么呢?

在我们开发的项目中,你可以理解为每一个组件都有一个由 `rootFiber` 构成的 `fiber` 树,但整个应用的根节点只有一个,那么就是 `fiberRoot`,`fiberRoot`可以指向不同的 `rootFiber` 以渲染不同的页面，具体请看下面的图片:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99a8cfb3d8024916ab32ed1dcf96ed62~tplv-k3u1fbpfcp-watermark.image?)

通过上面的图片我们可以得知,`fiberRoot` 它是整个应用的根节点, 绑定在`container._reactRootContainer`, 也就是绑定在真实 DOM 节点的 `_reactRootContainer` 属性上。

在 `createFiberRoot` 函数的最后调用了 `initializeUpdateQueue` 用于初始化 `rootFiber.updateQueue`:

```js
initializeUpdateQueue(uninitializedFiber);
```

要想了解该函数的作用,那么我们先来了解一下该函数是怎么定义的:

```ts
// react\packages\react-reconciler\src\ReactFiberClassUpdateQueue.old.js

export function initializeUpdateQueue<State>(fiber: Fiber): void {
  const queue: UpdateQueue<State> = {
    baseState: fiber.memoizedState,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: null,
      interleaved: null,
      lanes: NoLanes,
    },
    effects: null,
  };
  fiber.updateQueue = queue;
}
```

该函数的主要作用是用于初始化 `UpdateQueue`,这里主要有什么作用我们在后面的文章会讲到。

到这里,这个函数也执行完毕了,并且最后返回整个 `fiberRoot`,但是这里还会有一个知识点还没有讲到,就是 **双缓存**。

代码又回到 `createRoot` 函数这里,在这里有调用了 `markContainerAsRoot` 函数:

```js
// react\packages\react-dom\src\client\ReactDOMRoot.js

// 将 fiberNode 挂载到 container 对象上,也就是跟目录
markContainerAsRoot(root.current, container);
```

我们继续来了解一下该函数的内部构成:

```js
export function markContainerAsRoot(hostRoot: Fiber, node: Container): void {
  node["__reactContainer$" + randomKey] = hostRoot;
}
```

这个函数的主要作用是给 `containerInfo` 也就是 `div#root` 生成一个随机的 `key`,最后通过 `container.nodeType` 来判断节点的类型:

```ts
  const rootContainerElement: Document | Element | DocumentFragment =
    container.nodeType === COMMENT_NODE
      ? (container.parentNode: any)
      : container;
```

因为此时的 `container.nodeType` 是 `div#root` ,所以它的值为 `1`,而 `COMMENT_NODE` 的值为 `8`,所以最终返回的是 `container` 的本身,通过控制台打印,它完完整整的生成了一个 `DOM` 树:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997057cf4a004b159673cb916ef9bbfc~tplv-k3u1fbpfcp-watermark.image?)

而这正和我们在 `react` 项目中编写的代码基本一致:

```jsx
const App = () => {
  function foo() {
    console.log(1111111111);
  }
  return (
    <div className="moment">
      <div key="1" className="test" onClick={foo}>
        1111
      </div>
      <div key="2" test="111" className="niu">
        2222
      </div>
    </div>
  );
};
```

但是发现少了一样东西,我们定义的 `onclick` 事件去哪了,先别慌,往下看代码最后又继续调用 `listenToAllSupportedEvents` 函数,该函数的主要功能是会给 `div#root` 节点注册浏览器支持的所有原生事件,比如`onclick`等,而我们定义的事件会被关联到真实的 `DOM` 的 `__reactProps${key}` 上,如下图所示:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54170cca5c89450388f069d62d9c8686~tplv-k3u1fbpfcp-watermark.image?)

该函数最后返回 `fiberRoot`。到这里整个 `fiberRoot` 的创建也就结束了。

# 双缓存 Fiber 树

在 `React` 中最低同时会存在两颗 `fiber` 树,当前屏幕上显示内容对应的 `fiber` 树称为 `current fiber` 树,正在构建的 `fiber` 树称为 `workInProgress Fiber` 树,通过控制台可以看到如下输出

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ffc546a68444c4992d8806a24e468c6~tplv-k3u1fbpfcp-watermark.image?)

`React` 应用的根节点通过使 `current` 指针在不同 `Fiber` 树的 `rootFiber` 间切换来完成 `current Fiber` 树指向的切换。

即当 `workInProgress Fiber树` 构建完成交给`Renderer`渲染在页面上后，应用根节点的`current`指针指向 `workInProgress Fiber` 树，此时 `workInProgress Fiber` 树就变为 `current Fiber` 树。

每次状态更新都会产生新的 `workInProgress Fiber` 树，通过 `current` 与 `workInProgress` 的替换，完成 `DOM` 更新。

而最终的 `fiber` 树如下图所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/484260a7c23d44f2add62f5b0c3844d4~tplv-k3u1fbpfcp-watermark.image?)

好了到这里本篇文章也就讲解完了,如需学习后续内容敬请关注。

# 参考文章

- [react 技术揭秘](https://react.iamkasong.com/process/doubleBuffer.html#%E5%8F%8C%E7%BC%93%E5%AD%98fiber%E6%A0%91)
- [翻译翻译，什么叫 ReactDOM.createRoot](https://zhuanlan.zhihu.com/p/409225031)
- [图解 React](https://7kms.github.io/react-illustration-series/main/macro-structure)

# 总结

没有总结,请各位大佬自行总结。

欲知后事如何,请看下回分解.
