在日常的开发中想必大家都已经掌握了使用 `npm install <package>` 命令来安装相关的依赖包了,这里的 `<package>` 通常指的是我们所要安装的包名,默认配置下 `npm` 会从默认的源 `registry` 中查找该包名对应的包地址,下载文件并解压到 `node_modules` 目录下。

细心的小伙伴可能发现了,当你从网上刚拷贝下来的项目通过安装 `npm install`,你会发现安装速度超慢,但是在后面的开发过程中,会发现把 `node_modules` 目录下的文件清空之后再执行 `npm install` 会发现安装速度会比之前快了很多,那么这又是什么原因的。

或许你会发现在 `node_modules` 目录下的文件,并存在其他 `node_modules` 文件,按理来说基本每个依赖包都应该存在 `node_modules` 目录,那么这又是为什么呢?

虽然使用者无需关注这个目录里的文件结构细节,只管在业务代码中引用依赖包即可,但了解 `node_modules` 的内容可以使我们更好理解 `npm` 如何工作,那么在接下来的内容中我们将深入了解一下 `npm install` 是如何工作的。

# npm install 原理

要想在目录在中能正常执行 `npm install`,那么首先在当前目录下必须包含一个 `package.json` 文件,否则执行该命令不会生效,具体如下图所示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/087130964443432ba670b43b2a4bb978~tplv-k3u1fbpfcp-watermark.image?)

在内容开始之前,我们首先通过一张图来大致了解一下执行 `npm install` 之后会经过的几个流程,以方便我们在接下来的内容讲解中清楚地知道每个步骤具体在干什么事情,具体请看下图:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73ff21af3e0345f0bfd845a60454505d~tplv-k3u1fbpfcp-watermark.image?)

## 依赖包之间的嵌套

在开始文章之前,我们再来回顾一下 `package-lock.json` 文件中的内容,其中包含以下字段:

- `version`: 包唯一的版本号;
- `resolved`: 安装源,意思是存放该包的地址,类似于 `Github` 仓库中的地址;
- `integrity`: 包的 `hash` 中值,基于 `Subresource Integrity` 来验证已安装的软件包是否被改动过,是否已失效;
- `requires`: 依赖包所需要的所有依赖项,对应依赖包里的 `pacakge.json` 文件里的 `dependencies` 中的依赖项;
- `dependencies`: 结构和外层的 `dependencies` 结构相同,存储安装在子依赖 `node_modules` 中的依赖包;

在之前的 `npm` 版本中,`npm` 处理依赖的方式简单粗暴,直接用递归的方式,直接按照 `package.json` 文件中的结构以及子依赖包的 `package.json` 文件中的结构将依赖包安装到它们各自的 `node_modules` 中,知道所有子依赖包不存在其他依赖。

举个 🌰🌰🌰,我们在自己的项目 `moment` 中依赖 `axios`,具体 `package.json` 文件中如下定义:

```json
{
  "name": "moment",
  "version": "1.0.0",
  "dependencies": {
    "axios": "^1.3.1"
  }
}
```

在 `axios` 依赖中又依赖了三个依赖包,具体请看下列 `axios` 中的 `package.json` 文件中的配置:

```json
  "dependencies": {
    "follow-redirects": "^1.15.0",
    "form-data": "^4.0.0",
    "proxy-from-env": "^1.1.0"
  },
```

而 `axios` 中依赖的包又有对于其他的依赖,例如 `form-data`,该依赖包的又存在三个依赖,具体配置如下所示:

```json
  "dependencies": {
    "asynckit": "^0.4.0",
    "combined-stream": "^1.0.8",
    "mime-types": "^2.1.12"
  },
```

到这里就不一一列举所有的依赖包了,当执行 `npm install` 命令后,得到的 `node_modules` 中的模块目录结构如下图所示:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a51f1e31dc7a4c0e8a7c6317f1ff0ad0~tplv-k3u1fbpfcp-watermark.image?)

这样的仿方式有点很明显,`node_modules` 的结构和 `package.json` 结构一一对应,层次结构明显,并且保留了每次安装目录都是相同的,但是,当项目的依赖比较多太多的时候,`node_modules` 将会非常庞大,嵌套层级也会非常的深,并且可能会存在重复的依赖包。

举个 🌰🌰🌰,在我们自己的项目 `moment` 中使用了 `axios`,已知 `axios` 会被存放在当前目录下的 `mode_modules` 文件下,而 `axios` 依赖的 `node_modules` 中存在依赖 `follow-redirects`、`form-data`、`proxy-from-env`,而当我们在项目中又想使用 `proxy-from-env` 的时候,安装依赖,当前依赖会被存放在 `moment` 目录下的 `node_modules`,这样子,我们的项目中就存在了两个 `proxy-from-env` 的相同依赖。

