# 前言

在前面的内容纯属**胡说八道**,如果想要看正文,请直接滚动条往下拉,以省下宝贵的时间继续卷。

要想对 `JavaScript` 代码进行打包,我们可以依赖 `webpack` 对我们的帮助我们完成这一件事情。要想使用 `webpack`,首先需要我们安装 `webpack`,首先对项目进行初始化:

```shell
npm init -y
```

生成配置配置文件:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e151ddbbf50541828deb3df96b84d5a5~tplv-k3u1fbpfcp-watermark.image?)

要想使用 `webpack`,首先需要安装 `webpack` 以及 `webpack-cli`,这里还有 `html-webpack-plugin`,用于生成 `html` 模板,具体命令如下:

```bash
npm install  webpack webpack-cli html-webpack-plugin -D
```

依赖安装完成之后,我们需要在跟目录上面创建一个名为 `webpack.config.js`,当然,你也可以创建其他文件名的 `js` 文件,并添加以下配置:

```js
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  entry: "./src/index.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "./dist"),
  },
  mode: "production",
  plugins: [
    new HtmlWebpackPlugin({
      template: "./index.html",
    }),
  ],
};
```

为了能执行打包命令,我们在 `package.json` 文件中的 `script` 中添加这一段命令:

```json
"build": "webpack"
```

如果你使用的 `webpack` 配置文件名为其他的则需要在该命令中添加相对应的路径,否则在执行命令的时候 `webpack-cli` 则无法找到相关的配置。

接下来我们创建在根目录下创建一个 `src` 目录,在目录下面创建一个 `index.js` 文件,作为整个项目的入口,并在文件中添加一些自己想写的代码:

```js
console.log("hello webpack");
```

此时,在命令行中输入一下命令:

```bash
npm run build
```

此时文件被输出出来了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6cfe819320d444e9ed5c13a0874e022~tplv-k3u1fbpfcp-watermark.image?)

生成的文件是根据我们前面的 `webpack` 配置文件中生成的,通过运行 `index.html` 文件,`hello webpack` 也被输出在浏览器控制台上面:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1d00b9a4419460a83efab602627e2cf~tplv-k3u1fbpfcp-watermark.image?)

但是这个方法存在弊端,当我们对源代码进行修改的时候,它并不会对我所修改的代码进行重新编译,要想能够在浏览器中看到最新效果,你必须通过重新执行 `npm run build` 打包才可以,开发效率极其低下。

# watch

`webpack` 有一个 `watch` 属性可以监听文件的变化,当监听到文件变化,当它们修改后会重新编译,要启用 `watch`,你只需要在 `webpack.config.js` 文件中设置 `watch:true`即可,详情如下:

```js
module.exports = {
  ...,
  watch: true,
};
```

或者在 `package.json` 文件中 `script` 下修改添加 `--watch`.代码如下所示:

```json
"build": "webpack --watch"
```

控制台再次执行 `npm run build`,你会发现这次控制台不会结束了,会一直开启着,详情请看下图:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e0757ef89044862b028d2d7e52ac7c8~tplv-k3u1fbpfcp-watermark.image?)

当我们修改文件内容的时候,`watch` 会监听着文件变化,如下图终端所示:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43f904b6bf45473a83632d515381f96b~tplv-k3u1fbpfcp-watermark.image?)

并且浏览器上的内容也会随着变化而变化,你也可以配置 `watchOptions` 来使你的项目更快,例如:

```js
watchOptions: {
  aggregateTimeout: 6000,
  ignored: /node_modules/,
},
```

在上面的配置中,当第一个文件更改,会在重新构建前增加延迟,它会将这段时间内的所有更改都聚合到一次重新构建中,以毫秒为单位。对于某些系统,监听大量文件会导致大量的 `CPU` 或内存占用。可以使用正则排除像  `node_modules`  如此庞大的文件夹。

尽管有这些配置,效率仍然不高,编译成功后,都会生成新的文件,并且需要开启 `live-server`,但是这个插件属于 `vscode` 的,但是在其他编辑器上并没有,并不属于 `webpack`。

`live-server` 每次都会重新刷新整个页面,并不能保存当前页面的状态,还会编译所有的代码。

# webpack-dev-server 的基本使用

`webpack-dev-server`(简称 `WDS`) 为你提供了一个基本的 `web server`,并且具有 `live reloading`(实时重新加载)功能。要想使用使用你首先要安装该依赖:

```js
npm install --save-dev webpack webpack-dev-server
```

