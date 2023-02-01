# npm 介绍

`npm` 是随同 `Node.js` 一起安装的包管理工具,能解决 `Node.js` 代码部署上的很多问题,常见的使用场景有以下几种:

- 允许用户从 `NPM` 服务器下载别人编写的第三方包到本地使用;
- 许用户从 `NPM` 服务器下载并安装别人编写的命令行程序到本地使用;
- 允许用户将自己编写的包或命令行程序上传到 `NPM` 服务器供别人使用;

在现在的前端世界里,几乎已经离不开 `npm` 了,其提供的依赖安装、卸载、升级、发布等一条龙服务,使我们在日常的开发效率提升了不少。

`npm` 制定了一个包规范,所谓规范就是一些格式和约定,比如作为一个 `npm` 包中根目录必须包含一个 `package.json` 文件,并约定从 `package.json` 文件里读取这个包的所有信息,包括它的名字、版本号、它依赖于哪些别的包等;

并且创建一个 `node_modules` 目录专门用来存放第三方依赖,Node 为此提供的支持是内置的 `require()`方法默认会到这个目录下去检索模块,而无需手动指定路径。有了这些规范,一个包的开发、依赖安装、发布等都步骤都标准化了,省心省力。

我们接下来就让我们开始学习 `npm` 之旅吧!!!

# packages 和 modules 的区别

一个 `packages` 是一个文件,在该文件的根目录,必须包含一个 `package.json` 文件,通过该文件,你可以将包发布到 `registry`。

`modules` 是 `node_modules` 目录下可以被 `require()` 函数加载的任何文件和目录,这样的称为模块,一个 `packages` 也是模块,只不过是由多个模块组成的包。

为了考研通过 `require()` 函数加载到模块,它必须具有以下特征之一:

- 文件夹中包含 `package.json` 文件,并且包含一个 `main` 字段作为入口文件;
- 一个 `JavaScript` 文件;

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a59704ff67b47ecb1912e4b662f316a~tplv-k3u1fbpfcp-watermark.image?)

在上面的图中 `moment.js` 是一个 `modules`,但不是一个 `packages`,在外部引入可以使用。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a22aaeeeb624625b982c78f92403df6~tplv-k3u1fbpfcp-watermark.image?)

在上面的例子中,在 `node_modules` 目录下手动创建一个名为 `moment` 的文件,并且添加一个 `package.json` 的文件,在文件中添加 `main` 字段,设置 `main.js` 作为文件的入口。

在这里,调用 `const foo = require("moment");` 实际上加载的是 `moment` 文件夹下的 `main.js` 文件,所以 `foo` 函数被正常调用,最终输出 `hi`。

# package.json 详解

在前面的内容中我们多次说到 `package.json` 的文件,那么我们应该怎么创建这个文件呢,`npm` 给我们提供了一个命令 `npm init`,在终端输入该命令,会有以下问题并要求你输入回答:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb95975c19ec48aa97f4ea2a9546085b~tplv-k3u1fbpfcp-watermark.image?)

最终会在当前路径生成一个 `package.json` 文件,文件内容有如下代码所示:

```json
{
  "name": "moment",
  "version": "1.0.0",
  "description": "描述信息",
  "main": "index.js",
  "dependencies": {
    "moment": "^1.0.0",
    "axios": "^1.2.6"
  },
  "devDependencies": {},
  "scripts": {
    "test": "node ./index.js"
  },
  "repository": {
    "type": "git",
    "url": "github地址"
  },
  "keywords": ["react", "vue"],
  "author": "作者",
  "license": "ISC"
}
```

通过 `npm init -y` 会生成一个默认的 `package.json` 文件。接下来我们对 `package.json` 文件中部分常见的字段进行详细的讲解。

## name

如果你要发布你的包,`name` 字段是你的 `package.json` 文件中非常重要的字段,在 `npm` 官网搜索到的包名就是该字段的值,如果不打算发布你的包,则 `name` 是可选的,`name` 的 值遵循以下规则:

- 必须小于或等于 214 个字符;
- 不能以 `'.'` 、 `"_"` 、大写字母、中文开头,该字段最终成为 `URL`、命令行上的参数和文件夹名称的一部分。因此,`name` 不能包含任何非 url 安全字符;
- 不能使用 `NOdejs` 核心模块相同的名称;
- 这个名字可能会作为参数传递给 `require()` 函数,所以它应该很短,但也要合理地描述;
- 如果你需要发布包,你需要查看 `npm` 的注册表(`registry`)是否存在同名包,否则你将无法发布成功,当然你可以通过后期修改;

