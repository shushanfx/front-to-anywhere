# 解决微信小程序开发无法识别 sourcemap 的问题

## 概论

通过`uni-app`为小程序开发提供了极大的便利。然而，`uni-app`在处理`sourcemap`的问题上并不是很友好，它在测试环境的输出产物类似效果如下：

```javascript
// mp-weixin/common/runtime.js
// 此处省略JS代码

// # sourceMappingURL=../.sourcemap/mp-weixin/common/runtime.js.map
```

看起来输出了`sourcemap`文件以及对应的路径，但是实际上，微信小程序开发工具并不能识别这个`sourcemap`文件，导致我们在调试的时候并不能很好地定位到源码的位置。

## 为什么不能识别

使用微信小程序开发助手，开发同学可以很快发现，`sourcemap`文件根本找不到，这是为什么呢？

我们知道，微信小程序开发助手使用`nw`作为运行环境，一个具体的小程序实际上是通过`nw`创建了一个随机port的服务，小程序加载wxml、wxss、js等文件时，实际上是构建了一个个对应的`http`请求，请求路径即`http://127.0.0.1:{port}/{path}`。当解析到`# sourceMappingURL`时，会根据当前文件的路径，拼接上对应的`sourcemap`文件路径，然后再次请求这个`sourcemap`文件。

接下来我们看`uni-app`的输出产物：

```text
dist
|── .sourcemap
|── mp-weixin
|   ├── pages
|   ├── 小程序相关文件
```

小程序开发助手加载的文件限定为`dist/mp-weixin`目录下的文件，而`sourcemap`文件则被放在了`dist/.sourcemap`目录下，这就导致了微信小程序开发助手无法找到对应的`sourcemap`文件。

## 解决方案

怎么解决呢？首先，确定目标，即要让微信小程序开发助手能够找到对应的`sourcemap`文件。基于这个目标，我们设想如下方案：

1. 将`.sourcemap`放到`dist/mp-weixin`目录下；

2. 使用inline模式，将`sourcemap`文件内容直接写入到对应的文件中。

方案1和方案2均会导致mp-weixin目录下的文件变得臃肿，使用真机调试时简直是一个灾难。那么，有没有更好的方案呢？

答案是有的。参照我们在上一章中讲到的处理静态资源的经验，我们可以将`sourcemap`使用单独的静态资源服务器代理起来，同时将sourcemap
的URL地址替换为代理服务器的地址。这样，我们就可以在微信小程序开发助手中正常使用`sourcemap`了。

## 具体实现
