经过半年的幻想,一个多月的准备,十天的开发,我终于开源了自己的脚手架。

# 背景

在我最开始学习 `React` 的时候,使用的脚手架就是 `create-react-app`,我想大部分刚开始学的时候都是使用这个脚手架吧。

使用这个脚手架挺适合新手的,零配置,执行该脚手架命令安装特定的模板,安装相关依赖包,通过执行 `npm start` 即可把项目运行起来。

但是这个脚手架在开发的过程中我要引入相对应的模块,例如要引入一个组件 `import NiuBi from '../../../components/niubi.jsx'`,这个路径看起来就很丑,而且编写的时候极度困难,因此我们可以通过 `Webpack` 配置路径别名,可那时候我哪会配置 `Webpack` 啊,**善于思考**的我决定打开百度,发现可以使用 `carco` 配置 `Webpack`,但是发现 `carco` 版本和 `react-script` 版本并不兼容,因为这个问题把我折磨了一天,因此这个时刻我想自己去开源一个脚手架的想法从此诞生,虽然那时候的我技术非常菜,不过我现在的技术也菜,但是我胆子大啊!!!😏😏😏

所以在这里我总结一下 `create-react-app` 脚手架的一些缺点,但是这仅仅是个人观点:

- 难定制: 如果你需要自定义配置 `Webpack`,你需要额外使用第三方工具 `carco` 或者 `eject` 保留全部 `Webpack` 配置文件;
- 模板单一: 模板少而且简单,这意味着我们每次开发都要从零开始;

那么接下来就来看看我的这个脚手架是怎么使用的。

# 基本使用

## 全局安装

```sh
npm install @obstinate/react-cli -g
```

该脚手架提供的的全局指令为 `crazy`,查看该脚手架帮助,你可以直接使用:

```sh
crazy
```

输入该命令后,输出的是整个脚手架的命令帮助,如下图所示:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3950f1460b9b4ed2a6d1d4402e396667~tplv-k3u1fbpfcp-watermark.image?)

## 创建项目

要想创建项目,你可以执行以下命令来根据你想要的项目:

```sh
crazy create <projectName> [options]
```

例如创建一个名为 `moment`,如果当前终端所在的目录下存在同名文件时直接覆盖,你可以执行以下命令:

```sh
crazy create moment -f
```

如果你不想安装该脚手架,你也可以使用 `npx` 执行,使用 `npx @obstinate/react-cli` 代替 `crazy` 命令,例如,你要创建一个项目,你可以执行以下命令:

```sh
npx @obstinate/react-cli create moment -f
```

如果没有输入 `-f`,则会在后面的交互信息询问是否覆盖当前已存在的文件夹。

之后便有以下交互信息,你可以根据这些交互选择你想要的模板:

![动画.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d094c0b458ca4c38ac4e2939796fe647~tplv-k3u1fbpfcp-watermark.image?)

最终生成的文件如下图所示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f413a5f5950d4c5d8be5e0d9fb87bb01~tplv-k3u1fbpfcp-watermark.image?)

当项目安装完成之后你就可以根据控制台的指示去启动你的项目了。

## 创建文件

通过该脚手架你可以快速创建不同类型的文件,你可以指定创建文件的指定路径,否则则使用默认路径。

要想成创建创建文件,请执行以下指令:

```sh
crazy mkdir <type> [router]
```

其中 `type` 为必选命令,为你要创建的文件类型,现在可供选择的有 `axios、component、page、redux、axios`,`router` 为可选属性,为创建文件的路径。

具体操作请看下列动图:

![动画.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9ace1c86f6c4cdbb8b03c8158e49aa0~tplv-k3u1fbpfcp-watermark.image?)

> 输入不同的类型会有不同的默认路径,并且无需你输入文件的后缀名,会根据你的项目生成相对应的文件后缀名,其中最特别的是创建 redux 文件会自动全局导入 reduxer,无需你自己手动导入,方便了日常的开发效率。

## 灵活配置

与 `create-react-app` 不同的是,该脚手架提供了自定义 `Webpack` 和 `babel` 配置,并通过 `webpack-merge` 对其进行合并,美中不足的是暂时并还没有提供 `env` 环境变量,要区分环境你可以在你通过脚手架下来的项目的 `webpack.config.js` 文件中这样操作:

```js
// 开发环境
const isDevelopment = process.argv.slice(2)[0] === "serve";

module.exports = {
  // ...
};
```

> 最后一个小提示,如果全局安装失败,检查是否权限不够,可以通过管理员身份打开 cmd 即可解决。

这些就是目前仅有的功能,其他的功能正在逐渐开发中......

# 未来(画饼)

- 逐步优化用户体验效果,编写更完美的使用文档;
- 添加对 `vue` 的支持;
- 提供更多代码规范化配置选择,例如 `husky`;
- 提供单元测试;
- 添加 `env` 环境变量配置;
- 增加更多的完美配置,减少用户对项目的额外配置;
- 添加更多的模板,例如后台管理系统;
- 将来会考虑开发一些配套的生态,例如组件库;
- 等等......

# 如何贡献

项目从开发到现在都是我自己一人在开发,但仅凭一己之力会慢慢变得疲惫,其实现在这个版本早在几天前就已经写好了,就单纯不想写文档一直拖到现在。

所以希望能在这里找到一些志同道合的朋友一起把这个脚手架完善,我希望在不久的将来能创造出一个比 `create-react-app` 更好玩且好用的脚手架。

本人的联系方式请查看评论区图片。

# 最后

本人是一个掘金的活跃用户,一天里可能就两三次上 `GitHub`,如果你联系不到我,如果你不想添加我微信好友你可以通过掘金里私信我,掘金私信有通知,如果我不忙,我可能很快就能回复到你。

如果该脚手架有什么问题或者有什么想法可以通过 [Github](https://github.com/xun082/react-cli) 的 [issue](https://github.com/xun082/react-cli/issues) 给我留言。

如果觉得该项目对你有帮助,也欢迎你给个 `star`,让更多的朋友能看到。

如果本篇文章的点赞或者评论较高,后期会考虑出一期文章来讲解如何基于 `pnpm + monorepo + webpack` 开发的脚手架,如果本篇文章对你有帮助,希望你能随时点个赞,让更多的人看到!!!😉😉😉

最后贴上一些地址:

- [GitHub](https://github.com/xun082/react-cli)
- [npm](https://www.npmjs.com/package/@obstinate/react-cli)
- [小破站项目演示](https://www.bilibili.com/video/BV1xj411V7MM/?spm_id_from=333.999.0.0&vd_source=0eadfc102cca35c01fa23332247ec7f0)