## 扁平结构

为了解决以上的问题,`npm` 将早期的嵌套结构改为了扁平结构,那么什么是扁平结构呢,我们先来看一个例子:

```js
const array = [1, 2, 3, [4, [5, [6, [7, [8, [4]]]]]]];
console.log(array); // [ 1, 2, 3, [ 4, [ 5, [Array] ] ] ]

const result = array.flat(Infinity);

console.log(result); // [1, 2, 3, 4, 5, 6, 7, 8, 4];
```

在上面的代码中,创建了一个多维数组,数组中嵌套着数组,在后面的代码中我们通过使用 `Array` 中提供的原型方法将数组扁平至一维数组,最终输出了一个没有任何嵌套的一维数组。

我们再回到正文中,`npm` 采用的扁平结构正是采用这样的思想,当安装模块时,不管是直接依赖还是子依赖,优先将其安装在 `node_modules` 根目录,当我们执行 `npm install` 命令之后,`moment` 项目中的 `node_modules` 目录会生成以下的目录结构:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74ef17a5047f42bb8b43db2338bf433c~tplv-k3u1fbpfcp-watermark.image?)

这就是一个扁平结构,在这个过程中如果发现存在一个版本号一致的依赖,则直接将其丢弃,比如 `axios` 中的依赖 `proxy-from-env` 的版本是 `1.0.0`,而在我们自己的项目中的直接依赖又是 `1.0.0`。

每个 `semver` 都对应一段版本运行的范围,如果两个依赖的版本允许范围存在交集,那么就可以得到一个兼容版本,而不必要版本号一致,这可以使更多冗余模块被清除掉,减少项目的体积。

## 检查缓存

这一步将会检查本地中是否存在缓存,如果是直接将缓存解压到 `node_modules` 目录下,如果不存在则通过网络去下载包。

## npm 缓存

当我们执行 `npm install` 命令之后,从 `registry` 下载到的压缩包都会被存放到本地的缓存目录,那么我们应该怎么查看缓存目录呢?

我们可以使用以下命令查看缓存目录:

```sh
npm config get cache
```

在终端中执行该命令,会有这样的输出,`D:\node\node_cache` 这个路径便是存放 `npm` 缓存的目录,这个是我个人电脑的路径,每个人 `node` 的安装路径都不同,所以结果都不同。

打开目录,发现会存放着以下目录:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/838c6a4a78df48b88c16bd25895244a8~tplv-k3u1fbpfcp-watermark.image?)

其中 `content-v2` 存放的是实际的内容,而 `index-v5` 是存放依赖的索引信息。

首先我们来看一下 `index-v5` 目录下的内容是什么样子的(内容格式化了一下):

```json
86cf999b7f3ea591cf61925a0dbe3c4997b85c5d
{
    "key": "make-fetch-happen:request-cache:https://registry.npmjs.org/@babel/preset-react/-/preset-react-7.0.0.tgz",
    "integrity": "sha512-oayxyPS4Zj+hF6Et11BwuBkmpgT/zMxyuZgFrMeZID6Hdh3dGlk4sHCAhdBCpuCKW2ppBfl2uCCetlrUIJRY3w==",
    "time": 1675299618116,
    "size": 1987,
    "metadata": {
        "time": 1667289376213,
        "url": "https://registry.npmjs.org/@babel/preset-react/-/preset-react-7.0.0.tgz",
        "reqHeaders": {},
        "resHeaders": {
            "cache-control": "public, must-revalidate, max-age=31557600",
            "content-type": "application/octet-stream",
            "date": "Tue, 01 Nov 2022 07:56:15 GMT",
            "etag": "\"7fb248fc0c589b12896e0572085d0b7a\"",
            "last-modified": "Mon, 27 Aug 2018 21:44:38 GMT",
            "vary": "Accept-Encoding"
        },
        "options": {
            "compress": true
        }
    }
}
```

在上面这些字段中,主要来讲解一下下面这些字段:

- `key`: 是一个由 `SHA256` 生成的 `hash` 值,通过该值找到 `content-v2` 中的对应文件;
- `integrity`: 是校验这个文件是否完整用的;
- `time` 时间戳,在这里有两个 `time`,不知道是指代什么的时间;
- `size`: 对应的包大小;
- `url`: 对应依赖包的存放地址;
- `resHeaders`: `HTTP` 请求头;

我们再来看下 `content-v2` 目录下的内容,里面存放的基本都是一些 `二进制` 文件,我们把 `二进制` 文件的扩展名修改为 `.tgz` 再解压之后，会发现就是在我们熟知的依赖包,下图便是我刚才随便解压后生成的文件内容,这正是一个依赖包:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5328d563254e458c956956c39ed2a7dc~tplv-k3u1fbpfcp-watermark.image?)