继续在 `package.json` 文件中 `script` 下修改添加一段命令.代码如下所示:

```json
 "start": "webpack serve --open"
```

完成之后,在终端下输入以下命令执行:

```bash
npm start
```

`WDS` 会自动为为你自动开启电脑上的默认浏览器,并且默认的端口为 `http://localhost:8080/`,在终端里有如下输出:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3e5bc716bed4ab58c25cc10063d12af~tplv-k3u1fbpfcp-watermark.image?)

在浏览器上有如下输出:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ea7d71e5c404ea2a188ee1fd71fc834~tplv-k3u1fbpfcp-watermark.image?)

默认开启 `live Reload` ,当代码发生改变时会自动重新对代码进行编译。

# WDS 原理

`webpack-dev-server` 启动了一个使用 `express` 的 HTTP 服务,这个服务器与客户端采用 `WebSocket` 通信协议,当原始文件发生改变,`webpack-dev-server` 会实时编译,但是对 `index.html` 的修改不会做出处理。

通过查看 `webpack-dev-server` 源码中的 `package.json` 文件,我们发现这里定义了一个 `bin` 字段,那么这个 `bin` 有什么用呢?

`bin` 字段是命令名到本地文件名的映射。当我们使用 `npm` 或者 `yarn` 命令安装包时,如果该包的 `package.json` 文件有 `bin` 字段，就会在` node_modules` 文件夹下面的 `.bin` 目录中复制了 bin 字段链接的执行文件。我们在调用执行文件时,可以不带路径,直接使用命令名来执行相对应的执行文件。

也就是说,当我们在终端中输入 `npm install webpack-dev-server -D` 的时候,会在 `node_modules/.bin` 目录下生成了三个文件:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d20f64f71d441fdbfe2964d1e5d52b1~tplv-k3u1fbpfcp-watermark.image?)

这三个文件中,其中的 ·`cmd` 是 `windows` 中默认的可执行文件,当我们不添加后缀名时,自动根据 `pathext` 查找文件。

当我们执行 `npm start` 的时候,会在 `node_modules/.bin` 目录下找到 `webpack-dev-server.cmd` 目录,因为这个文件是 `windows` 的批处理脚本:

```js
@ECHO off
GOTO start
:find_dp0
SET dp0=%~dp0
EXIT /b
:start
SETLOCAL
CALL :find_dp0

IF EXIST "%dp0%\node.exe" (
  SET "_prog=%dp0%\node.exe"
) ELSE (
  SET "_prog=node"
  SET PATHEXT=%PATHEXT:;.JS;=;%
)

endLocal & goto #_undefined_# 2>NUL || title %COMSPEC% & "%_prog%"  "%dp0%\..\webpack-dev-server\bin\webpack-dev-server.js" %*
```

在这写代码里的最后一行 `"%dp0%\..\webpack-dev-server\bin\webpack-dev-server.js"` 命令中,通过软连接链接到 `node_modules/webpack-dev-server/bin` 中的 `webpack-dev-server.js` 目录。

所以当我们运行 `npm start` 的时候,也就是相当于运行 `node_modules/.bin/webpack-dev-server.cmd serve` 命令,并最终以 `webpack-dev-server/bin/webpack-dev-server.js`  就是整个命令行的入口,通过这里,它会自动给你开启 `webpack-cli`,详情请看下图:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6540c5ed3cfe4e8fbf5f355fbca56c29~tplv-k3u1fbpfcp-watermark.image?)

所以当我们没有安装 `webpack-cli` 的时候运行 `npm start` 时会有以下提示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed2311c9ca994c38846fbab9d58e18ea~tplv-k3u1fbpfcp-watermark.image?)

在这里 `webpack` 就会基于我们 `webpack.config.js` 里创建一个 `compiler`,然后基于 `compiler` 和 `devServer` 相关配置生成一个 `WebpackDevServer` 实例,该实例会启动一个`expores` 服务来帮我们监听静态资源变化并更新。

在 `webpack-dev-server` 源代码中,有一个 `Server.js` 的目录,创建了一个 `Server` 类,用于启动的是 `start(...)` 方法中通过 `socket` 监听一个端口,默认使用的是 `8080`,初始化 `client` 和 `dev-server`,以 `p[lugin` 的形式挂载到 `compiler` 上,添加 `hooks` 插件,实例化 `express` 服务等等,有以下代码,详情可自行查看,可以通过安装依赖的方式,也可以到 `GitHub`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/210c089e14354e4090384dc63c9c169b~tplv-k3u1fbpfcp-watermark.image?)

