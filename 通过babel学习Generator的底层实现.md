如果对生成器函数这些还不是很了解的可以通过这篇 [🍓 一文带你彻底搞懂 JavaScript 异步编程](https://juejin.cn/post/7178768412582084664) 进行学习。

# 深入理解生成器内部构成

在前面的章节讲解中我们已经知道了调用一个生成器不会实际执行它。相反,它创建了一个新的迭代器,通过该迭代器我们才能从生成器中请求值。在生成器生成或过度了一个值后,生成器会挂起执行并等待下一个请求的到来。在某种方面来说,生成器的工作更像是一个小程序,一个在状态中运动的状态机。

- **挂起开始**: 创建了浴缸生成器后,它最先以这种状态开始,其中的任何代码都未执行;
- **执行**: 生成器中的代码执行的状态。执行要么是刚开始,要么是从上次挂起的时候继续的。当生成器对应的迭代器调用了 `next(...)` 方 法,并且当前存在可执行的代码时,生成器都会转移到这个状态;
- **挂起让渡**: 当生成器在执行过程中遇到了一个 `yield` 表达式,它会创建一个包含着返回值的新对象，随后再挂起执行。生成器在这个状态暂停并等待继续执行。
- **完成**: 在生成器执行期间,如果代码执行到 `return` 语句或者全部代码执行完毕,生成器就进入该状态;

## 通过执行上下文跟踪生成器函数

首先让我们通过一张图更进一步补充一些知识,看看生成器究竟是如何耿穗执行环境上下文的,请看下图:

```js
function* generator() {
  yield "moment";
  yield "777";
}

const g = generator();

const result1 = g.next();
console.log(result1); // {value: 'moment', done: false}
const result2 = g.next();
console.log(result2); // {value: '777', done: false}
const result3 = g.next();
console.log(result3); // {value: undefined, done: true}
```

在上面的代码中 `const g = generator();` 创建生成器,处于挂起开始状态,在 `result1` 和 `result2` 中,`激活`和`重新激活`生成器,从挂起状态转为`执行状态`,直到执行到下一个 `yield` 语句终止,进而转为 `挂起让渡`,返回新对象,也就是输出的结果,在 `result3` 中后面没有代码可执行,转为完成状态,返回新对象。

上述代码可以有以下流程图所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d878bfbad2741b0ae3fdb5e63119193~tplv-k3u1fbpfcp-watermark.image?)

一般情况下,当程序从一个标准函数返回后,对应的 `执行环境上下文` 会从栈中弹出,并被完整地销毁。但在生成器中不是这样,相对应的 `generator` 会从栈中弹出,但由于 `g` 还保存着对它的引用,所以它并不会。你可以把它看作一种类似闭包的事物。在闭包中，为了在闭包创建的时候保证变量都可用，所以函数会对创建它的环境持有一个引用。

以这种方式,我们能保证只要函数还存在,环境及变量就都存在着。生成器,从另一个角度看,还必须恢复执行。由于所有函数的执行都被执行上下文所控制,故而迭代器保持了一个对当前执行环境的引用,保证只要迭代器还需要它的时候它都存在。

如果是一个普通的函数调用,这个语句会创建一个新的 `next(...)` 的执行环境上下文项并放入栈中,但是生成器并不是这样,对 `next(...)` 方法调用的表现也很不同。它会重新激活对应的执行上下文,在这个例子中是 `generator` 上下文,并把该上下文放入栈的顶部,从它上次离开的地方继续执行。

# 生成器底层实现

如果你想看到完整的可用的代码,你可以使用 [babel](https://babeljs.io/repl#?browsers=&build=&builtIns=false&corejs=3.21&spec=false&loose=false&code_lz=GYVwdgxgLglg9mAVAAgOYFMzoE4EMpzYAUAlMgN4BQyyAnjOgDYAmyARALZweZRsDc1OgxbsA7BIGUAvpUoQEAZyhpkAXjSYc-QqX5A&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=env%2Cstage-2%2Cstage-3&prettier=false&targets=&version=7.20.12&externalPlugins=&assumptions=%7B%7D)的,其用于将 `ES6` 的 `generator` 函数编译的目标为 `ES5`。

```js
function* generator() {
  yield "moment";
  yield "777";
}

const g = generator();
```

编译后带代码结构大概如下图所示:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9810afb6f3148238dd508d8b0c2fcae~tplv-k3u1fbpfcp-watermark.image?)

