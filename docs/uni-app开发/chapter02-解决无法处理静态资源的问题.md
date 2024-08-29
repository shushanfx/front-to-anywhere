# 解决处理处理静态资源的问题

说实在话，这个标题其实有点夸大了。 `uni-app`并没有不解决，而是解决的不够充分不够彻底。这里我们来讨论一下`uni-app`在处理静态资源的问题上的一些不足之处。

## 1. 问题描述

在`uni-app`中，我们可以将静态资源放在`static`目录下，然后通过相对路径的方式引用。比如我们有一个图片资源`logo.png`，我们可以通过`<img src="@/static/logo.png" />`的方式引用。

目前引用包括三个方式：

* `@/static/`：引用`static`目录下的资源；

* 通过`~@/static/`引用`static`目录下的资源。

如果需要引入其他目录的静态资源，那就只能通过变量的形式，如：

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