在实际的开发中,很多项目都不需要发布到 `npm` 上,所以这个 `name` 在这里并不是那么重要,不会影响项目正常运作,可以忽略。

## version

和 `name` 一样,如果要发布你的包,这个字段很重要,如果不需要,这个字段则显得不是那么重要了。

`version` 必须可以被 [node-semver](https://github.com/npm/node-semver) 解析,`node-semver` 作为依赖项与 `npm` 捆绑在一起。

`semver` 的版本规范是 `X.Y.Z`:

- `X` 为主版本号(`major`): 通常在涉及重大功能更新,产生了破坏性变更时会更新此版本号;
- `Y` 为次版本号(`Minor`): 在引入了新功能,但并未产生破坏性变更,依然向下兼容时更新此版本号;
- `Z` 为修订号(`Patch`): 在修复了一些问题,但未产生破坏性变更时会更新此版本号;

除了 `X.Y.Z` 这样的版本号,还有 `premajor | preminor | prepatch | prerelease ` 这样的版本号,这些属于测试版本,很多时候一些新改动,并不能直接发布到稳定版本上,这是可以发布一个预发布版本,不会影响到稳定版本。

可以通过以下命令来查看 `npm` 包的版本信息,如想要查看的 react 的:

- 查看最新版本

```sh
npm view react version
```

- 查看所有版本

```sh
npm view react versions
```

我们可以通过 `child_process` 执行该命令以获取某的包的版本,这个方法将在后面的脚手架中用到,具体操作请看以下代码:

```js
const cp = require("child_process");

cp.exec("npm view react version", (err, stdout, stderr) => {
  console.log(stdout); // 18.2.0
});
```

## description

`description` 字段用来描述这个包,它可以是一个字符串,也可以是中文,在搜索的时候可以看到该包的描述,并且该描述也可以作为搜索的关键词。

例如 `react` 中的 `description` 字段的值为 `React is a JavaScript library for building user interfaces.`,通过搜索 `"React is"` 能显示会有以下结果:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41b9524e1b2e4128b2e9a242a88a0659~tplv-k3u1fbpfcp-watermark.image?)

## keywords

`keywords` 字段在里面放关键字,好的关键词可以帮助开发者在 `npm` 官网上更好的搜索到此项目,增加曝光率。例如 `axios` 的 `keywords` 如下:

```json
"keywords": [
  "xhr",
  "http",
  "ajax",
  "promise",
  "node"
],
```

## homepage

项目主页的链接,通常是项目 `github` 链接,项目官网或者文档首页,如果是 `npm` 包,会在这里显示:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/163f302153ce4660909401b274a5cf0f~tplv-k3u1fbpfcp-watermark.image?)

## bugs

`bugs` 字段表示项目提交问题的地址,该字段是一个字符串也可以是一个对象,最常见的 `bugs` 是 `github` 的 issue,例如 `react` 中的 `bugs` 是这样的:

```json
"bugs": "https://github.com/facebook/react/issues"
```

## license

[license](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#license) 字段表示项目的开源许可证,你应该为你的包指定一个许可证,方便其他开发者知道如何允许他们使用它,以及你对他施加的任何限制,例如:

```json
"license": "ISC"
```

## repository

`repository` 字段用于指定代码所在的位置,通常是 `github`,如果是 `npm` 包,则会在包的首页中显示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d44b0a7a50454291849126691224a21f~tplv-k3u1fbpfcp-watermark.image?)

## files

`files` 字段是一个数组,当需要项目进行 `npm` 发布时,可以通过 `files` 字段指定哪些文件需要发布到 `registry` 来控制 `npm` 包大小,如果指定的是文件夹,那么该文件夹下面的所有文件都会被提交,例如 `react` 的配置是这样的,如下代码所示:

```json
"files": [
  "LICENSE",
  "README.md",
  "index.js",
  "cjs/",
  "umd/",
  "jsx-runtime.js",
  "jsx-dev-runtime.js",
  "react.shared-subset.js"
]
```

## main 和 browser

