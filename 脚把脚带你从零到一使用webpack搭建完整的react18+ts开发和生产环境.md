时间过得真快,一下子就到了兔年了,感觉虎年就在前几天,在这里提前祝大家龙年大吉。

# 开篇
在上一篇文章发布之后,我收获到了很多重要的情报,发现很多工作了一两年的都不太懂 `webpack`,虽然我也不是很懂,就连我们经常使用的 `npm start` 或者 `npm run xxx` 之后都发生了啥都不是很清楚,这主要是在日常的开发中,大部分的时间都是在完成业务的开发,并没有过多的空闲时间去研究其内部原理。

在开发项目中,我们通常使用别人编写好的脚手架即可,简单方便零配置,例如 `create-react-app`,这也导致了很多人并没有机会接触到 `webpack` 配置,更不用说对相关配置进行修改、优化了。

所以本篇文章正是为了解决这一痛点,本文将使用最新的 `webpack5` 从零到一搭建一个完整的 `react-ts` 开发和生产环境,希望能够给想要了解 `webpack` 配置的朋友进行参考,希望能给大家带来不一样的知识点。

# 目录结构
在文章开始之前,我们先了解一下整个项目的目录结构和文件

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c12ccc5434fc4eae9c19b2138d2dc255~tplv-k3u1fbpfcp-watermark.image?)


# 项目初始化
在项目开始之前,我们需要生成 `package.json` 文件,在接下来的命令中,包管理使用的是 `npm`,你也可以使用 `yarn` 或其他工具:
```sh
npm init -y
```
在下面的代码中便是通过以上命令生成的内容:
```json
{
  "name": "learn",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

首先安装 `Webpack` 和 `Webpac-cli`:
```sh
npm install webpack webpack-cli -D
```

在 `webpack.dev.js` 文件中,我们先来测试一下,添加以下简单代码:
```js
// config/webpack.dev.js
module.exports = {
  mode: "development",
};
```

由于目前 `Webpack` 已经提供了很多基本配置,所以我们这里只要先加上 `mode` 就够了。并且我们在 `src` 目录下添加初始文件  `index.js`:
```js
// src/index.js
console.log("hello webpack");
```

接下来我们试着执行打包命令:
```sh
npx webpack --config ./config/webpack.dev.js
```

`-- cofig` 后面紧紧跟着的是路径,执行 `npx webpack` 的时候,会自动查找根目录上是否存在 `webpack.config.js`,如果不存在,则会使用默认使用 `webpack` 基本配置,如果需要指定其他路径则需添加 `--config` 配置。

此刻控制台中应该提示我们打包成功并生成了 `dist/main.js`,虽然还无法在浏览器中看到结果,但是至少目前思路是对的。

# 配置Webpack常量
接下来我们在 `config/utils` 目录下配置一些经常用到的常量:
```js
// config/utils/helper.js
function getEnv() {
  return process.env.NODE_ENV || "development";
}

module.exports = {
  getEnv,
};

// config/utils/constants.js
const path = require("path");
const { getEnv } = require("./helper");

// 构建目录
const DIST_PATH = path.resolve(__dirname, "../../", "dist");
// src目录
const SRC_PATH = path.resolve(__dirname, "../../", "src");
//public 目录
const PUBLIC_PATH = path.resolve(__dirname, "../../", "public");
//根节点目录
const ROOT_PATH = path.resolve(__dirname, "../../");

const NODE_ENV = getEnv();
//是否是生产环境
const IS_PRODUCTION = NODE_ENV === "production";
//是否是开发环境
const IS_DEVELOPMENT = NODE_ENV === "development";

module.exports = {
  DIST_PATH,
  SRC_PATH,
  PUBLIC_PATH,
  ROOT_PATH,
  IS_PRODUCTION,
  IS_DEVELOPMENT,
};
```

# Webpack公共配置
`Webpack` 要求我们每个配置文件以一个模块的方式导出,因此我们通过 `cjs` 的方式导出一个对象:
```js
// webpack.common.js
module.exports = {
    
}
```

## 配置entry
要使 `webpack` 能正常运行,我们需要需要指定整个项目的入口文件,`webpack` 会根据该入口文件生成一个模块依赖图,然后将你项目中所需的每一个模块组合成一个或多个 `chunks`,它们均为静态资源,用于展示你的项目内容。
```js
// webpack.common.js
const { SRC_PATH } = require("./utils/constants");
const path = require("path");

