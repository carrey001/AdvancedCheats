##  OkHttp3源码分析StreamAllocation
---

最近rxJava retrofit 比较火爆。我们看到源码可以知道。retrofit是基于OKHttp的封装。所以我们首先 来看一下OkHttp 是怎么发送请求的。

>文中所取的名称均为作者脑补。如有不正确或者不准确的地方望私信或者评论

**OkHttp3中比较重要的几个类**

- OKHttpClient 客户端
- Request 请求
- Dispatcher  请求分派者
- StreamAllocation  这个真不知道怎么翻译（-_-）

### StreamAllocation
