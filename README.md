#SplitChunksPlugin（翻译初版）

chunks（以及模块依赖）最初是通过webpack内部的父子关系进行的关联。`CommonsChunkPlugin`插件被用来避免它们之间的重复依赖，但是这样很难进行更深一步的优化。

从webpack4开始，`CommonsChunkPlugin`被移除，新增了`optimization.splitChunks`和`optimization.runtimeChunk`选项，接下来看一下新流程的工作原理。

##Defaults

开袋即食的`SplitChunksPlugin`应该对大多数用户来说都更友好。

默认情况下，它只会影响按需加载的chunks，而不会影响到初始chunks，因为更改初始chunks将会影响到HTML文件项目运行时需要引入的的script内容。

webpack将会按照以下原则自动分离公共chunks：

* 新chunk可以共享/依赖的模块来自`node_modules`
* 新chunk大于30kb（在 min+gz 压缩之前）
* 按需加载的chunks最大并行请求数<=5
* 页面初始加载的chunks最大并行请求数<=3

当你在尝试满足最后两个条件时，应该优先体积较大的chunks。

来看一些例子。

### Defaults:Example 1

``` js
// index.js

// dynamically import a.js
import("./a");
```

``` js
// a.js
import "react";

// ...
```
**结果：**一个包含`react`的独立chunk将被分离创建。在这个chunk被原chunk（index.js）引入时，两个chunk并行加载。

为什么：

* 条件1：这个chunk的依赖模块来自`node_modules`
* 条件2：`react`大于30kb
* 条件3：按需加载的并行请求数为2
* 条件4：不影响初始页面加载

更深一层的原因是，`react`可能并不会像你的其他代码那样频繁改动。通过把它打包进一个单独的chunk，就可以脱离你的应用代码而进行单独缓存（如果你有使用chunkhash, records, Cache-Control或者其他的长缓存方式）。

### Defaults:Example 2

``` js
// entry.js

// dynamically import a.js and b.js
import("./a");
import("./b");
```

``` js
// a.js
import "./helpers"; // helpers is 40kb in size

// ...
```

``` js
// b.js
import "./helpers";
import "./more-helpers"; // more-helpers is also 40kb in size

// ...
```
**结果：**包含`./helpers`以及它所有依赖的独立trunk将被创建。在这个trunk被原trunk（entry.js）引入时，两个trunk并行加载。

为什么：

* 条件1：这个trunk被两次引入
* 条件2：`helpers`大于30kb
* 条件3：按需加载的并行请求数为2
* 条件4：不影响初始页面加载

把`helpers`在两个trunk中分别打包会导致重复下载。通过把它单独打包只会下载一次。经过权衡考量，我们增加了一次额外的网络请求。这也是为什么默认会有30kb的大小限制。

## Configuration

由于开发者希望有更多的默认功能以上的管理项，webpack提供了一些配置项来满足你的需求。

如果你手动更改默认的配置项，请确保你知道它造成的改变和影响，并且确认所做的配置是在做**优化**。

```
除非你的项目的最佳配置置于其后，默认的配置项将会是符合web性能的最佳策略。
```

### Configuring cache groups

默认把所有来自`node_modules`的模块（命名为`vendors`）分配到一个缓存组内，所有至少被2个trunks加载的模块分配到`default`缓存组。

一个模块可以被分配到多个缓存组。最佳的缓存策略依赖于配置更高优先级的`priority`（`priority`选项）或者体积比较大的trunks。

### Conditions

在满足所有条件的情况下，来自相同trunks和缓存组的模块将会打包一个新的trunk。

一共有4个配置条件：

* `minSize`（默认：300000）一个trunk的最小体积
* `minChunks` （默认：1）分离之前引入模块的最少trunks数
* `maxInitialRequests` （默认：3）一个入口的最大并行请求数
* `maxAsyncRequists` （默认：5）按需加载的最大并行请求数

### Naming