module.exports = {
  entry: {
    index: path.join(SRC_PATH, "index.js"),
  },
};
```
## 配置output
项目的入口有了,我们也需要一个出口文件,用于存放打包输出的文件,已经对该文件进行相关配置:
```js
// webpack.common.js
const {
  SRC_PATH,
  IS_DEVELOPMENT,
  DIST_PATH,
  IS_PRODUCTION,
} = require("./utils/constants");
const path = require("path");

module.exports = {
  // ...
  output: {
    path: IS_PRODUCTION ? DIST_PATH : undefined,
    // 图片、字体资源
    assetModuleFilename: "assets/[hash][ext][query]",
    filename: IS_DEVELOPMENT
      ? "static/js/[name].bundle.js"
      : "static/js/[name].[contenthash:8].bundle.js",
    // 动态导入资源
    chunkFilename: IS_DEVELOPMENT
      ? "static/js/[name].chunk.js"
      : "static/js/[name].[contenthash:8].chunk.js",
    // webpack5内置了 clean-webpack-plugin,只需将 clean 设置为 true
    clean: true,
    pathinfo: false, // 关闭 Webpack 在输出的 bundle 中生成路径信息
  },
};
```

在上面的配置中,`path` 属性为打包输出文件存放的路径,当其为开发环境时,不需要输出文件,所以设置为 `undefined`。

## 配置loader
在开始之前,我们应该了解一下什么是 `loader`,以便于我们在接下来的内容中更好的理解。

> loader 用于对模块的源代码进行转换。loader 可以使你在 `import` 或 "load(加载)" 模块时预处理文件。因此，loader 类似于其他构建工具中“任务(task)”，并提供了处理前端构建步骤的得力方式。loader 可以将文件从不同的语言（如 TypeScript）转换为 JavaScript 或将内联图像转换为 data URL。loader 甚至允许你直接在 JavaScript 模块中 `import` CSS 文件！

### 处理css和scss文件
要想使用相关 `loader`,我们必须先安装对应的 `loader`
```sh
npm install style-loader css-loader postcss-loader sass-loader mini-css-extract-plugin -D
```

在该文章中我们使用的是 `scss` 进行开发,如有需要后续也可以添加 `less` 来进行项目的开发,已知`loader` 的执行顺序是自下而上和自右向左的,`sass-loader` 的主要作用是加载 `Sass/SCSS` 文件并将他们编译为 `CSS`,`sass-loader` 需要预先安装:
```sh
npm install sass -D
```
`postcss-loader`是一款使用插件去转换CSS的工具,他有许多非常好用的插件,例如 `autoprefixer` 能自动补全浏览器前缀,`postcss-pxtorem` 能自动把px代为转换成rem,如果需要额外配置 `postcss`,则需要在根目录额外创建一个 `postcss.config.js` 进行配置:
```js
// postcss.config.js
const stylelint = require("stylelint");

module.exports = {
  plugins: [
    require("autoprefixer"),
    // 添加css代码校验
    stylelint({
      config: {
        rules: {
          "declaration-no-important": true,
        },
      },
    }),
  ],
};
```
要想正常使用则必须安装 `autoprefixer` 和 `stylelint` 两个依赖,后者是检验 `css` 代码是否合格,类似于 `eslint`。

`postcss-loader` 的转换规则按照按照 `.broswerslistrc` 文件或者 `package.json` 文件下的配置,此项目的规则在 `package.json` 下配置:
```json
{
  // ...
  "browserslist": [
    "last 2 version",
    "> 1%",
    "not dead"
  ]
}
```

当 `postcss-loader` 进行浏览器兼容的时候会跟着此配置进行。

最终处理 `css` 资源的配置如下所示:
```js
// webpack.common.js

//...
const MiniCssExtractPlugin = require("mini-css-extract-plugin");

