阅读本篇文章之前,建议先阅读一下这篇 [💯💯💯 通过 babel 学习 Generator 的底层实现](https://juejin.cn/post/7186133102740111397),因为本篇文章的内容是基于这篇的。

大家面试的时候有很大的可能会被问到 `async/await` 原理,很难答出来满意的,那么下面我就来演示一下面试场景:

- 面试官:你知道 `async/await` 的原理吗?
- 面试者:知道啊,它的底层是 `Genertator`;
- 面试官:还有呢?
- 面试者:没有了;
- 面试官:那么你能模拟实现一下 `async/await`吗?
- 面试者:不能啊,我不能定义关键字,你能实现一个给我看看吗?
- 面试官:......看你小子思路还挺清晰的嘛,你被录用了,月薪三千,有空先去把厕所打扫一下吧!

那么接下来的这篇文章,我们通过 `async/await` 的底层实现,已经模拟实现一个 `async/await` 来加深对 `async/await` 的认识,以及学习一下是如何捕捉错误的。

# async/await 原理解析

相信大家都知道,`async/await` 是基于 `Generator` 的,而 `async/await` 又是到目前为止解决异步编程最优雅的方式了,那么在 `Generator` 内部它是如何实现 `"暂停"` 然后又恢复执行的?

对于这个问题,`Generator` 其实就是 `JavaScript` 语法层面上对协程的支持,协程就是主程序和子协程直接控制权的切换,并伴随通信的过程,从 `Generator` 的角度来讲,`yield`,`next`就是通信接口,`next` 是主协程向子协程通信,两者相互交替。

在维基百科中有这样的定义:

> 协程（英语：coroutine）是计算机程序的一类组件，推广了协作式多任务的子例程，允许执行被挂起与被恢复。相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。协程更适合于用来实现彼此熟悉的程序组件，如协作式多任务、异常处理、事件循环、迭代器、无限列表和管道。

其中 `"允许执行被挂起与被恢复"`就是很好的解释,协程可以通过 `yield`（取其“让步”之义而非“出产”）来调用其它协程，接下来的每次协程被调用时，从协程上次 `yield` 返回的位置接着执行，通过 yield 方式转移执行权的协程之间不是调用者与被调用者的关系，而是彼此对称、平等的。

接下来我们继续打开 [Babel 官网](https://www.babeljs.cn/repl#?browsers=&build=&builtIns=false&corejs=3.21&spec=false&loose=false&code_lz=IYZwngdgxgBAZgV2gFwJYHsL3egFAShgG8AoGGKTEZGAJwFMQEAbZARhgF4ZgB3YVDTYBuMhSo0GTVgCYuPfoJgzRYhsgS0sUluxgBqOo10qSAXxIk4OAsKA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=env%2Creact&prettier=false&targets=&version=7.20.12&externalPlugins=&assumptions=%7B%7D)继续编译以下代码:

```js
async function foo() {
  const result1 = await 1;
  const result2 = await 2;

  return result1 + result2;
}

foo();
```

但是,细心的你一定会发现,这些代码和前面的 `Generator` 的代码几乎一毛一样,所以这也就是为什么会说 `async/await` 的底层实现是基于 `Generator` 了,详情请看下图:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68529465d7e44f08862e93af169e29f5~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f25ebdd9a5c4739a2be9d11119a81c5~tplv-k3u1fbpfcp-watermark.image?)

# async/await 实现

由于 `async/await` 是关键字,我们并没有定义关键字的能力,所以我们可以通过函数来模拟实现,首先实现迭代处理函数:

```js
function asyncGeneratorStep(gen, resolve, reject, _next, _throw, key, arg) {
  try {
    var info = gen[key](arg);
    var value = info.value;
  } catch (error) {
    reject(error);
    return;
  }
  if (info.done) {
    // 迭代器完成
    resolve(value);
  } else {
    // 将所有值转变为 Promise 形式
    // 同一以 Promise.resolve() 的方法返回并且递归调用 next() 函数
    // 直到 done === true 为止
    Promise.resolve(value).then(_next, _throw);
  }
}
```

再来实现模拟异步函数:

```js
function _asyncToGenerator(fn) {
  return function () {
    // this 指向全局
    var self = this,
      args = arguments;
    // 将返回值promise化
    return new Promise(function (resolve, reject) {
      var gen = fn.apply(self, args);
      // 执行下一步
      function _next(value) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "next", value);
      }
      // 抛出异常
      function _throw(err) {
        asyncGeneratorStep(gen, resolve, reject, _next, _throw, "throw", err);
      }
      // 第一次触发
      _next(undefined);
    });
  };
}
```

最后通过一个案例进行测试,完美通过:

```js
const asyncFunc = _asyncToGenerator(function* () {
  const e = yield new Promise((resolve) => {
    setTimeout(() => {
      resolve("e");
    }, 1000);
  });
  const a = yield Promise.resolve("a");
  const d = yield "d";
  const b = yield Promise.resolve("b");
  const c = yield Promise.resolve("c");
  return [a, b, c, d, e];
});

asyncFunc().then((res) => {
  console.log(res); // ['a', 'b', 'c', 'd', 'e']
});
```

# async/await 异常捕捉

在开始之前,我们先来理清一个问题,就是为什么要进行错误处理?

由于 `JavaScript` 是一个单线程语言,假如不进行错误处理,会导致代码直接报错而无法执行,显然这不是我们想要的,那么我们应该怎么去捕捉到这个错误呢,`async.await` 本身并没有提供这个机制给我们进行处理,那么接下来我们将讲解四个方案,一步一步,由浅入。

## try...catch

最简单的办法自然是 `try.catch` 了,具体代码如下所示:

```js
async function foo() {
  try {
    var result = await Promise.reject(new Error(111));
  } catch (e) {
    console.log(e);
  }
  return result;
}

foo();
```

通过 `catch` 能清楚的捕获到错误,具体到哪行:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58b5895687e34585a73f14dc3e65caa5~tplv-k3u1fbpfcp-watermark.image?)

