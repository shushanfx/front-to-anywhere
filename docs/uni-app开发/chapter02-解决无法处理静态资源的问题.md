# 解决处理处理静态资源的问题

说实在话，这个标题其实有点夸大了。 `uni-app`并没有不解决，而是解决的不够充分不够彻底。这里我们来讨论一下`uni-app`在处理静态资源的问题上的一些不足之处。

## 1. 问题描述

在`uni-app`中，我们可以将静态资源放在`static`目录下，然后通过相对路径的方式引用。比如我们有一个图片资源`logo.png`，我们可以通过`<img src="@/static/logo.png" />`的方式引用。

目前引用包括三个方式：

* `@/static/`：引用`static`目录下的资源；

* 通过`~@/static/`引用`static`目录下的资源。

* 如果需要引入其他目录的静态资源，那就只能通过变量的形式，如：

```vue
<template>
  <view>
    <image src="/static/logo.png"></image>
    <image :src="imgUrl"></image>
  </view>
</template>

<script>
import imgUrl from './a.png'
export default {
  data() {
    return {
      imgUrl
    }
  }
}

</script>
```

我们接下来看，它最终的编译产物是什么呢？

编译之后的形成的文件夹结构如下：

```text
mp-weixin
|── assets
│   ├── a.xxxxxxx.png
├── static
│   ├── logo.png
```

可以看到，`static`目录下的资源是直接拷贝到了`mp-weixin`目录下，而其他目录下的资源则是被编译成了`assets`目录下的文件。

最终生成的WXML文件中，引用的路径是：

```xml
<image src="/static/logo.png"></image>
<image src="{{imgUrl}}"></image>
```

JS文件中引用的路径是：

```javascript
const imgUrl = __webpack_require__('./a.png')
```

同时在runtime.js中，有`./a.png`路径的定义

```javascript
  './a.png': function(module, exports, __webpack_require__) {
    module.exports = '/assets/a.xxxxxxx.png';
  }
```

> 以上代码均为伪代码，实际情况可能有所不同。

看起来，对于`static`目录以及其他目录的静态资源都处理了， 最终编译产物也是符合预期的。但是，这里有一个问题，`static`和`assets`目录的资源都位于`mp-weixin`目录下，这将导致小程序整体包大小的增大。

熟悉小程序开发的同学都知道，小程序的包大小是有限制的，根据官方介绍，目前限制如下：

* 整个小程序所有分包大小不超过 30M（服务商代开发的小程序不超过 20M）
* 单个分包/主包大小不能超过 2M

很明显，我们不能过多的将资源放在编译后的`mp-weixin`目录下。因此如何对该部分资源进行优化变成了一个问题。

## 2. 解决方案

描述到上述问题之后，我们如何解决这个问题呢？我们有必要描述一下我们的目标：

* 开发环境，能够将`assets`目录移出`mp-weixin`文件夹，并提供文件服务器，以便于开发时能够访问到资源；

* 生产环境，能够将`assets`目录移出`mp-weixin`文件夹，删除`assets`文件，并将资源放在CDN上，以便于小程序能够访问到资源。

那么具体怎么做呢 ？

仔细研究`webpack`的构建过程，我们发现`webpack`的complier的`hooks`包含了几个特殊的阶段：

* emit：资源输出阶段，这个阶段是在资源输出之前的一个阶段，我们可以在这个阶段对资源进行处理。

* afterEmit：资源输出后阶段，这个阶段是在资源输出之后的一个阶段，我们可以在这个阶段对资源进行处理。

* done：构建完成阶段，这个阶段是在构建完成之后的一个阶段，我们可以在这个阶段对资源进行处理。

> 不涉及的hooks我们就不在赘述了，有兴趣的同学可以查看[webpack官方文档](https://webpack.js.org/api/compiler-hooks/)

从上面的描述我们可以看出，我们可以在`afterEmit`和`done`阶段对资源进行处理，将`assets`目录移出`mp-weixin`文件夹，同时开启一个本地的服务器，以便于开发时能够访问到资源。

### 代码解释

我们新建一个`build`目录，然后在`build`目录下新建一个`webpack-plugin-file.js`文件，用于定义一个webpack plugin：

```javascript
module.exports = class WebpackPluginFile {
  constructor(options) {
    this.options = options;
  }

  apply(compiler) {
    this.logger = compiler.getInfrastructureLogger('WebpackPluginFile');
    compiler.hooks.done.tap('WebpackPluginFile', () => {
      this.copyAssets(compiler).then(() => {
        this.startServer(compiler);
      });
    });
  }
}
```

接下来我们要实现`copyAssets`。

```javascript
const fs = require('fs');
const fsPromise = require('fs/promises');
const path = require('path');

module.exports = class WebpackPluginFile {
  async copyAssets(compiler) {
    const outputPath = compiler.options.output.path;
    const assetsPath = path.resolve(outputPath, 'assets');
    const targetPath = path.resolve(outputPath, '..', 'assets');
    return this.copyDir(assetsPath, targetPath);
  }
  async copyDir(src, dist) {
    const stats = await fsPromise.stat(src).catch(() => null);
    if (!stats) {
      // 判断是否存在，如果不存在则返回
      return null;
    }
    if (stats.isFile()) {
      // 如果是文件，则直接拷贝
      return this.copyFile(src, dist);
    }
    // 如果是目录，则遍历目录
    if (stats.isDirectory()) {
      const paths = await fsPromise.readdir(src);
      for (let i = 0; i < paths.length; i++) {
        const path = paths[i];
        await this.copyDir(
          path.resolve(src, path),
          path.resolve(dist, path)
        );
      }
    }
  }
}
```



## 参考资料

* [uni-app官方文档](https://uniapp.dcloud.io/)
* [小程序分包加载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages.html)