最终生成的代码有 500 行,首先我们先来调用一下 `_regeneratorRuntime(...)` 函数:

```js
console.log(_regeneratorRuntime());
```

通过浏览器控制台查看可以得知, `_regeneratorRuntime(...)` 函数返回了**8** 个函数,详情请看下图:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93206610b4a54ad9aeb7ff220ecb7999~tplv-k3u1fbpfcp-watermark.image?)

首先我们闲来开始第一段代码:

```js
var exports = {},
  Op = Object.prototype,
  hasOwn = Op.hasOwnProperty,
  defineProperty =
    Object.defineProperty ||
    function (obj, key, desc) {
      obj[key] = desc.value;
    },
  $Symbol = "function" == typeof Symbol ? Symbol : {},
  iteratorSymbol = $Symbol.iterator || "@@iterator",
  asyncIteratorSymbol = $Symbol.asyncIterator || "@@asyncIterator",
  toStringTagSymbol = $Symbol.toStringTag || "@@toStringTag";
```

当调用 `_regeneratorRuntime(...)` 函数,执行栈进入到该函数,首先在该函数作用域内声明了一个 `exports` 的空对象,再把 `Object` 的原型赋值给了 `Op`,并且如果 `Op.hasOwnProperty` 方法赋值给 `hasOwn`等等一系列声明和赋值操作,目的是为了做兼容。下图就是赋值之后各变量的值:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b67f149976b84b7190671b516a8474fd~tplv-k3u1fbpfcp-watermark.image?)

接下来我们开始执行第二段代码,来带 `try` 语句,调用了函数 `define({}, "")`,并最终返回一个空字符串:

```js
function define(obj, key, value) {
  return (
    Object.defineProperty(obj, key, {
      value: value,
      enumerable: !0,
      configurable: !0,
      writable: !0,
    }),
    obj[key]
  );
}
```

代码来到第三段,向`exports` 对象添加了一个 `wrap` 方法,这个方法就是一个 `wrap` 函数,但是并没有调用,紧接着又声明了一系列对象及函数:

```js
var ContinueSentinel = {};
function Generator() {}
function GeneratorFunction() {}
function GeneratorFunctionPrototype() {}
var IteratorPrototype = {};
```

在接下来的一段代码中又继续调用了 `define()` 函数,具体代码如下:

```js
define(IteratorPrototype, iteratorSymbol, function () {
  return this;
});
```

该调用返回一个函数,函数返回的是一个 `this`。

在下一段代码中,`Object.getPrototypeOf` 被赋值了给 `getProto`,该方法返回指定对象的原型`([[prototype]])`,通过 `getProto` 方法获取 `values([])` 调用的原型:

```js
function values(iterable) {
  if (iterable) {
    var iteratorMethod = iterable[iteratorSymbol];
    if (iteratorMethod) return iteratorMethod.call(iterable);
    if ("function" == typeof iterable.next) return iterable;
    if (!isNaN(iterable.length)) {
      var i = -1,
        next = function next() {
          for (; ++i < iterable.length; )
            if (hasOwn.call(iterable, i))
              return (next.value = iterable[i]), (next.done = !1), next;
          return (next.value = undefined), (next.done = !0), next;
        };
      return (next.next = next);
    }
  }
  return { next: doneResult };
}
```

在这里传进去的 `Iterator` 是一个 `[]`,此时 `iteratorMethod` 的值是迭代器的 `values(...)` 方法,最终返回 `iteratorMethod.call([])` 的调用。 而此时的 `NativeIteratorPrototype` 的值有下图所示:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f59075bc19464efaa303b97bddd2d987~tplv-k3u1fbpfcp-watermark.image?)

在这行代码中声明了一个 `Gp` 并赋值了一个空对象,而且原型方法也为空。

这时代码执行到了本函数的 `return` 语句,有以下定义:

```js
(GeneratorFunction.prototype = GeneratorFunctionPrototype),
  defineProperty(Gp, "constructor", {
    value: GeneratorFunctionPrototype,
    configurable: !0,
  }),
  defineProperty(GeneratorFunctionPrototype, "constructor", {
    value: GeneratorFunction,
    configurable: !0,
  }),
  (GeneratorFunction.displayName = define(
    GeneratorFunctionPrototype,
    toStringTagSymbol,
    "GeneratorFunction"
  )),
```