接下来我们看看 `await this initialize(...)` 都干了些啥?详情请看下列代码(省略了后面部分):

```js
  async initialize() {
    if (this.options.webSocketServer) {
      const compilers =
        /** @type {MultiCompiler} */
        (this.compiler).compilers || [this.compiler];

      compilers.forEach((compiler) => {
        this.addAdditionalEntries(compiler);
        const webpack = compiler.webpack || require("webpack");
        new webpack.ProvidePlugin({
          __webpack_dev_server_client__: this.getClientTransport(),
        }).apply(compiler);

        compiler.options.plugins = compiler.options.plugins || [];
        if (this.options.hot) {
          const HMRPluginExists = compiler.options.plugins.find(
            (p) => p.constructor === webpack.HotModuleReplacementPlugin
          );
          if (HMRPluginExists) {
            this.logger.warn(
              `"hot: true" automatically applies HMR plugin, you don't have to add it manually to your webpack configuration.`
            );
          } else {
            // Apply the HMR plugin
            const plugin = new webpack.HotModuleReplacementPlugin();
            plugin.apply(compiler);
          }
        }
      });

      if (
        this.options.client &&
        /** @type {ClientConfiguration} */ (this.options.client).progress
      ) {
        this.setupProgressPlugin();
      }
    }
```

在 `initialize()` 方法中还有以下方法的调用:

```js
this.setupHooks();
this.setupApp();
this.setupHostHeaderCheck();
this.setupDevMiddleware();
this.setupBuiltInRoutes();
this.setupWatchFiles();
this.setupWatchStaticFiles();
this.setupMiddlewares();
this.createServer();
```

在这里,主要做的事情是将 `client` 以 `plugin` 的形式挂载到 `compiler`,如果存在 `HMR`,则开启 `HMR`,通过 `this.setupWatchFiles()` 监听文件变化。

在 `this.setupHooks()` 中,主要做的事情就是在 `webpack` 的 `done` 钩子上挂了个给客户端广播消息的回调,通过这个回调就知道工程代码有更新,这时候客户端就会发送请求给 `express` 服务去请求最新的 `webpack` 打包的代码,详情请看以下代码:

```js
  setupHooks() {
    this.compiler.hooks.invalid.tap("webpack-dev-server", () => {
      if (this.webSocketServer) {
        this.sendMessage(this.webSocketServer.clients, "invalid");
      }
    });
    this.compiler.hooks.done.tap(
      "webpack-dev-server",
      /**
       * @param {Stats | MultiStats} stats
       */
      (stats) => {
        if (this.webSocketServer) {
          this.sendStats(this.webSocketServer.clients, this.getStats(stats));
        }

        /**
         * @private
         * @type {Stats | MultiStats}
         */
        this.stats = stats;
      }
    );
  }
```

当客户端接收到 `websocket` 广播的消息后,会触发`reloadApp`方法(`webpack`打包时注入进去的)`reloadApp`会根据广播消息里的更新类型选择是页面更新 `liveReload` 还是模块更新 `HMR`,通过测试,发现每次修改正是都会经过 `sendMessage` 方法。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e749813c958f47c78e025cb98b86cc96~tplv-k3u1fbpfcp-watermark.image?)

上图正是 `webpack-dev-server` 的整个流程图,到这来,这篇文章的内容也就讲完了,如有错误,烦请批评指出。

# 参考文献

- [# webpack-dev-server 运行原理](https://juejin.cn/post/7075260685864665119#heading-3)
- [WDS 源码](https://github.com/webpack/webpack-dev-server)

# 往期文章

[💯💯💯 Map、Set、WeakMap、WeakSet 看这一篇就够](https://juejin.cn/post/7183237715217874999 "https://juejin.cn/post/7183237715217874999")

[🍓 一文带你彻底搞懂 JavaScript 异步编程](https://juejin.cn/post/7178768412582084664 "https://juejin.cn/post/7178768412582084664")

[一文让你彻底搞懂 JS 垃圾回收机制](https://juejin.cn/post/7173644980240515085 "https://juejin.cn/post/7173644980240515085")

[💯💯💯 你是会 async/await,但是你知道它的原理以及如何捕捉到错误吗](https://juejin.cn/post/7186224254990417979)

[学习事件循环,这篇文章值得你看看](https://juejin.cn/post/7186575283082756154)
