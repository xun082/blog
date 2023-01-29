在前面的文章中讲到了如何开启 `webpack-dev-server` 以及 `npm start` 背后的原理,那么这篇文章我们将来讲解一下 `HMR(模块热替换)`,上一篇文章地址 [在终端里输入 npm start 后都发生了啥](https://juejin.cn/post/7187368174952644665)

# 模块热替换

在开始之前,首先我们应该了解一下什么是 `模块热替换`。

在早期开发工具还比较简单和匮乏的年代,调试代码的方式基本都是改代码一刷新网页查看结果--然后再改代码,这反复地修改和测试。后来,一些 Web 开发框架和工具提供了更便捷的方式,就是只要检测到代码改动就会自动重新构建,然后触发网页刷新。

这种一般称为 `live reload`,`Webpack` 则在 `live reload` 的基础上又进了一步,可以让代码在网页不刷新的前提下得到最新的改动,甚至可以让我们不需要拍重新发起请求就能看到更新后的效果。这个就是模块热替换 (`Hot Module Replacement`) 功能。

`HMR` 对于大型应用尤其适用。试想一个辅助的系统每改动一个地方都要经历资源重新构建、网络请求、浏览器重新渲染等过程,怎么也要爱几秒甚至几十秒的时间才能完成。况且我们调试的页面可能位于很深的层级,每次都要通过一些人为操作才能验证结果,其效率是非常低下的。

`HMR` 功能会在应用程序运行过程中,替换、添加或删除模块,而无需重新加载整个页面,主要是通过以下几种方式,来显著加快开发速度:

- 保留在完全重新加载页面期间丢失的应用程序状态;
- 只更新变更内容,以节省宝贵的开发时间;
- 在源代码中 `CSS/JS` 产生修改时,会立刻在浏览器中进行更新,这几乎相当于在浏览器 `devtools` 直接更改样式;

# HMR 的基本使用

从  `webpack-dev-server` v4.0.0 开始,热模块替换是默认开启的。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6e775bdf2ae436bb296070abe7e2664~tplv-k3u1fbpfcp-watermark.image?)

整个结论我们可以从 `webpack-dev-server` 中的 `Server` 类中的 `initialize()` 方法可以得出结论,只需要将 `webpack.config.js` 中的 `hot` 设置为 `true` 即可。

```js
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  entry: "./src/index.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "./dist"),
  },
  mode: "development",
  devServer: {
    hot: true, // 开启模块热替换
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: "./index.html",
    }),
  ],
};
```

上面配置产生的结果是 `Webpack` 会为每个模块绑定一个 `module.hot` 对象,这个对象包含了 `HMR` 的 `API`。借助这些 `API` 我们不仅可以实现对特定模块开启或关闭 `HMR`,也可以添加热替换之外的逻辑。

下面是一些使用的例子:

```js
// utils.js
export function sum() {
  return 1 + 2;
}

export function multiplication(x, y) {
  return x * y;
}

// index.js
import { multiplication, sum } from "./utils";

console.log(sum());

console.log(multiplication(3, 6));

if (module.hot) {
  module.hot.accept();
}
```

`index.js` 作为引用的入口,那么我们就可以把调用 `HMR` API 的代码放在该入口中,这样 `HMR` 对于 `index.js` 和其依赖的所有模块都会生效。当发现有模块发生变动时,`HMR` 会使用在当前浏览器环境下重新执行一遍 `index.js` (包括其依赖的内容),但是页面本身不会刷新。

例如当我修改了 `index.js` 的内容时,浏览器会有以下输出:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7478a7d2a21c47e6a6da4b37116d7e7f~tplv-k3u1fbpfcp-watermark.image?)

而当我们修改了 `utils.js` 文件的时候,`index.js` 文件也会跟着更新,因为 `index.js` 有对该模块的引用:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a8a63803ae7a4aeba5c1ce7e5db2594f~tplv-k3u1fbpfcp-watermark.image?)