在这些定义中,主要将 `Gp` 的构造方法定义成了 `GeneratorFunctionPrototype()` 函数,将 `GeneratorFunctionPrototype` 的构造方法定义成了 `GeneratorFunction()` 函数,并且向 `GeneratorFunction` 添加了属性 `displayName`,其值为 `GeneratorFunction`。

在下一段代码中,有一个函数调用:

```js
defineIteratorMethods(AsyncIterator.prototype);
```

在这行代码中主要向 `AsyncIterator` 的原型添加了 `"next", "throw", "return"` 三个方法,紧接着继续对 `Gp` 做同样的方法,并且向 `Gp` 添加了私有方法 `Symbol(Symbol.toStringTag)`,它的值是 `"Generator"`,它有 `Object.prototype.toString` 内部方法访问,返回的值是 `"Generator"`。

```js
define(Gp, iteratorSymbol, function () {
      return this;
    }),
    define(Gp, "toString", function () {
      return "[object Generator]";
    }),
```

在上面的代码中定义了一个迭代方法,并且添加了一个 `toString()` 方法,返回的是一个 `Gp` 的类型,也就是前面我们所说的。到了这里,整个函数结束,继续回到刚开始的地方:

```js
var _marked = _regeneratorRuntime().mark(generator);
```

到这个时候,通过函数 `_regeneratorRuntime()` 的调用方法的 `mark()` 方法继续调用,该方法会向传进来的函数 `generator()` 添加隐式原型方法 `GeneratorFunctionPrototype()`,并且会将传进来的 `generator` 函数的原型设计为类型是 `Generator` 的空对象,详情请看下图:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9335b92e61344983b74a84479618a354~tplv-k3u1fbpfcp-watermark.image?)

在这个时候,`mark()` 函数的调用已经结束,代码执行到了最底部:

```js
var g = generator();
```

此时的执行栈来带了 `generator()`,这里调用了又 `_regeneratorRuntime()`,此时是来到了这个函数:

```js
_regeneratorRuntime = function _regeneratorRuntime() {
  return exports;
};
```

该函数返回了 `exports` 对象,返回的对象有以下行为:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46c8a0544f0846f0a81d77ad4d2feea4~tplv-k3u1fbpfcp-watermark.image?)

接着调用 `wrap()` 方法,通过一些简要的操作:

```js
var protoGenerator =
    outerFn && outerFn.prototype instanceof Generator ? outerFn : Generator,
  generator = Object.create(protoGenerator.prototype),
  context = new Context(tryLocsList || []);
```

在这个时候,`wrap()` 内部主要有以下值和方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33f73668657e4573a246b1ee72422fa5~tplv-k3u1fbpfcp-watermark.image?)

在这里遇到了 `return` 语句,调用 `makeInvokeMethod()` 方法,并且传入 `innerFn, self, context` 三个参数,在这个函数里面,首先定义了 `state`,作为生成器的状态,在下面的那个 `for` 循环中便是对我们使用生成器的一些方法的时候返回的结果的处理了,例如调用 `next()` 返回什么,状态是什么,调用 `return()` 返回的是什么,状态又是什么,并且最终把这个处理函数返回给 `value`,最终作为创建的 `generator` 对象的新属性 `_invoke` 的值,具体有以下代码:

```js
function wrap(innerFn, outerFn, self, tryLocsList) {
  var protoGenerator =
      outerFn && outerFn.prototype instanceof Generator ? outerFn : Generator,
    generator = Object.create(protoGenerator.prototype),
    context = new Context(tryLocsList || []);
  return (
    defineProperty(generator, "_invoke", {
      value: makeInvokeMethod(innerFn, self, context),
    }),
    console.log(generator),
    generator
  );
}
```

当我们在调用生成器函数中的 `next()`,`return()`,`throw()` 等方法的时候,实际上就是经过下面这个循环进行处理:

```js
for (context.method = method, context.arg = arg; ; ) {
  var delegate = context.delegate;
  if (delegate) {
    var delegateResult = maybeInvokeDelegate(delegate, context);
    if (delegateResult) {
      if (delegateResult === ContinueSentinel) continue;
      return delegateResult;
    }
  }
  if ("next" === context.method) context.sent = context._sent = context.arg;
  else if ("throw" === context.method) {
    if ("suspendedStart" === state) throw ((state = "completed"), context.arg);
    context.dispatchException(context.arg);
  } else "return" === context.method && context.abrupt("return", context.arg);
  state = "executing";
  var record = tryCatch(innerFn, self, context);
  if ("normal" === record.type) {
    if (
      ((state = context.done ? "completed" : "suspendedYield"),
      record.arg === ContinueSentinel)
    )
      continue;
    return { value: record.arg, done: context.done };
  }
  "throw" === record.type &&
    ((state = "completed"),
    (context.method = "throw"),
    (context.arg = record.arg));
}
```

我们通过那个 `log` 打印,发现和下面这段代码的打印是同一个对象:

```js
var g = generator();
console.log(g);
```

请看下图:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/964dce4e962a42ab97f65054cb842a19~tplv-k3u1fbpfcp-watermark.image?)

至此,所有代码结束,我们再通过完整的代码来具体运行一遍:

```js
function _typeof(obj) {
  "@babel/helpers - typeof";
  return (
    (_typeof =
      "function" == typeof Symbol && "symbol" == typeof Symbol.iterator
        ? function (obj) {
            return typeof obj;
          }
        : function (obj) {
            return obj &&
              "function" == typeof Symbol &&
              obj.constructor === Symbol &&
              obj !== Symbol.prototype
              ? "symbol"
              : typeof obj;
          }),
    _typeof(obj)
  );
}

function _regeneratorRuntime() {
  "use strict";
  _regeneratorRuntime = function _regeneratorRuntime() {
    return exports;
  };
  var exports = {},
    Op = Object.prototype,
    hasOwn = Op.hasOwnProperty,
    defineProperty =
      Object.defineProperty ||
      function (obj, key, desc) {
        obj[key] = desc.value;
      },
    $Symbol = "function" == typeof Symbol ? Symbol : {},
    iteratorSymbol = $Symbol.iterator || "@@iterator",
    asyncIteratorSymbol = $Symbol.asyncIterator || "@@asyncIterator",
    toStringTagSymbol = $Symbol.toStringTag || "@@toStringTag";
  function define(obj, key, value) {
    return (
      Object.defineProperty(obj, key, {
        value: value,
        enumerable: !0,
        configurable: !0,
        writable: !0,
      }),
      obj[key]
    );
  }
  try {
    define({}, "");
  } catch (err) {
    define = function define(obj, key, value) {
      return (obj[key] = value);
    };
  }
  function wrap(innerFn, outerFn, self, tryLocsList) {
    var protoGenerator =
        outerFn && outerFn.prototype instanceof Generator ? outerFn : Generator,
      generator = Object.create(protoGenerator.prototype),
      context = new Context(tryLocsList || []);
    return (
      defineProperty(generator, "_invoke", {
        value: makeInvokeMethod(innerFn, self, context),
      }),
      generator
    );
  }
  function tryCatch(fn, obj, arg) {
    try {
      return { type: "normal", arg: fn.call(obj, arg) };
    } catch (err) {
      return { type: "throw", arg: err };
    }
  }
  exports.wrap = wrap;
  var ContinueSentinel = {};
  function Generator() {}
  function GeneratorFunction() {}
  function GeneratorFunctionPrototype() {}
  var IteratorPrototype = {};
  define(IteratorPrototype, iteratorSymbol, function () {
    return this;
  });
  var getProto = Object.getPrototypeOf,
    NativeIteratorPrototype = getProto && getProto(getProto(values([])));
  NativeIteratorPrototype &&
    NativeIteratorPrototype !== Op &&
    hasOwn.call(NativeIteratorPrototype, iteratorSymbol) &&
    (IteratorPrototype = NativeIteratorPrototype);
  var Gp =
    (GeneratorFunctionPrototype.prototype =
    Generator.prototype =
      Object.create(IteratorPrototype));
  function defineIteratorMethods(prototype) {
    ["next", "throw", "return"].forEach(function (method) {
      define(prototype, method, function (arg) {
        return this._invoke(method, arg);
      });
    });
  }
  function AsyncIterator(generator, PromiseImpl) {
    function invoke(method, arg, resolve, reject) {
      var record = tryCatch(generator[method], generator, arg);
      if ("throw" !== record.type) {
        var result = record.arg,
          value = result.value;
        return value &&
          "object" == _typeof(value) &&
          hasOwn.call(value, "__await")
          ? PromiseImpl.resolve(value.__await).then(
              function (value) {
                invoke("next", value, resolve, reject);
              },
              function (err) {
                invoke("throw", err, resolve, reject);
              }
            )
          : PromiseImpl.resolve(value).then(
              function (unwrapped) {
                (result.value = unwrapped), resolve(result);
              },
              function (error) {
                return invoke("throw", error, resolve, reject);
              }
            );
      }
      reject(record.arg);
    }
    var previousPromise;
    defineProperty(this, "_invoke", {
      value: function value(method, arg) {
        function callInvokeWithMethodAndArg() {
          return new PromiseImpl(function (resolve, reject) {
            invoke(method, arg, resolve, reject);
          });
        }
        return (previousPromise = previousPromise
          ? previousPromise.then(
              callInvokeWithMethodAndArg,
              callInvokeWithMethodAndArg
            )
          : callInvokeWithMethodAndArg());
      },
    });
  }
  function makeInvokeMethod(innerFn, self, context) {
    var state = "suspendedStart";
    return function (method, arg) {
      if ("executing" === state)
        throw new Error("Generator is already running");
      if ("completed" === state) {
        if ("throw" === method) throw arg;
        return doneResult();
      }

      for (context.method = method, context.arg = arg; ; ) {
        var delegate = context.delegate;
        if (delegate) {
          var delegateResult = maybeInvokeDelegate(delegate, context);
          if (delegateResult) {
            if (delegateResult === ContinueSentinel) continue;
            return delegateResult;
          }
        }
        if ("next" === context.method)
          context.sent = context._sent = context.arg;
        else if ("throw" === context.method) {
          if ("suspendedStart" === state)
            throw ((state = "completed"), context.arg);
          context.dispatchException(context.arg);
        } else
          "return" === context.method && context.abrupt("return", context.arg);
        state = "executing";
        var record = tryCatch(innerFn, self, context);
        if ("normal" === record.type) {
          if (
            ((state = context.done ? "completed" : "suspendedYield"),
            record.arg === ContinueSentinel)
          )
            continue;
          return { value: record.arg, done: context.done };
        }
        "throw" === record.type &&
          ((state = "completed"),
          (context.method = "throw"),
          (context.arg = record.arg));
      }
    };
  }
  function maybeInvokeDelegate(delegate, context) {
    var methodName = context.method,
      method = delegate.iterator[methodName];
    if (undefined === method)
      return (
        (context.delegate = null),
        ("throw" === methodName &&
          delegate.iterator["return"] &&
          ((context.method = "return"),
          (context.arg = undefined),
          maybeInvokeDelegate(delegate, context),
          "throw" === context.method)) ||
          ("return" !== methodName &&
            ((context.method = "throw"),
            (context.arg = new TypeError(
              "The iterator does not provide a '" + methodName + "' method"
            )))),
        ContinueSentinel
      );
    var record = tryCatch(method, delegate.iterator, context.arg);
    if ("throw" === record.type)
      return (
        (context.method = "throw"),
        (context.arg = record.arg),
        (context.delegate = null),
        ContinueSentinel
      );
    var info = record.arg;
    return info
      ? info.done
        ? ((context[delegate.resultName] = info.value),
          (context.next = delegate.nextLoc),
          "return" !== context.method &&
            ((context.method = "next"), (context.arg = undefined)),
          (context.delegate = null),
          ContinueSentinel)
        : info
      : ((context.method = "throw"),
        (context.arg = new TypeError("iterator result is not an object")),
        (context.delegate = null),
        ContinueSentinel);
  }
  function pushTryEntry(locs) {
    var entry = { tryLoc: locs[0] };
    1 in locs && (entry.catchLoc = locs[1]),
      2 in locs && ((entry.finallyLoc = locs[2]), (entry.afterLoc = locs[3])),
      this.tryEntries.push(entry);
  }
  function resetTryEntry(entry) {
    var record = entry.completion || {};
    (record.type = "normal"), delete record.arg, (entry.completion = record);
  }
  function Context(tryLocsList) {
    (this.tryEntries = [{ tryLoc: "root" }]),
      tryLocsList.forEach(pushTryEntry, this),
      this.reset(!0);
  }
  function values(iterable) {
    if (iterable) {
      var iteratorMethod = iterable[iteratorSymbol];
      if (iteratorMethod) return iteratorMethod.call(iterable);
      if ("function" == typeof iterable.next) return iterable;
      if (!isNaN(iterable.length)) {
        var i = -1,
          next = function next() {
            for (; ++i < iterable.length; )
              if (hasOwn.call(iterable, i))
                return (next.value = iterable[i]), (next.done = !1), next;
            return (next.value = undefined), (next.done = !0), next;
          };
        return (next.next = next);
      }
    }
    return { next: doneResult };
  }
  function doneResult() {
    return { value: undefined, done: !0 };
  }
  return (
    (GeneratorFunction.prototype = GeneratorFunctionPrototype),
    defineProperty(Gp, "constructor", {
      value: GeneratorFunctionPrototype,
      configurable: !0,
    }),
    defineProperty(GeneratorFunctionPrototype, "constructor", {
      value: GeneratorFunction,
      configurable: !0,
    }),
    (GeneratorFunction.displayName = define(
      GeneratorFunctionPrototype,
      toStringTagSymbol,
      "GeneratorFunction"
    )),
    (exports.isGeneratorFunction = function (genFun) {
      var ctor = "function" == typeof genFun && genFun.constructor;
      return (
        !!ctor &&
        (ctor === GeneratorFunction ||
          "GeneratorFunction" === (ctor.displayName || ctor.name))
      );
    }),
    (exports.mark = function (genFun) {
      return (
        Object.setPrototypeOf
          ? Object.setPrototypeOf(genFun, GeneratorFunctionPrototype)
          : ((genFun.__proto__ = GeneratorFunctionPrototype),
            define(genFun, toStringTagSymbol, "GeneratorFunction")),
        (genFun.prototype = Object.create(Gp)),
        genFun
      );
    }),
    (exports.awrap = function (arg) {
      return { __await: arg };
    }),
    defineIteratorMethods(AsyncIterator.prototype),
    define(AsyncIterator.prototype, asyncIteratorSymbol, function () {
      return this;
    }),
    (exports.AsyncIterator = AsyncIterator),
    (exports.async = function (
      innerFn,
      outerFn,
      self,
      tryLocsList,
      PromiseImpl
    ) {
      void 0 === PromiseImpl && (PromiseImpl = Promise);
      var iter = new AsyncIterator(
        wrap(innerFn, outerFn, self, tryLocsList),
        PromiseImpl
      );
      return exports.isGeneratorFunction(outerFn)
        ? iter
        : iter.next().then(function (result) {
            return result.done ? result.value : iter.next();
          });
    }),
    defineIteratorMethods(Gp),
    define(Gp, toStringTagSymbol, "Generator"),
    define(Gp, iteratorSymbol, function () {
      return this;
    }),
    define(Gp, "toString", function () {
      return "[object Generator]";
    }),
    (exports.keys = function (val) {
      var object = Object(val),
        keys = [];
      for (var key in object) keys.push(key);
      return (
        keys.reverse(),
        function next() {
          for (; keys.length; ) {
            var key = keys.pop();
            if (key in object)
              return (next.value = key), (next.done = !1), next;
          }
          return (next.done = !0), next;
        }
      );
    }),
    (exports.values = values),
    (Context.prototype = {
      constructor: Context,
      reset: function reset(skipTempReset) {
        if (
          ((this.prev = 0),
          (this.next = 0),
          (this.sent = this._sent = undefined),
          (this.done = !1),
          (this.delegate = null),
          (this.method = "next"),
          (this.arg = undefined),
          this.tryEntries.forEach(resetTryEntry),
          !skipTempReset)
        )
          for (var name in this)
            "t" === name.charAt(0) &&
              hasOwn.call(this, name) &&
              !isNaN(+name.slice(1)) &&
              (this[name] = undefined);
      },
      stop: function stop() {
        this.done = !0;
        var rootRecord = this.tryEntries[0].completion;
        if ("throw" === rootRecord.type) throw rootRecord.arg;
        return this.rval;
      },
      dispatchException: function dispatchException(exception) {
        if (this.done) throw exception;
        var context = this;
        function handle(loc, caught) {
          return (
            (record.type = "throw"),
            (record.arg = exception),
            (context.next = loc),
            caught && ((context.method = "next"), (context.arg = undefined)),
            !!caught
          );
        }
        for (var i = this.tryEntries.length - 1; i >= 0; --i) {
          var entry = this.tryEntries[i],
            record = entry.completion;
          if ("root" === entry.tryLoc) return handle("end");
          if (entry.tryLoc <= this.prev) {
            var hasCatch = hasOwn.call(entry, "catchLoc"),
              hasFinally = hasOwn.call(entry, "finallyLoc");
            if (hasCatch && hasFinally) {
              if (this.prev < entry.catchLoc) return handle(entry.catchLoc, !0);
              if (this.prev < entry.finallyLoc) return handle(entry.finallyLoc);
            } else if (hasCatch) {
              if (this.prev < entry.catchLoc) return handle(entry.catchLoc, !0);
            } else {
              if (!hasFinally)
                throw new Error("try statement without catch or finally");
              if (this.prev < entry.finallyLoc) return handle(entry.finallyLoc);
            }
          }
        }
      },
      abrupt: function abrupt(type, arg) {
        for (var i = this.tryEntries.length - 1; i >= 0; --i) {
          var entry = this.tryEntries[i];
          if (
            entry.tryLoc <= this.prev &&
            hasOwn.call(entry, "finallyLoc") &&
            this.prev < entry.finallyLoc
          ) {
            var finallyEntry = entry;
            break;
          }
        }
        finallyEntry &&
          ("break" === type || "continue" === type) &&
          finallyEntry.tryLoc <= arg &&
          arg <= finallyEntry.finallyLoc &&
          (finallyEntry = null);
        var record = finallyEntry ? finallyEntry.completion : {};
        return (
          (record.type = type),
          (record.arg = arg),
          finallyEntry
            ? ((this.method = "next"),
              (this.next = finallyEntry.finallyLoc),
              ContinueSentinel)
            : this.complete(record)
        );
      },
      complete: function complete(record, afterLoc) {
        if ("throw" === record.type) throw record.arg;
        return (
          "break" === record.type || "continue" === record.type
            ? (this.next = record.arg)
            : "return" === record.type
            ? ((this.rval = this.arg = record.arg),
              (this.method = "return"),
              (this.next = "end"))
            : "normal" === record.type && afterLoc && (this.next = afterLoc),
          ContinueSentinel
        );
      },
      finish: function finish(finallyLoc) {
        for (var i = this.tryEntries.length - 1; i >= 0; --i) {
          var entry = this.tryEntries[i];
          if (entry.finallyLoc === finallyLoc)
            return (
              this.complete(entry.completion, entry.afterLoc),
              resetTryEntry(entry),
              ContinueSentinel
            );
        }
      },
      catch: function _catch(tryLoc) {
        for (var i = this.tryEntries.length - 1; i >= 0; --i) {
          var entry = this.tryEntries[i];
          if (entry.tryLoc === tryLoc) {
            var record = entry.completion;
            if ("throw" === record.type) {
              var thrown = record.arg;
              resetTryEntry(entry);
            }
            return thrown;
          }
        }
        throw new Error("illegal catch attempt");
      },
      delegateYield: function delegateYield(iterable, resultName, nextLoc) {
        return (
          (this.delegate = {
            iterator: values(iterable),
            resultName: resultName,
            nextLoc: nextLoc,
          }),
          "next" === this.method && (this.arg = undefined),
          ContinueSentinel
        );
      },
    }),
    exports
  );
}

var _marked = _regeneratorRuntime().mark(generator);

function generator() {
  return _regeneratorRuntime().wrap(function generator$(_context) {
    while (1)
      switch ((_context.prev = _context.next)) {
        case 0:
          _context.next = 2;
          return "moment";
        case 2:
          _context.next = 4;
          return "777";
        case 4:
        case "end":
          return _context.stop();
      }
  }, _marked);
}
var g = generator();

console.log(g.next()); // {value: 'moment', done: false}
console.log(g.return()); // {value: undefined, done: true}
console.log(g.next()); // {value: undefined, done: true}
```

# 总结

- 要想了解生成器函数的底层原理的代码 `500行` 说实话对我刚开始学习阅读源码来说很多,但是认真去阅读发现也并不是很难,但是前提需要你对生成器函数的使用有较深的理解,以及对 `Object` 对象的一些原型方法和原型有较深入的理解。