`main` 字段用来指定包的入口入口文件,在上面的内容就忽略不讲了,如果不设置,则默认使用包的根目录中的 `index.js` 文件作为入口。

`main` 字段可以在 `browser` 环境和 `Nodejs` 环境都可以使用,如果 使用 `browser` 字段,则表明只能在浏览器环境下使用,不能用于服务端。

## type

`type` 字段表示在 `Node` 环境中可以使用的模块化方式,默认是 `CommonJs`,另外还可以选择 `EsModule`。

## bin

一些包的作者希望包可以作为命令行工具使用,配置好 `bin` 字段之后,通过 `npm install package_name  -g` 命令可以将脚本添加到执行路径中,之后可以在命令行中直接执行。

`bin` 字段的主要作用可以看 [在终端里输入 npm start 后都发生了啥 😮😮😮](https://juejin.cn/post/7187368174952644665#heading-3) 这篇文章。

## exports

`exports` 字段可以配置不同环境对应的模块入口文件,并且当他存在时,它的优先级最高,当 `package.json` 文件中存在 `exports` 字段,设置的 `main` 字段会失效,在 `react` 中的 `package.json` 文件中的 `exports` 字段有如下配置:

```json
"exports": {
  ".": {
    "react-server": "./react.shared-subset.js",
    "default": "./index.js"
  },
  "./package.json": "./package.json",
  "./jsx-runtime": "./jsx-runtime.js",
  "./jsx-dev-runtime": "./jsx-dev-runtime.js",
  "./src/*": "./src/*"
},
```

这个配置相当于 `weboack` 中的 `resolve.alies` 配置路径别名,请看下面的例子,在 `node_modules` 文件夹下创建一个 `moment` 文件,并且添加 `package.json` 文件让其作为一个 `npm` 模块,通过 `require('moment')` 会查找到当前文件夹,并通过 `main` 字段或者 `exports` 字段查找当前字段的入口,请看如下代码所示:

```json
"exports": {
".":{
  "require":"./test.js"
}
```

在上面的代码中,使用 `require('moment')` 则表示会自动导入 `moment` 包中的 `test.js` 文件,所以在自己的项目中导入会有以下输出:

```js
// moment包中的 test.js
function foo() {
  return "test";
}

module.exports = foo;

const foo = require("moment");

console.log(foo()); // test
```

再在 `package.json` 中的 `exports` 字段中添加这行代码:

```json
"./src": {
  "require": "./src/moment.js"
}
```

当在自己的项目中使用 `require('moment/src')` 进行导入时,实际上导入的是 `moment/src` 目录下的 `moment.js` 文件,默认情况下会 `require('moment/src')` 会加载 `index.js`,因为按照 `require` 规则会先查找 `index.js`,具体示例如下代码所示:

```js
// moment/src/moment.js

function bar(params) {
  return "moment.js";
}

module.exports = bar;

// moment/src/index.js

function bar(params) {
  return "index.js";
}

module.exports = bar;

const bar = require("moment/src");

console.log(bar()); // moment.js
```

在自己的项目中使用 `require('moment/src')` 最终会输出 `moment.js`,如果将 `exports` 字段删除会输出 `index.js`,更多 `require` 细节可以参考 [深入浅出 CommonJs](https://juejin.cn/post/7166039059867893767) 这篇文章。

## config 用于设置 `script` 字段里的脚本运行是的参数,如设置 `prot` 为**3000**:

```json
"config": {
  "port": "8080"
}
```

并且在 `script` 字段下添加 `dev`属性,值设置为 `nodemon index.js`,终端运行 `npm run dev`

```json
"scripts": {
  "dev": "nodemon index.js"
},
```

在执行脚本的时候,我们可以通过 `process.env.npm_package_config_port` 这个变量访问到 `3000`

```js
console.log(process.env.npm_package_config_port); // 3000
```

## dependencies

`dependencies` 字段表示运行依赖,也就是项目中生产环境下需要用到的依赖,例如 `react`、`vue`、`axios` 等 `npm` 包,使用 `npn install <packagename>` 或者 `npm install <packagename> --save` 时,会被自动插入到该字段中:

```json
  "dependencies": {
    "axios": "^1.2.6",
    "react": "^18.2.0",
    "shelljs": "^0.8.5"
  },
```

## devDependencies

`devDependencies` 字段表示开发阶段时需要的依赖包,例如 `webpack`、`eslint`、`sass`,该依赖用于提升开发速度,和 `dependencies` 不同的是,该字段只需要在开发阶段中使用,不需要在生产环境中使用,要使依赖安装到 `devDependencies` 字段上,可以使用 `npm install <packagename> --save-dev` 或者 `npm install <packagename> -D`

## dependencies 扩展

在依赖包中后面的参数就是版本,这些版本信息有以下规则:

- `version`: 表示确定的版本号;
- `>version`: 大于当前版本;
- `~version`: 表示主版本 `X` 和次版本 `Y` 保持不变,修订号 `Z` 永远安装最新的版本;
- `^version`: 表示主版本 `X` 保持不变,次版本 `Y` 和修订号 `Z` 永远安装最新的版本;
- `1.2.x`: 表示该版本中的 `x` 可以是任何符合规范的值;
- `*`和 `""`: 表示匹配任何版本;
- `version1 - version2`: 表示 version<= `x` >= version;
- `version1 || version2`: 表示只要其中一个符合即可;
- `latest`: 代表最新版本;
- `path/path/path`: 提供到包含包的本地目录的路径,本地路径可以使用 `npm install` 保存,该文件会被保存在 `node_modules` 目录下,例如,在根目录创建一个 `bar` 的文件并且在 `package.json` 中添加以下字段:

```json
"dependencies": {
  "bar": "file:./bar"
},
```

在 `bar` 的文件中创建 `index.js` 文件,并编写以下代码:

```js
function foo() {
  return "hi";
}

module.exports = foo;
```

通过执行 `npm install` 会把该文件链接到 `node_modules` 目录文件下,在根目录中编写以下代码:

```js
const bar = require("bar");

console.log(bar());
```

执行命令 `node .\index.js`,你会发现 `hi` 会被正常输出了。

## private

`private` 字段可以防止我们意外地将私有库发布到 `registry`,只需将 `private` 字段设置为 `true` 即可:

```json
"private": true
```

## @ 作用域

已知所有的 `npm` 包都有一个名字,但是随着用户量的增大,`npm` 包的包名就出现了被占用的问题,占用后其他人就不能使用了,当然 `npm` 也官方不是闲着,他们提供了一个作用域的概念,该作用域遵循包名的通常规则,即 `url` 安全字符,在包名使用时,在作用域前面添加一个 `@` 符号,表示这个是一个作用域,在后面加一个斜杠,例如:

```js
@somescope/somepackagename
```

每个 `npm` 用户或者组织都有自己的作用域,它可以是你的用户名、自己创建的组织名,只有你可以在你的作用域中添加包,这就意味着你不必担心别人会抢在你前面取你的包名,类似于你的 `github`,你可以取任何的仓库名称,只需名称合法且在你自己的仓库中不存在同名仓库。

作用域包通过在 `npm install` 中引用它的名称(前面加@符号)来安装:

```sh
npm install @myorg/mypackage
```

在 `package.json` 中有以下的形式:

```json
"dependencies": { "@myorg/mypackage": "^1.3.0" }
```

因为作用域包安装在作用域文件夹中,所以在代码中需要它们时,必须包含作用域的名称:

```js
require("@myorg/mypackage");
```

## script

`script` 字段支持许多内置脚本和它们的预设生命周期事件以及任意脚本,这些你都可以使用 `npm run <stage>` 来执行,例如当你要执行 `npm run dev` 命令,具有匹配名称的 `Pre` 和 `Post` 命令也就作为这些命令一起执行,执行顺序为 `predev` - `dev` - `postdev `,例如在 `package.json` 文件中有如下定义:

```json
"scripts": {
  "dev": "node index.js",
  "predev": "node pre.js",
  "postdev": "node post.js"
},
```

当执行 `npm run dev`,会首先执行 `pre.js` 文件,然后执行 `index.js` 文件,再最后执行 `post.js` 文件。

更多关于 `scirpt` 字段的细节将会在另一篇文章中讲解,敬请期待。

# 参考文章

- 书籍: `深入浅出Nodejs`;
- [npm 官网](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#people-fields-author-contributors)
- [字节跳动技术团队](https://juejin.cn/post/7145001740696289317)

# 结尾

整个系列会比较长,如果想阅读后续的推文,可以点赞+关注 😁😁😁