使用`name`选项可以重命名分离出来的trunk的名字。

```
如果给分离出来的多个chunks设置了相同的名字，所有的vendor模块将会被置于一个单独的trunk中。
我们并不建议这样做，因为这样会导致下载更多的代码。
```
`true`属性会自动根据chunks和缓存组的关键字来自动命名，否则可以传一个字符串或者一个函数。

如果命名和一个入口文件的命名冲突，入口文件将被移除。

#### `optimization.splitChunks.automaticNameDelimiter`

webpack默认使用原名称+chunk名称来命名，类似`vendors~main.js`。

如果你的项目规范不允许使用`~`字符，可以设置成其他替代字符，比如：`automaticNameDelimiter: "-"`。

然后后最终的命名就会变成`vendors-main.js`。

### Select chunks

配置指定的chunks可以使用`chunks`来做选择。

有3个可选项：`initial`，`async`和`all`。3个选项分别对应选择初始trunks，异步chunks，或者全部trunks。

`reuseExistingChunk`配置项允许使用现有的chunks，而不是在模块完全匹配时创建新的模块。

这个选项可以配置每一个缓存组。

#### `optimization.splitChunks.chunks: all`
我们之前提到，这个配置项会影响到所有动态加载的模块。如果把`optimization.splitChunks.chunks`设置为`"all"`，初始trunks也将会收到影响（即使某项依赖并没有被动态加载）。这个配置项甚至可以让trunks在入口文件和按需加载模块中共享。

这是一个推荐的配置。

T> 你可以结合[HtmlWebpackPlugin](https://webpack.docschina.org/plugins/html-webpack-plugin/)来使用，它将注入生成的所有vendor chunks。

#### `optimization.splitChunks`

这个配置项用来定义`SplitChunksPlugin`的默认表现。

```js
splitChunks: {  
	chunks: "async",
	minSize: 30000,   
	minChunks: 1,   
	maxAsyncRequests: 5,   
	maxInitialRequests: 3,
	automaticNameDelimiter: '~',
	name: true,
	cacheGroups: {
		vendors: {
			test: /[\\/]node_modules[\\/]/,
			priority: -10
		},
    default: {
			minChunks: 2,
			priority: -20,
			reuseExistingChunk: true
		}
	}
}
```
默认的缓存组继承`splitChunks.*`，但是`test`，`priorty`和`reuseExistingChunk`只能在缓存组的level中配置。

`cacheGroups`是以缓存组命名为键名的对象。上面列出的所有项都是可以被配置的，如：`chunks`，`minSize`，`minChunks`，`maxAsyncRequests`，`maxInitialRequests`，`name`。

你可以把`optimization.splitChunks.cacheGroups.default`设置为`false`来禁用默认的缓存组，`vendors`也一样。

默认的缓存组优先级设置为负数是为了让默认的缓存组有更高的优先级（默认为0）.

下面是一些例子：

### Split Chunks:Example 1

创建一个`commons`chunk，其中包含了所有入口文件的公共代码。

```js
splitChunks: {
    cacheGroups: {
        commons: {
            name: "commons",
            chunks: "initial",
            minChunks: 2
        }
    }
}
```

W> 这项配置会使你的初始文件体积增大，如果一个模块不是即时需要，建议按需加载。

### Split Chunks:Example 2

创建一个`vendors`chunk，其中包含了所有来自`node_modules`的依赖。

```js
splitChunks: {
    cacheGroups: {
        commons: {
            test: /[\\/]node_modules[\\/]/,
            name: "vendors",
            chunks: "all"
        }
    }
}
```
W> 这项配置可能会使包含外部依赖的chunk体积增大，建议只包含框架和公共依赖，并在需要的时候动态加载其余依赖。

#### `optimization.runtimeChunk`

把`optimization.runtimeChunk`设置为`true`会在运行时为每个入口文件增加一个trunk。

`single`会生成一个trunk，以便生成的其他trunks公用。