但是有一个问题,是每一个 `await` 都进行捕获吗,还是部分进行捕获,那又对哪部分进行捕获,难道你知道哪段代码在接下来运行的时候会报错吗,这显然是一个很大的问题,但是 `async/await` 的错误又该如何捕获呢?

## await-to-js

其实已经有一个库 [await-to-js](https://github.com/scopsy/await-to-js)已经帮我们做了这件事，我们可以看看它是怎么做的，它的源码只有`短短十几行`，我们应该读读它的源码，学学它的思想

```ts
/**
 * @param { Promise } promise
 * @param { Object= } errorExt - Additional Information you can pass to the err object
 * @return { Promise }
 */
export function to<T, U = Error>(
  promise: Promise<T>,
  errorExt?: object
): Promise<[U, undefined] | [null, T]> {
  return promise
    .then<[null, T]>((data: T) => [null, data])
    .catch<[U, undefined]>((err: U) => {
      if (errorExt) {
        const parsedError = Object.assign({}, err, errorExt);
        return [parsedError, undefined];
      }

      return [err, undefined];
    });
}

export default to;
```

to 函数返回一个值,这个值是一个数组,数组之中有两个元素。如果索引为 `0` 的元素不为空值,说明该请求报错,如果索引 `0` 的元素为空值说明该请求没有报错,也就是成功。

那么这个 `to` 函数怎么使用呢,由于我们已经把 `await-to-js` 的源码拷贝下来了,那么我们就不引入库, 就直接使用了,具体示例如下:

```js
async function foo() {
  const [error, result] = await to(Promise.resolve(1));
  console.log(result); // 1
  console.log(error); // null

  const [error1, result1] = await to(Promise.reject("error错误了"));
  console.log(result1); // undefined
  console.log(error1); // error错误了

  console.log("代码还能正常执行");
}

foo();
```

但是这个处理方法据说类似于 `Golang` 一直被吐槽的对象,如今又移植到了 `JavaScript`上,那么还有什么方法进行捕获呢?

## babel 插件

首先在自己的项目的 `babel` 文件中的 `plugins` 中添加 [babel-plugin-await-add-trycatch](https://www.npmjs.com/package/babel-plugin-await-add-trycatch),具体配置可参考下图:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/997aaf4665e445a78fb2fcde9f6f9fec~tplv-k3u1fbpfcp-watermark.image?)

使用了该插件之后,在编译的过程中,实际上就是给每一个 `await` 添加了 `try...catch` 语句,我们通过具体的代码来尝试一下:

```js
const handle = async () => {
  const result = await Promise.reject("error");
  console.log(result);
};

handle();
```

再通过浏览器查看打印的报错信息:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4be6c317532441d39455b9f86a8ffb8c~tplv-k3u1fbpfcp-watermark.image?)

报错的信息包含了报错的文件路径,报错的方法,报错的具体原因,通过该插件我们能通过该插件能对 `async/await` 进行错误处理,那么还有没有更好的方法呢,答案是有的。

## addEventListener 全局捕获

我们可以通过 `windwo.addEventListener` 来捕获到 `async/await` 抛出的错误,并在这个方法的回调中的 `event` 接收这个报错信息,具体代码实现如下:

```js
window.addEventListener("unhandledrejection", (e) => {
  e.preventDefault();
  console.log(e.reason);
});
```

在这个 `reason` 里就存在着这些报错信息,其中包括报错的文件路径,报错信息,已经报错的具体行数,详情请看如下示例:

```js
window.addEventListener("unhandledrejection", (e) => {
  e.preventDefault();
  console.log(e.reason);
});

async function foo() {
  const result = await Promise.reject(new Error(111));
  return result;
}

foo();
```

具体的输出如下所示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/16d34329cebd4a3e92eedc432beed777~tplv-k3u1fbpfcp-watermark.image?)

好了,本篇文章到这来也就结束了。

# 文章推荐

[💯💯💯 Map、Set、WeakMap、WeakSet 看这一篇就够](https://juejin.cn/post/7183237715217874999)

[🍓 一文带你彻底搞懂 JavaScript 异步编程](https://juejin.cn/post/7178768412582084664)

[一文让你彻底搞懂 JS 垃圾回收机制](https://juejin.cn/post/7173644980240515085)

[深入浅出 es6 模块化](https://juejin.cn/post/7166046272300777508)