大多数情况下,还是建议用于的开发者使用第三方提供的 `HMR` 解决方案,因为 `HMR` 触发过程中可能会出现很多预想不到的问题,导致模块更新后应用的表现和正常加载的表现不一致。

`Webpack` 社区还提供了许多其他 `loader` 和示例,可以使 `HMR` 与各种框架和库平滑地进行交互:

- [react-refresh](https://www.npmjs.com/package/react-refresh):实时编辑 `React`组件而不会丢失它们的状态;
- [Vue Loader](https://github.com/vuejs/vue-loader): 此 loader 支持 vue 组件的 HMR，提供开箱即用体验;
- [Elm Hot webpack Loader](https://github.com/klazuka/elm-hot-webpack-loader): 支持 Elm 编程语言的 HMR;
- [Angular HMR](https://github.com/gdi2290/angular-hmr): 没有必要使用 loader！直接修改 NgModule 主文件就够了，它可以完全控制 HMR API;
- [Svelte Loader](https://github.com/sveltejs/svelte-loader): 此 loader 开箱即用地支持 Svelte 组件的热更新;

# HMR 原理

在本地开发环境下,浏览器就是客户端,`webpack-dev-server` 相当于我们的服务端。`HMR` 的核心就是客户端从服务端拉取更新后的资源,更准确的说法就是 `HMR` 卡去的不是整个资源文件,而是 `chunk diff`,即 `chunk` 需要更新的部分,关于 `chunk` 的概念如下解释:

> chun 字面的意思是代码亏啊,在 Webpack 中可以理解成被抽象和包装后的一些模块。它就像一个装着很多文件的文件袋,里面的文件就是各个模块,Webpack 在外面加了一层包裹,从而形成了 chunk,根据具体配置不同,一个工程打包时可能会产生一个或多个 chunk。

当在终端运行 `webpack serve` 的命令后,`webpack-dev-server` 会调用 `Server` 类中的 `initialize()` 方法(在上一篇文章有讲到),在这里面有一个 `compiler,forEach` 的语句:

```js
compilers.forEach((compiler) => {
  this.addAdditionalEntries(compiler);

  const webpack = compiler.webpack || require("webpack");

  new webpack.ProvidePlugin({
    __webpack_dev_server_client__: this.getClientTransport(),
  }).apply(compiler);

  // TODO remove after drop webpack v4 support
  compiler.options.plugins = compiler.options.plugins || [];

  if (this.options.hot) {
    const HMRPluginExists = compiler.options.plugins.find(
      (p) => p.constructor === webpack.HotModuleReplacementPlugin
    );

    if (HMRPluginExists) {
      this.logger.warn(
        `"hot: true" automatically applies HMR plugin, you 
	don't have to add it manually to your webpack configuration.`
      );
    } else {
      // Apply the HMR plugin
      const plugin = new webpack.HotModuleReplacementPlugin();
      plugin.apply(compiler);
    }
  }
});
```

并且在每一个循环中都调用了 `addAdditionalEntries()` 方法,这里主要做的是将 `websocket` 需要的一些参数组装拼接到 `client/index.js` 的 `require()` 请求路径中:

```js
additionalEntries.push(
  `${require.resolve("../client/index.js")}?${webSocketURLStr}`
);
```

随后又将 `webpack/hot/dev-server` 的请求存入 `additionalEntrires` 中,最后调用 `webpack.EntryPlugin` 将他们存入到模块请求 `hooks` 中:

```js
if (typeof webpack.EntryPlugin !== "undefined") {
  for (const additionalEntry of additionalEntries) {
    new webpack.EntryPlugin(compiler.context, additionalEntry, {
      // eslint-disable-next-line no-undefined
      name: undefined,
    }).apply(compiler);
  }
}
```

这里的 `webpack.EntryPlugin` 主要的作用就是在编译的时候添加一个入口 `chunk`,然后等待 `compiler` 的 `make` 这个 `hook` 被调用时再执行 `compilation` 的 `addEntry` 方法,然后又电泳 `handleModuleCreate` 函数,正式开始构建内容。

我们再回看 `initialize()` 方法中有段代码 `new webpack.HotModuleReplacementPlugin` ,这里的主要作用是如果 `hot` 设置为 `true` 就把 `HMR` 挂载到 `plugin` 上,开启模块热替换,具体代码如下操作:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/692ff8d50e0642cfad5e60b5c9467d08~tplv-k3u1fbpfcp-watermark.image?)

那么我们再看看 `webpack-dev-server/client/index.js` 文件做了些什么?

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f4a597353c874b3ba4a3e9bc48213318~tplv-k3u1fbpfcp-watermark.image?)

在 `hot` 方法中,client 端接收 server 端发送 `hot` 指定,开启 HMR 模式;
在 `hash` 方法中,接收 `webpack` 每次编译的最新 `hash`,浏览器端将 `sendStats` 发送过来的 `hash` 值保存下来,它将会用到后来的模块热更新;

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a98126e692a4a62ad282f19924bdd73~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2391b7061d4f4badad81f701c74067f5~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89ecc3c336b7459fa4e2b36e9442a9ff~tplv-k3u1fbpfcp-watermark.image?)

在 `hash` 方法中,当 `hash` 消息发送完成后,`socket` 还会发送一次 `ok` 的信息告知 `webpack-dev-server`,第一次编译不需要执行 `reloadApp`,因为第一次全部编译的代码并没有额外的代码需要更新;

`webpack-dev-server/client/index.js` 当接收到 `type` 为 `hash` 消息后会将 `hash` 值暂存起来，当接收到 `type` 为 `ok` 的消息后对应用执行 `reload` 操作，如下图所示，`hash` 消息是在 `ok` 消息之前:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cd9733995e24a3dbebddc2aa4715ce9~tplv-k3u1fbpfcp-watermark.image?)

那么我们看看 `reloadApp` 内部都干了些啥?这里主要做的事情是通过 `hotEmitter.emit` 触发 `webpackHotUpdate` 事件:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7c1ccc530d441b196bc285c2ff13679~tplv-k3u1fbpfcp-watermark.image?)

我们再通过浏览器控制台查看,当我们修改了代码之后会触发这个方法,代码中的 `log.info()` 的内容会输出在浏览器控制台上:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/33af7838d9364667b71dc1c8dc92a118~tplv-k3u1fbpfcp-watermark.image?)

当 `hash` 值发生变化时,`webpack` 会监听到浏览器端 `webpackHotUpdate` 的消息,这里 `webpack` 的定义文件为 `webpack/hot/dev-server.js`,将新模块 `hash` 值传到客户端 `HMR` 核心中的 `HotModuleReplacement.runtime`,文件目录为 `webpack/lib/hmr/HotModuleReplacement.runtime.js`,这个文件主要的功能我们稍后再看,在 `dev-server` 文件中,主要调用 `check` 方法检测更新,并判断是浏览器刷新还是模块热更新,如果是浏览器刷新的话,会自动调用 `window.location.reload()`,如果代码出错,也会调用 `window.location.reload()`,最后 `HMR` 重新开启。

当监听到文件变化,`dev-server` 可以监听到哪个文件发生变化,当我们修改 `utils.js` 文件时,`index.js` 的内容也要改变,所以会有以下输出:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/32b36a35450447c492908ecb6b9725ae~tplv-k3u1fbpfcp-watermark.image?)

`check` 代码如下图所示:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3818c6c22d4040bbb4930886740439fc~tplv-k3u1fbpfcp-watermark.image?)

我们再来看看 `HotModuleReplacement.runtime.js` 这个文件主要做的事情是什么、

这个文件的主要功能是 `hotCheck` 方法,首先我们来看看 `$hmrDownloadManifest$` 是个什么东西,我们在 `hotCheck` 方法打印一下:

```js
console.log($hmrDownloadManifest$);
```

通过查看控制台,返回的主要有以下内容:

```js
__webpack_require__.hmrM = () => {
  if (typeof fetch === "undefined")
    throw new Error("No browser support: need fetch API");
  return fetch(__webpack_require__.p + __webpack_require__.hmrF()).then(
    (response) => {
      if (response.status === 404) return; // no update available
      if (!response.ok)
        throw new Error(
          "Failed to fetch update manifest " + response.statusText
        );
      return response.json();
    }
  );
};
```

其中 `__webpack_require__.p` 的值为 `http://localhost:8080/`,而 `__webpack_require__.hmrF()` 有以下的定义:

```js
__webpack_require__.hmrF = () =>
  "main." + __webpack_require__.h() + ".hot-update.json";

__webpack_require__.h = () => "376a09b5cf1d64e19a6d";
```

`__webpack_require__.h` 中的字符串正是当前返回的 `hash` 值,全部拼接到一起就是 `[name].[hash].hot-update.json`,所以 `hotCheck` 的主要功能是提供客户端 `HMR` 运行时下载发生变化的 `chunk` 文件,将最新代码加载到本地:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee76a92e4dd24097b404e2f4abd449de~tplv-k3u1fbpfcp-watermark.image?)

我们通过打印,`hotCheck` 最终返回的结果是以下这样的结果:

```js
__webpack_require__.hmrC.jsonp = function (
  chunkIds,
  removedChunks,
  removedModules,
  promises,
  applyHandlers,
  updatedModulesList
) {
  applyHandlers.push(applyHandler);
  currentUpdateChunks = {};
  currentUpdateRemovedChunks = removedChunks;
  currentUpdate = removedModules.reduce(function (obj, key) {
    obj[key] = false;
    return obj;
  }, {});
  currentUpdateRuntime = [];
  chunkIds.forEach(function (chunkId) {
    if (
      __webpack_require__.o(installedChunks, chunkId) &&
      installedChunks[chunkId] !== undefined
    ) {
      promises.push(loadUpdateChunk(chunkId, updatedModulesList));
      currentUpdateChunks[chunkId] = true;
    } else {
      currentUpdateChunks[chunkId] = false;
    }
  });
  if (__webpack_require__.f) {
    __webpack_require__.f.jsonpHmr = function (chunkId, promises) {
      if (
        currentUpdateChunks &&
        __webpack_require__.o(currentUpdateChunks, chunkId) &&
        !currentUpdateChunks[chunkId]
      ) {
        promises.push(loadUpdateChunk(chunkId));
        currentUpdateChunks[chunkId] = true;
      }
    };
  }
};
```

