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



## 参考资料

* [uni-app官方文档](https://uniapp.dcloud.io/)
* [小程序分包加载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages.html)