module.exports={
// ...
  module: {
    rules: [
      {
        test: /\.css$|\.scss$/i, // 指定的文件类型
        include: [SRC_PATH], // 匹配 src 目录下的所有文件
        exclude: /node_module/, // 排除 node_module 目录下的文件
        use: [
          // 如果是开发环境使用 style-loader,生产环境则使用 mini-css-extract-plugin
          IS_DEVELOPMENT ? "style-loader" : MiniCssExtractPlugin.loader,
          {
            loader: "css-loader",
            options: {
              sourceMap: !IS_PRODUCTION, // 在开发环境下开启
              modules: {
                mode: "local",
                auto: true,
                exportGlobals: true,
                localIdentName: IS_DEVELOPMENT
                  ? "[path][name]__[local]--[hash:base64:5]"
                  : "[local]--[hash:base64:5]",
                localIdentContext: SRC_PATH,
                exportLocalsConvention: "camelCase",
              },
              // 该选项允许你配置在 css-loader 之前有多少 loader 应用于 @imported 资源与 CSS 模块/ICSS 导入。
              importLoaders: 2,
            },
          },
          "postcss-loader",
          "sass-loader",
        ],
      },
    ],
  },
}
```

### 处理ts文件
现在我们来添加对于 `typescript` 的支持,要想使 `typescript` 则需添加 `tsconfig.json` 作为 typescript的配置文件:
```json
{
  "compilerOptions": {
    "rootDir": "./src",
    "target": "es5", // 指定 ECMAScript 目标版本
    "module": "CommonJS", //指定生成那个模块的代码
    "strict": true, // 开启所有的严格检查配置
    "esModuleInterop": true,
    "outDir": "dist",
    "lib": ["DOM", "DOM.Iterable", "ESNext"], // 指定要包含在编译中的库文件
    "checkJs": true, // 检查js文件
    "allowJs": true, // 允许编译js文件
    "skipLibCheck": true, // 跳过所有声明文件的检查
    "jsx": "react-jsx",
    "noEmit": true, // 禁止编译器生成文件
    "moduleResolution": "node", // 将模块解析模式设置为 node 模式
    "removeComments": true, // 删除编译后的所有的注释

    "noImplicitAny": true, // 在表达式和声明上有隐含的 any类型时报错
    "strictNullChecks": true, // 启用严格的 null 检查
    "noImplicitThis": false, // 当 this 表达式值为 any 类型的时候，生成一个错误

    "baseUrl": "./",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src", "src/typed-css.d.ts"],
  "exclude": ["node_module", "dist"]
}
```

随着 `JavaScript` 的不断发展,越来越多好用的语法也不断出现,例如 `async/await` 等语法极大的提升了代码可读性和开发效率,但是很多浏览器并不能识别这些新语法,需要把这些最新的语法转换为浏览器能够识别的代码,而 `babel` 正是为了解决这个问题的,要想使用 `babel`,则需要先安装相关依赖:
```sh
npm i babel-loader @babel/core @babel/preset-env core-js @babel/preset-typescript @babel/plugin-transform-runtime @babel/plugin-transform-modules-commonjs @babel/plugin-syntax-dynamic-import
```

我们可以在 `webpack.common.js` 文件下添加以下配置:
```js
// webpack.common.js

// ...
module.exports = {
  module: {
    rules: [
      // ...
      {
        test: /\.(tsx?|jsx?)$/,
        include: [SRC_PATH],
        loader: "babel-loader",
        options: {
          cacheDirectory: true,
          cacheCompression: false, // 缓存不压缩
          plugins: [
            IS_DEVELOPMENT && "react-refresh/babel", // 激活 js 的 HMR
          ].filter(Boolean),
        },
        exclude: [/node_modules/, /public/, /(.|_)min\.js$/],
      },
    ],
  },
};
```
要额外配置 `babel` ,我们可以在跟目录添加 `babel.config.js`,也是需要使用 `module.exports` 将模块导出:
```js
module.exports = {
  presets: [
    [
      "@babel/preset-env",
      {
        useBuiltIns: "usage", // 代码中需要那些polyfill,就引用相关的api
        corejs: 3, // 配置使用core-js低版本
      },
    ],
    "@babel/preset-react",
    "@babel/preset-typescript",
  ],
  compact: true,
  comments: true, // 魔法注释,用于分包
  plugins: [
    ["@babel/plugin-transform-runtime"],
    ["@babel/plugin-transform-modules-commonjs"],
    ["@babel/plugin-syntax-dynamic-import"],
  ],
};
```

值得注意的是,`babel` 只能做语法的转换,当 `ts` 类型错误的时候并babel依然能正常使用,使用什么插件进行校验我们在后面会讲到。

### 处理其他资源
对于处理图片,音频,字体之类的文件有如下配置:
```js
// webpack.common.js

// ...
module.exports = {
  module: {
    rules: [
      // ...
      // 处理图片
      {
        test: /\.(jpe?g|png|gif|webp|svg|mp4)$/,
        type: "asset",
        // 文件生成路径
        generator: {
          filename: "./images/[hash:8][ext][query]",
        },
        parser: {
          dataUrlCondition: {
            maxSize: 10 * 1024, // 小于10kb会被压缩成base64
          },
        },
      },
      // 处理字体
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/i,
        type: "asset/resource",
        generator: {
          filename: "./assets/fonts/[hash][ext][query]",
        },
      },
    ],
  },
};
```

## 配置resolve
配置模块如何解析。例如,当在 `ES2015` 中调用 `import 'lodash'`,`resolve` 选项能够对 `webpack` 查找 `'lodash'` 的方式去做修改或者查找模块。

通过配置 `resolve.alias`,创建 `import` 或 `require` 的别名，来确保模块引入变得更简单。例如，一些位于 `src/` 文件夹下的常用模块:
```js
// webpack.common.js