在模块热替换的过程中,还有一个重要的步骤就是在更新之前会删除过期的模块和依赖,这个功能的代码为 `webpack/lib/hmr/JavascriptHotModuleReplacement.runtime.js`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e4b2c81e0064d398023b702cb41b52b~tplv-k3u1fbpfcp-watermark.image?)

这些代码的主要功能是移除缓存中的模块,移除过期依赖中不需要使用的处理方法,以及移除所有子元素的引用和模块子组件中过时的依赖项。

处理完这个步骤之后,将新模块代码添加到 `modules` 中,当下次调用 `__webpack-require__` 方法的时候,就是获取到了新的模块代码了:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79f0c6eeed1440f884423e8a2eff24b1~tplv-k3u1fbpfcp-watermark.image?)

# 参考文章

- [webpack5 源码分析-模块热替换/更新](https://zhuanlan.zhihu.com/p/564046307)
- [Webpack HMR 原理解析](https://www.zhihu.com/search?type=content&q=hmr%E5%8E%9F%E7%90%86)

# 总结

本文主要讲解了 `HMR` 的基本使用,以及讲解了 `HMR` 的原理,下面通过一张图片来看看 `HMR` 的整体流程:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cda57d7b224c49dc9e52528a9d2fc0e2~tplv-k3u1fbpfcp-watermark.image?)

总的来说,`WDS` 不能没有 `HMR` ,`HMR` 需要 `WDS`,天生一对,完美结合... 😍😍😍