## 下载包

如果检查到本地缓存中不存在对应的依赖包,便会通过发送网络请求去下载包。

那么怎么下载呢,用什么链接去下载呢?

答案是通过 `package-lock.json` 文件中的 `resolved` 字段,当我们尝试通过该字段中的值从浏览器中输入,发现会直接给我们下载了一个文件,例如,我们使用 `axios` 中的 `resolved` 中的值,具体值是:

```url
https://registry.npmjs.org/axios/-/axios-1.3.1.tgz
```

查看浏览器下载的内容,会发现有以下文件:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8b9025b50444a378e2bab6b4eb54aba~tplv-k3u1fbpfcp-watermark.image?)

通过解压该文件,发现正是一个 `axios` 的依赖包:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/177271f814734afbb23089e8180ef9fc~tplv-k3u1fbpfcp-watermark.image?)

当我们从命令行中执行 `npm install <package>` 的时候,包会经过上面缓存目录下的 `tmp` 临时目录:

![1675397928214.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19d70325ff3543ce816c4526a154ee24~tplv-k3u1fbpfcp-watermark.image?)

从 `registry` 中下载下来的文件会被检查是否完整性,如果检验不通过,则重新下载,否则会将下载下来的依赖包添加到缓存,并最终将包解压到 `node_modules` 目录,最终会生成一个全新的 `package-lock.json` 文件。

如果存在 `package-lock.json` 文件,会检查 `package.json` 文件中的依赖版本是否和 `package-lock.json` 文件中的依赖有冲突,如果没有冲突,直接跳过获取依赖包信息、构建依赖书过程,开始在缓存中查找包信息。

到这里,`npm install` 之后所经历的大概流程在这里也就讲完了。

# npx

什么是 `npx`,在 `npm` 里是这样定义的:

> 从本地或远程 npm 包中运行命令。

当我们安装 `npm` 的时候,`npx` 也会跟着来了,买一送一,你会发现两者的版本一致:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40b414191bc4455da96020a3e007d5c2~tplv-k3u1fbpfcp-watermark.image?)

`npx` 就是一个会帮你执行依赖包里的二进制文件。`npx` 是一个工具,是一个 `npm` 包执行器,当要想使用指定包是可以避免全局安装,使得我们可以随时使用 `cli` 工具或者其他依赖包,大大简化了一些事情。

如果 `package.json` 的文件中的 `bin` 字段只有一个入口,那么执行 `npx` 将会从这个入口开始运行依赖包。

它主要有以下的优点:

- 临时安装可执行依赖包，不用全局安装，不用担心长期的污染;
- 可以执行依赖包中的命令，安装完成自动运行;
- 自动加载 `node_modules` 中依赖包，不用指定 `$PATH`;

## npx 使用

用我自己写的脚手架来说(功能还没开发完,后期会正式开源),当我们要想正常使用该脚手架,我们必须使用全局安装该依赖包:

```sh
npm install fast-create-app -g
```

安装成功之后才能在终端中执行相对应的命令,而有了 `npx` 之后,我们可以直接这样做:

```sh
npx fast-create-app create-app  xun
```

执行该命令之后,你会发现控制台会有以下输出:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fda4e4b30ef4bd0b3f5ac81832013f8~tplv-k3u1fbpfcp-watermark.image?)

也就是说 `npx` 会自动查找当前依赖包中 `bin` 字段中的入口文件,并执行该文件,例如在我自己的配置中的 `bin` 字段有如下配置:

```json
  "bin": {
    "moment": "./index.js"
  },
```

## npx 原理

`moment` 代表的是,当我们全局安装了该依赖包的时候,终端中输入该字段便会开始执行 `./index.js` 路径下的文件,在使用的过程中,该依赖包会被存放在 `D:\node\node_cache\_npx` 目录下,当 `npx` 命令执行完完成之后,包就会自动删除了。

总的来说,`npm` 中的 `m` 是 `Management`,`npx` 中的 `x` 可以理解为 `eXecute`。

当执行 `npx xxx` 的时候,`npx` 先看看 `xxx` 在 `$PATH`里是否存在,例如 `node` 就是存在于 `$PATH` 中的,如果没有,查找当前目录的 `node_modules` 里有没有,如果依然没有,则按照这个 `xxx` 来执行。当命令执行完成之后,会自动删除 `xxx`。

# 参考资料

- [npm 和 npx 有什么区别？](https://www.zhihu.com/question/327989736)
- [npm install 原理分析](https://mp.weixin.qq.com/s?__biz=Mzk0MDMwMzQyOA==&mid=2247490258&idx=1&sn=b293a8deef3b41693e9b547c95f7b135&source=41#wechat_redirect)