// ...
module.exports = {
  module: {
    //...
    resolve: {
      extensions: [".js", ".json", ".jsx", ".ts", ".css", ".tsx"],
      alias: {
        "@": path.resolve(__dirname, "../src"),
      },
    },
  },
};
```

## 配置全局变量
在开始之前,我们先来了解一下什么是插件?

> 插件是 webpack 的支柱功能。Webpack 自身也是构建于你在 webpack 配置中用到的 相同的插件系统之上！插件目的在于解决 loader 无法实现的其他事。Webpack 提供很多开箱即用的插件。

`DefinePlugin` 允许在 **编译时** 将你代码中的变量替换为其他值或表达式。这在需要根据开发模式与生产模式进行不同的操作时，非常有用。例如，如果想在开发构建中进行日志记录，而不在生产构建中进行，就可以定义一个全局常量去判断是否记录日志。这就是 `DefinePlugin` 的发光之处，设置好它，就可以忘掉开发环境和生产环境的构建规则。

要想配置全局变量,则有以下配置:
```js
// webpack.common.js

// ...
const { DefinePlugin } = require("webpack");

module.exports = {
  module: {
    //...
    plugins: [
      // 定义全局常量
      new DefinePlugin({
        BASE_URL: '"./"',
      }),
    ],
  },
};
```

因为 `webpack` 内置了 `DefinePlugin`,在这里不需要额外安装。

## 配置HTML模板
要想使用指定HTML模板,我们要事先编写好该文件,在 `publish/index.html` 文件下编写以下代码:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" href="<%= BASE_URL %>favicon.ico" />
    <title><%= htmlWebpackPlugin.options.title %></title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```
在上面的代码中使用 `ejs` 语法传入参数。接下来我们需要安装 `html-webpack-plugin` 进行指定模板,其主要作用就是在 `webpack` 构建后生成 `html` 文件,同时把构建好入口js文件引入到生成的html文件中。
使用该插件需要安装相关依赖,并且有以下配置:
```js
// webpack.common.js

// ...
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  module: {
    //...
    plugins: [
      new HtmlWebpackPlugin({
        template: path.join(PUBLIC_PATH, "index.html"), // 指定生成的模板
        filename: "index.html",
        title: "moment", // html中的title
        inject: true, // script标签存放位置
        hash: true,
        minify: IS_DEVELOPMENT
          ? false
          : {
              // https://github.com/terser/html-minifier-terser#options-quick-reference
              removeComments: true, // 删除注释
              collapseWhitespace: true, //是否去除空格
              minifyCSS: true, // 压缩 HTML 中的 css 代码
              minifyJS: true, // 压缩 HTML 中出现的 JS 代码
              caseSensitive: true, // 区分大小写
              removeRedundantAttributes: true, // 当值与默认值匹配时删除属性。
              removeEmptyAttributes: true, // 删除所有只有空白值的属性
              removeStyleLinkTypeAttributes: true, // 从样式和链接标签中删除type="text/css"。其他类型属性值保持不变
              removeScriptTypeAttributes: true, // 从脚本标签中删除type="text/javascript"其他类型属性值保持不变
              useShortDoctype: true, // 将文档类型替换为短(HTML5)文档类型
            },
      }),
    ],
  },
};
```

## 美化控制台输出
为了美化控制台输出,我们可以把一些无关紧要的信息隐藏掉,我们在后面会讲到,使用 `webpackbar` 可以给控制台添加进度条以监控项目的打包进度以及项目启动进度,需要安装依赖:
```sh
npm install webpackbar -D
```

`webpackbar` 的配置不难,添加简简单单的配置即可:
```js
// webpack.common.js

// ...
const WebpackBar = require("webpackbar");

module.exports = {
  module: {
    //...
    plugins: [
      // 进度条
      new WebpackBar({
        color: "green",
        basic: false,
        profile: false,
      }),
    ],
  },
};
```

但是这个插件唯一的缺点是该进度会永远停留在 `99%`,强迫症疯狂吐槽:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69f8042da3ff43f9a2b681078d098a0b~tplv-k3u1fbpfcp-watermark.image?)

