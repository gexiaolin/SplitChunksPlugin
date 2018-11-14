# SplitChunksPlugin文档翻译

chunks以及它的内部引入模块本质上是通过webpack对于模块之间的父子关系做的关联。在webpack4以前，我们使用`CommonsChunkPlugin`插件来避免依赖的重复打包，但是这样很难进行更深一步的优化。

从webpack4开始，`CommonsChunkPlugin`被移除，新增了`optimization.splitChunks`和`optimization.runtimeChunk`选项。

## Defaults

开袋即食的`SplitChunksPlugin`对大多数用户来说表现都更好。

默认情况下，它只会影响按需加载的chunks，而不会影响到初始chunks，因为更改初始chunks将会影响到HTML文件项目运行时需要引入的的script项。

webpack将会按照以下原则自动切割chunks：

* 新chunk可被共享，或者是从`node_modules`引入的模块
* 新chunk大于30kb（在 min+gz 压缩之前）
* 对于按需加载的模块，数量不能超出5个
* 对于入口模块，数量不能超出3个

当你在尝试满足最后两个条件时，应该优先体积较大的chunks。

## Configuration

webpack提供一些可配置项来满足开发者的更多功能性要求。

> 默认的配置项在我们看来是适合浏览器表现的最好方式，但是对你的项目来说可能会有所差异。如果你尝试覆盖默认项，请确认你的这些配置对于项目来说真的有优化作用。

### `optimization.splitChunks`

以下是`SplitChunksPlugin`的默认配置：

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async',
      minSize: 30000,
      maxSize: 0,
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
  }
};
```

### `splitChunks.automaticNameDelimiter`

`string`

webpack默认使用origin ～ chunkName的形式生成name（例如：`vendors~main.js`）。该选项可以自定义连接符。

### `splitChunks.chunks`

`function (chunk) | string`

该选项决定优化哪些chunks。如果值为string，请确定为以下三种：`all`、`async`和`initial`。设置为`all`表现会更强大，因为这意味着公共chunk可能同时被同步和异步引入。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // 包含所有chunks
      chunks: 'all'
    }
  }
};
```

你也可以通过一个function来更精确的进行chunks控制。返回值就是需要优化的chunks。

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks (chunk) {
        // 返回所有不是`my-excluded-chunk`的chunks
        return chunk.name !== 'my-excluded-chunk';
      }
    }
  }
};
```

> 你可以配合[HtmlWebpackPlugin](https://webpack.js.org/plugins/html-webpack-plugin/)，使生成的html页面注入公共chunks。

### `splitChunks.maxAsyncRequest`

`number`

按需载入的最大请求数。

### `splitChunks.maxIntialRequests`

`number`

单个入口文件的最大请求数。

### `splitChunks.minChunks`

`number`

代码切割之前的最小共用数量。

### `splitChunks.minSize`

`number`

生成的公共模块的最小size，单位是bytes。

### `splitChunks.maxSize`

`number`

`maxSize`项（无论是全局的`optimization.splitChunks.maxSize`还是每个缓存组内的`optimization.splitChunks.cacheGroups[x].maxSize`，或者缓存组回调`optimization.splitChunks.fallbackCacheGroup.maxSize`）会尝试对大于该值的chunks分割为更小的几部分。每部分需要大于`minSize`的值。该算法是固定的，并且模块的改变仅会造成局部影响。所以在使用长缓存组并且不需要记录时，可以使用该选项。如果切割结果不符合`minSize`要求，`maxSize`不会生效。

如果该chunk已经有了一个name，分割出的每个chunk都会有一个新名字。`optimization.splitChunks.hidePathInfo`项会使分割项根据原始chunk的name派生出一个hash值。

`maxSize`项意在HTTP/2和长缓存中通过增加网络请求数来达到更好的缓存效果。它也可以通过减少文件体积来加快构建速度。

> `maxSize`比`maxInitialRequest/maxAsyncRequests`有更高的优先级。优先级顺序为`maxInitialRequest/maxAsyncRequests < maxSize < minSize`。
 
### `splitChunks.name`

`boolean: true | function (module) | string`

切割出模块的name。设置为`true`会根据chunks和缓存组的key来自动命名。设置为string或者function来确定你想要的命名。如果name和入口文件name相同，入口文件将被移除。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      name (module) {
        // generate a chunk name...
        return; //...
      }
    }
  }
};
```

> 如果给不同的spilt chunk分配相同的name，所有的依赖项都将被打包进同一个公共chunk。如果可以分为不同的模块，我们不建议前者的做法。

### `splitChunks.cacheGroups`

缓存组的配置项会继承`splitChunks.*`的配置，但是`test`，`priority`和`reuseExistingChunk`只能在缓存组中配置。如果不需要默认的缓存组，设置为`false`。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        default: false
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.priority`

`number`

一个模块可能会被多个缓存组匹配到，我们会根据具有更高优先级的缓存组来对它进行优化（简单的说就是优先级更高的缓存组才具有打包它的资格）。默认有个负数的优先级。

#### `splitChunks.cacheGroups.{cacheGroup}.reuseExistingChunk`

`boolean`

如果当前chunk包含的模块已经被主bundle打包进去，将不再被打包进当前chunk。这会影响到chunk name。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendors: {
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.test`

`function (module, chunk) | RegExp | string`

确定缓存组会选择哪些模块进行优化。默认会对所有模块进行优化。你可以对路径或者chunk name做匹配。如果匹配到一个chunk name，它的所有子模块都会被选择进行优化。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendors: {
          test (module, chunks) {
            //...
            return module.type === 'javascript/auto';
          }
        }
      }
    }
  }
};
```

#### `splitChunks.cacheGroups.{cacheGroup}.filename`

`string`

配置chunk name。`output.filename`的命名规则在这里也适用。

> 该选项也可以通过`splitChunks.filename`全局配置，但是我们不建议这么做，因为如果`splitChunks.chunks`没有设置为`initial`，有可能引发报错。

**webpack.config.js**

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendors: {
          filename: '[name].bundle.js'
        }
      }
    }
  }
};
```
