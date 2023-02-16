在前面的几篇文章中讲到了 `npm` 包管理工具,那么今天我们来聊一聊它的老大哥 `pnpm`,在开始之前,我们再来聊聊 `npm`。

# npm 缺点

在 [`你真的了解npm install和npx原理吗`](https://juejin.cn/post/7195815771447885885) 这篇文章中,我们通过 `npm install` 去安装依赖包,会对其依赖包进行扁平化,例如当我们项目中需要依赖 `axios` 时的时候,会将 `axios` 依赖的依赖包一起安装到当前项目中的 `node_modules` 下,这就形成了**幽灵依赖**。

在 `axios` 中的 `package.json` 文件中存在 `proxy-from-env`,而当我们在自己的项目中想使用 `proxy-from-env` 依赖包的时候,你不需要安装其依赖,因为通过 `import` 或者 `require` 的查找规则能查找到 `node_modules` 下的 `proxy-from-env` 依赖包,这就是所谓的**幽灵依赖**。

在我们的项目中使用 `npm` 时,依赖每次被不同的项目使用,都会重复安装一次,这不仅浪费了我们大量的摸鱼时间,而且还占据了大量的内存,这必然造成空间的浪费,虽然我内存还多,不影响 😉😉😉

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01367c36e636432d8a2e661645f955ef~tplv-k3u1fbpfcp-watermark.image?)

为了解决这一痛点,`pnpm` 包管理工具从此诞生,我们先聊聊什么是 `pnpm`。

# 什么是 pnpm

根据官网的介绍 `p` 是单词 `performant` 的缩写,所以 `pnpm` 可以理解为 `performant npm`。

`pnpm` 主要有以下优点:

- 快速: `pnpm` 比其他包管理工具快两倍;
- 高效: `node_modules` 中的文件链接自特定的内容寻址存储库;
- 支持 monorepo: `pnpm` 内置了对存储库中的多个包的支持;
- 严格: `pnpm` 默认创建一个非平铺的 `node_modules`,因此代码不能访问任意包;

# 软连接和硬连接

在维基百科中,软链接是被这样定义的:

> 符号链接也称为软链接,它是一类特殊的文件,其包含有一条以绝对路径或者相对路径的形式执行其他文件或者目录的引用。

一个符号链接文件仅包含有一个文本字符串,其被操作系统解释为一条指向另一个文件或者目录的路径。它是一个独立文件,其存在并不依赖于目标文件。如果删除一个符号链接,它指向的目标文件不受影响,如果目标文件被移动、重命名或者删除,任何指向它的符号链接仍然存在,但是它们将会指向一个不复存在的文件。

在维基百科中,硬链接是被这样定义的:

> 硬链接是计算机文件系统中的多个文件平等地共享同一个文件存储单元。硬链接必须在同一个文件系统中；一般用户权限下的硬链接只能用于文件,不能用于目录,因为其父目录就有歧义了。删除一个文件名字后,还可以用其它名字继续访问该文件。硬链接只能用于同一个文件系统,不能用于不存在的文件。

在上面的定义中看着很难懂,那么下面我们就来用一张图来搞清楚这之间的关系:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68773005068543afaa1095239a3315ca~tplv-k3u1fbpfcp-watermark.image?)

在上面的图中,硬链接的文件上实际上是一个指向实际内存中的指针,通过操作系统的地址寻址方式找到具体的物理位置,所有的硬链接都是指向同一个磁盘块,删除一个指针不会真正删除文件,只有把所有的指针都删除才会真正删除文件。

软连接是另外一种类型的文件,保存的是它指向文件的全路径,访问时会替换成绝对路径,这就类似于我们桌面的快捷方式。

软链接和硬链接用我个人的理解,可以有以下代码所示:

```js
// 硬链接
const hard = {
  nickname: "moment",
  age: 77,
  address: "广州",
};

// 软链接
const soft = hard;
```

这里的 `soft` 变量通过 `hard` 变量最终可以访问到这个对象的实际地址。

# 节约磁盘空间并提升安装速度

当我们使用 `npm` 进行不同的项目开发的时候,依赖都会重复安装一次,而在使用 `pnpm` 时,依赖会被存储在内容可寻址的存储中,所以:

- 如果你用到了某依赖项的不同版本,只会将不同版本间有差异的文件添加到仓库。例如,如果某个包有 100 个文件,而它的新版本只改变了其中 1 个文件。那么  `pnpm update`  时只会向存储中心额外添加 1 个新文件,而不会因为仅仅一个文件的改变复制整新版本包的内容。
- 所有文件都会存储在硬盘上的某一位置,当软件包被被安装时,包里的文件会硬链接到这一位置,而不会占用额外的磁盘空间,这允许你跨项目地共享同一版本的依赖。

# 创建非扁平的 `node_modules`

当我们在项目中使用 `pnpm add axios` 下载依赖包的时候,生成的 `node_modules` 有如下结构:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e143ce99b0845cf901247c6bffcd5cd~tplv-k3u1fbpfcp-watermark.image?)

查看图片你会发现 `axios` 只是一个符号链接,当开始解析依赖的时候,它使用这些依赖的真实位置,所以它不报了符号链接,那么 `axios` 的真实位置在哪呢?

答案是: `node_modules/.pnpm/axios@1.3.3/node_modules/axios`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84ee0ff2425444589ad56cec1237cfba~tplv-k3u1fbpfcp-watermark.image?)

在这里我们也可以知道 `.pnpm/` 文件夹的用途了,它以平铺的形式存储着所有的依赖包.所以每个包都可以在这种模式命名的文件夹中找到:

```js
.pnpm/<name>@<version>/node_modules/<name>
```

这种方式称之为虚拟存储目录,这个平铺的结构避免了 `npm` 之前版本创建的 `node_modules` 引起的长路径问题,又与之后的版本创建的平铺的 `node_modules` 不同的是,它保留了包之间的相互隔离。

# pnpm 特点

`pnpm` 与其他包管理工具不同的是,`pnpm` 能管理 `nodejs` 版本,安装 `LTS` 版本的 `Nodejs`:

```sh
pnpm env use --global lts
```

安装 v16 的 Node.js:

```sh
pnpm env use --global 16
```

安装 Node.js 的预发行版本:

```sh
pnpm env use --global nightly
pnpm env use --global rc
pnpm env use --global 16.0.0-rc.0
pnpm env use --global rc/14
```

安装最新版本的 Node.js:

```sh
pnpm env use --global latest
```

`pnpm` 还内置了对单一存储库的支持,也就是 `monorepo`,你可以创建一个` workspace` 以将多个项目合并到一个仓库中。

一个 `workspace` 的根目录下必须有  `pnpm-workspace.yaml`  文件，有了 `pnpm` 就意味着我们不再需要 `lerna` 来管理多包项目,之所以我现在这么爱 `pnpm`,就是因为我前段时间在写项目的时候在吐槽下载 `lerna` 的时候慢,可能是我网络慢,于是便有了以下这一幕:

![微信图片_20230216180332.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/030363f7139a4814976f42f7d1f1a027~tplv-k3u1fbpfcp-watermark.image?)

![微信图片_20230216180407.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/318761af549e44afaf6f057a9191df30~tplv-k3u1fbpfcp-watermark.image?)

# 参考文献

- [pnpm 中文网](https://pnpm.io/zh)