## stats
`stats` 选项让你更精确地控制 bundle 信息该怎么显示。 如果你不希望使用 `quiet` 或 `noInfo` 这样的不显示信息，而是又不想得到全部的信息，只是想要获取某部分 bundle 的信息，使用 stats 选项是比较好的折衷方式。

`stats` 只需要一个简单的配置即可:
```js
// webpack.common.js

// ...

module.exports = {
  // ...
  stats: "errors-only",
};
```

公共部分的配置在这里就结束了,接下来就是讲该配置和开发环境进行结合。

# 开发环境配置
在开始配置之前,我们需要先使用 `webpack-merge` 将 `webpack.common.js` 文件和 `webpack.dev.js` 文件进行合并,形成一个开发环境的配置。

为什么是 `webpack-merge`?

- development(开发环境) 和 production(生产环境) 这两个环境下的构建目标存在着巨大差异 所以webpack的配置写的差距会非常的大;
- 虽然，以上我们将 生产环境 和 开发环境 做了细微区分，但是，请注意，我们还是会遵循不重复写配置的原则，保留一个 `common( 公共 )"`配置。为了将这些配置合并在一起，我们将使用一个名为 webpack-merge 的工具。此工具会引用 `common”` 置,此我们不必再在环境特定env的配置中编写重复代码。

安装:
```sh
npm install --save-dev cross-env
```

进行合并,并且将 `mode` 设置为 `development`模式:
```js
// config/webpack.dev.js

const { merge } = require("webpack-merge");
const webpackCommonConfig = require("./webpack.common");

const devWebpackConfig = merge(webpackCommonConfig, {
  mode: "development",
  devtool: "source-map",
});
```

## 配置 devServer
`devServer` 在这篇文章中已经讲过,详情可看这篇文章 [DevServer与模块热替换之间的爱恨情仇💘以及HMR原理](https://juejin.cn/post/7188086151075332153)

`devServer` 有如下配置:
```js
// config/webpack.dev.js

// ...
const devWebpackConfig = merge(webpackCommonConfig, {
  // ...
  devServer: {
    open: true, // 自动打开浏览器,不指定浏览器就打开默认浏览器
    host: "localhost",
    hot: true, // 开发模块热替换可以可以
    compress: true, // 是否启用 gzip 压缩
    historyApiFallback: true, // 解决前端路由刷新404现象
    client: {
      logging: "info",
      overlay: false, // 关闭报错时覆盖整个屏幕
    },
  },
});
```

细心的你会发现是不是少了个 `port`,如果指定了某个端口,如果该端口被占用,则无法启动成功,那么我们可以通过 `portfinder` 来寻找端口,如果当前端口存在则加一,首先安装依赖:
```sh
npm i portfinder -D
```

使用方法有如下配置:
```js
const FriendlyErrorsWebpackPlugin = require("@nuxt/friendly-errors-webpack-plugin");

module.exports = new Promise((resolve, reject) => {
  portFinder.getPort(
    {
      port: 3000, // 默认端口为3000,如果占用了则加一,以此类推
      stopPort: 30000,
    },
    (error, port) => {
      if (error) reject(error);
      devWebpackConfig.devServer.port = port;
      devWebpackConfig.plugins.push(
        new FriendlyErrorsWebpackPlugin({
          compilationSuccessInfo: {
            messages: [
              `You application is running here http://localhost:${port}`,
            ],
            notes: [
              "Some additional notes to be displayed upon successful compilation",
            ],
          },
        }),
      );
      resolve(devWebpackConfig);
    },
  );
});
```

在上面的配置中使用 `FriendlyErrorsWebpackPlugin` 来优化控制台输出,有以下效果:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc01ee46bfd94e90878002b1e66319af~tplv-k3u1fbpfcp-watermark.image?)

`devServer` 我们已经配置完成了,接下来可以配置 `package.json` 文件用来启动项目:
```json
{
  // ...
  "scripts": {
    "start": "npm run dev",
    "dev": "cross-env NODE_ENV=development webpack serve --config ./config/webpack.dev.js"
  }
}
```

当你使用NODE_ENV =development来设置环境变量时,大多数Windows命令提示将会阻塞或者报错,借助 `cross-env` 使得您可以使用单个命令,而不必担心为平台正确设置或使用环境变量。只要在POSIX系统上运行就可以设置好,而cross-env将会正确地设置它。

要想使用需要安装依赖并且在 `script` 命令的最前门添加 `cross-env`即可,这个时候,如果你的配置没有错的话,通过终端输入 `npm start` 应该能正常启动起来并且自动打开浏览器了。

## 配置plugins
通过 `@pmmmwh/react-refresh-webpack-plugin` 开启模块热替换,使用 `fork-ts-checker-webpack-plugin` 解决 `babel-loader` 无法检查ts类型错误问题,使用 `circular-dependency-plugin` 解决模块循环依赖问题:
```js
const { merge } = require("webpack-merge");
const ReactRefreshWebpackPlugin = require("@pmmmwh/react-refresh-webpack-plugin");
const webpackCommonConfig = require("./webpack.common");
const CircularDependencyPlugin = require("circular-dependency-plugin");
const ForkTsCheckerWebpackPlugin = require("fork-ts-checker-webpack-plugin");

const devWebpackConfig = merge(webpackCommonConfig, {
  // ...
  plugins: [
    new ReactRefreshWebpackPlugin(),
    // 解决babel-loader无法检查ts类型错误问题
    new ForkTsCheckerWebpackPlugin({
      async: false,
    }),
    //  解决模块循环引入问题
    new CircularDependencyPlugin({
      exclude: /node_modules/,
      include: /src/,
      failOnError: true,
      allowAsyncCycles: false,
      cwd: process.cwd(),
    }),
  ],
});
```

使用 `eslint` 检查代码:
```js
const { merge } = require("webpack-merge");

const webpackCommonConfig = require("./webpack.common");
const ESLintPlugin = require("eslint-webpack-plugin");

const devWebpackConfig = merge(webpackCommonConfig, {
  // ...
  plugins: [
    new ESLintPlugin({
      extensions: [".js", ".jsx", ".ts", ".tsx"],
      context: path.resolve(__dirname, "../src"),
      exclude: "node_modules",
      cache: true, // 开启缓存
      // 缓存目录
      cacheLocation: path.resolve(
        __dirname,
        "../node_modules/.cache/.eslintCache"
      ),
    }),
  ],
});
```

通过根目录创建 `.eslintrc.json` 编写 `eslint` 规则,有以下配置:
```json
{
  "plugins": ["react"],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 8,
    "sourceType": "module",
    "ecmaFeatures": {
      "jsx": true,
      "experimentalObjectRestSpread": true
    }
  },
  "env": {
    "es6": true,
    "browser": true,
    "node": true,
    "mocha": true
  },
  "extends": ["eslint:recommended", "plugin:react/recommended"],
  "rules": {
    "react/prop-types": 0,
    "no-unused-vars": 1,
    "no-multiple-empty-lines": ["error", { "max": 1 }],
    "no-var": 2, // 禁止使用 var 声明变量
    "prefer-rest-params": 2, // 要求使用剩余参数而不是 arguments
    "eqeqeq": 2, // 强制使用 === 和 !==
    "no-multi-spaces": 1, // 禁止使用多个空格
    "default-case": 1, // 要求 switch 语句中有 default 分支
    "no-dupe-args": 2 // 禁止 function 定义中出现重名参数
  },
  "settings": {
    "react": {
      "version": "detect"
    }
  }
}
```

具体配置你们看着玩就好了,我也是随便玩玩的,继续添加 `.eslintignore` 来忽略哪些文件不需要代码校验:
```test
*.html
node_modules
coverage
dist
env
postcss.config.js
config
babel.config.js
```

既然使用了 `eslint`,那么就肯定要使用 `prittier` 来对代码进行格式化,在根目录创建文件 `.prittierrc`(需要安装该插件),并别写以下规则,随便玩玩,详情还请观看官网:
```json
{
  "printWidth": 80,
  "tabWidth": 2,
  "trailingComma": "all",
  "semi": true,
  "endOfLine": "auto",
  "arrowParens": "avoid",
  "bracketSpacing": true
}
```

到这来,开发环境的配置已经完成,接下来就是生产环境的了。

# 生产环境配置
和前面的一样,使用 `webpack-merge` 将 `webpack.common.js` 和 `webpack.prod.js` 进行合并,并将 `mode` 设置为 `production`:
```js
const webpackCommonConfig = require("./webpack.common");
const { merge } = require("webpack-merge");

module.exports = {
  mode: "production",
};
```

修改 `package.json` 配置文件:
```json
{
// ...
  "scripts": {
    "start": "npm run dev",
    "dev": "cross-env NODE_ENV=development webpack serve --config ./config/webpack.dev.js",
    "build": "cross-env NODE_ENV=production webpack --config ./config/webpack.prod.js"
  },
}
```

## 配置plugins
添加 `mini-css-extract-plugin` 对 `css` 资源进行压缩,并指定输出文件位置和文件名:
```js
// ...
const MiniCssExtractPlugin = require("mini-css-extract-plugin");

module.exports = {
  // ...
  plugins: [
    new MiniCssExtractPlugin({
      filename: "static/css/[name].[contenthash].css",
      chunkFilename: "static/css/[name].[contenthash].css",
      ignoreOrder: true,
    }),
  ],
};
```

`copy-webpack-plugin` 插件的作用是将项目中的某单个文件或整个文件夹在打包的时候复制一份到打包后的文件夹中:
```js
// ...
const copyWebpackPlugin = require("copy-webpack-plugin");
const { PUBLIC_PATH } = require("./util/constants");

module.exports = {
  // ...
  plugins: [
    new copyWebpackPlugin({
      patterns: [
        {
          from: PUBLIC_PATH,
          to: "./",
          // 选择要忽略的文件
          globOptions: {
            dot: true,
            gitignore: true,
            ignore: ["**/index.html*"],
          },
        },
      ],
    }),
  ],
};
```

`webpack-manifest-plugin` 生成一个manifest.json 默认文件名。也是一个文件清单， 内容是打包前文件对应打包后的文件名:
```js
// ...
const { WebpackManifestPlugin } = require("webpack-manifest-plugin");

module.exports = {
  // ...
  plugins: [
    // 生成目录文件
    new WebpackManifestPlugin({
      fileName: "asset-manifest.json",
      generate: (seed, files, entryPoints) => {
        const manifestFiles = files.reduce((manifest, file) => {
          if (manifest) manifest[file.name] = file.path;
          return manifest;
        }, seed);
        const entrypointFiles = entryPoints.index.filter(
          (fileName) => !fileName.endsWith(".map")
        );

        return {
          files: manifestFiles,
          entryPoints: entrypointFiles,
        };
      },
    }),
  ],
};
```

打包后会生成如下信息:
```json
{
  "files": {
    "index.css": "auto/static/css/index.590b8ab7ce2383862a0b.css",
    "index.js": "auto/static/js/index.4d25eebc.bundle.js",
    "runtime.js": "auto/static/js/runtime.894f45d1.bundle.js",
    "static/css/src_pages_home_index_tsx.css": "auto/static/css/src_pages_home_index_tsx.31d6cfe0d16ae931b73c.css",
    "static/js/src_pages_home_index_tsx.80b5c512.chunk.js": "auto/static/js/src_pages_home_index_tsx.80b5c512.chunk.js",
    "static/js/src_pages_title_index_tsx.9d34e261.chunk.js": "auto/static/js/src_pages_title_index_tsx.9d34e261.chunk.js",
    "vendors.js": "auto/static/js/vendors_vendors.js",
    "favicon.ico": "auto/favicon.ico",
    "index.html": "auto/index.html"
  },
  "entryPoints": [
    "static/js/runtime.894f45d1.bundle.js",
    "static/js/vendors_vendors.js",
    "static/css/index.590b8ab7ce2383862a0b.css",
    "static/js/index.4d25eebc.bundle.js"
  ]
}
```

`webpack-bundle-analyzer` 是一个打包文件分析工具,主要作用是可以直观分析打包出的文件包含哪些,大小占比如何,压缩后的大小等等:
```js
// ...
const BundleAnalyzerPlugin =require("webpack-bundle-analyzer").BundleAnalyzerPlugin;
module.exports = {
  // ...
  plugins: [
    // 打包体积分析
    new BundleAnalyzerPlugin(),
  ],
};
```

打包后会自动在浏览器开启一个端口,主要效果有如下所示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/914dc0ef3b0047f4a855c30772588c47~tplv-k3u1fbpfcp-watermark.image?)


`compression-webpack-plugin` 插件能够通过压缩算法,将前端打包好的资源文件进一步压缩,生成指定的、体积更小的压缩文件,让浏览器能够更快的加载资源:
```js
// ...
const CompressionWebpackPlugin = require("compression-webpack-plugin");
const { gzip } = require("zlib");

module.exports = {
  // ...
  plugins: [
    new CompressionWebpackPlugin({
      filename: "[path][base].gz",
      algorithm: "gzip",
      test: /\.js$|\.json$|\.css/,
      threshold: 10240, // 只有大小大于该值的资源会被处理
      minRatio: 0.8, // 只有压缩率小于这个值的资源才会被处理
      algorithm: gzip,
    }),
  ],
};
```

## optimization
`optimization` 的中文意思是优化,其中 `splitChunks` 是用于分包,`minimizer` 可以添加一系列插件用于优化 `HTML`、`CSS`、`JavaScript`等资源,其中 `terser-webpack-plugin` 使用 `terser` 来压缩你的 `JavaScript` 代码,你可以清除开发环境下的 `console.log`,完整配置代码有如下代码所示:
```js
// ...
const { PUBLIC_PATH, SRC_PATH } = require("./util/constants");
const TerserPlugin = require("terser-webpack-plugin");
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin");
const HtmlMinimizerPlugin = require("html-minimizer-webpack-plugin");

module.exports = {
  // ...
  optimization: {
    chunkIds: "named",
    moduleIds: "deterministic", //单独模块id，模块内容变化再更新
    minimize: true,
    usedExports: true,
    minimizer: [
      new TerserPlugin({
        test: /\.(tsx?|jsx?)$/,
        include: [SRC_PATH],
        exclude: /node_module/,
        parallel: true,
        terserOptions: {
          toplevel: true, // 最高级别，删除无用代码
          ie8: true,
          safari10: true,
          compress: {
            arguments: false,
            dead_code: true,
            pure_funcs: ["console.log"], // 删除console.log
          },
        },
      }),
      // 优化和缩小 html 和 css
      new CssMinimizerPlugin(),
      new HtmlMinimizerPlugin(),
    ],
    // 分包
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        vendor: {
          name: "vendors",
          enforce: true, // ignore splitChunks.minSize, splitChunks.minChunks, splitChunks.maxAsyncRequests and splitChunks.maxInitialRequests
          test: /[\\/]node_modules[\\/]/,
          filename: "static/js/[id]_vendors.js",
          priority: 10,
        },
        react: {
          test(module) {
            return (
              module.resource && module.resource.includes("node_modules/react")
            );
          },
          chunks: "initial",
          filename: "react.[contenthash].js",
          priority: 1,
          maxInitialRequests: 2,
          minChunks: 1,
        },
      },
    },
    runtimeChunk: {
      name: "runtime",
    },
  },
};
```

## CDN
配置cdn代码有如下所示,但是cdn可能会不稳定,谨慎使用:
```js
// ...

module.exports = {
  // ...
  externals: {
    react: "react",
    redux: "redux",
    "react-dom": "ReactDOM",
    "react-router-dom": "ReactRouterDOM",
    axios: "axios",
  },
};
```

配置完成了 `externals`,还需要在模板中引入相关的 `cdn` 链接:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="icon" href="<%= BASE_URL %>favicon.ico" />
    <title><%= htmlWebpackPlugin.options.title %></title>
    <% if (process.env.NODE_ENV === 'production') { %>
    <script src="https://cdn.bootcdn.net/ajax/libs/react/18.2.0/cjs/react-jsx-dev-runtime.development.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/axios/1.2.5/axios.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/redux/4.2.0/redux.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/react-dom/18.2.0/cjs/react-dom-server-legacy.browser.development.js"></script>
    <script src="https://cdn.bootcdn.net/ajax/libs/react-router-dom/6.8.0/react-router-dom.development.js"></script>
    <% } %>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
```

在这里使用的 `ejs` 语法,通过判断是否是生产环境,如果在开发环境使用 `cdn` 链接会比本地资源慢得多。

# 一些额外的配置
我们在开发的时候遇到了通过模块的方式导入 `scss` 文件的时候:
```js
import styles from "./style.module.scss";
```
会有以下报错 **找不到模块“./style.module.scss”或其相应的类型声明。**

在这里,我们可以在 `src` 目录下添加 `type-css.d.ts` 文件,并添加以下配置:
```ts
declare module "*.module.css" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.module.sass" {
  const classes: { readonly [key: string]: string };
  export default classes;
}

declare module "*.module.scss" {
  const classes: { readonly [key: string]: string };
  export default classes;
}
```

到这里,本篇文章的讲解也就结束了,希望能对你有所帮助。

# 结尾
在本篇文章中,仅仅是列了整个模板的开发流程,其中的代码并并不能直接运行,如果想看到完整源码,可以访问我的 [github](https://github.com/xun082/react-template) 地址进行学习。

在接下来的时间里会继续添加对 `less` 的支持。

**在下一篇文章中将会讲解如何从零到一开发一个脚手架,功能庞大,敬请期待。**

最后祝大家在新的一年里都能身体健康,工作顺利,学业有成。