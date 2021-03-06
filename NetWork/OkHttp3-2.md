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

上一篇文章中说了OkHttp3  中的前三点， 再文章的最后出现了个类StreamAllocation 。本片文章就详细了解下这个类。
按照字面意思直接翻译应该叫 分配流
看看这个类的成员变量和构造方法。


```

public final class StreamAllocation {
  public final Address address;
  private Route route;
  private final ConnectionPool connectionPool;

  // State guarded by connectionPool.
  private RouteSelector routeSelector;
  private RealConnection connection;
  private boolean released;
  private boolean canceled;
  private HttpStream stream;

  public StreamAllocation(ConnectionPool connectionPool, Address address) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.routeSelector = new RouteSelector(address, routeDatabase());
  }


```
- Address 地址包含请求的一切地址信息
- Route  链路
- ConnectionPool 连接池
- RouteSelector 链路选择
- RealConnection 连接链路
- HttpStream http序列化对象
- released  已经链接的状态
- canceled  已经取消的状态

构造方法中需要两个参数(ConnectionPool connectionPool, Address address)线程池和地址，初始化了routeSelector线路选择。


### routeSelector  
链路选择。 此对象主要管理代理和端口  里面有个主要的方法 next（） 是个递归方法。 有点绕
先看构造方法： 构造方法中会去调用  resetNextProxy(address.url(), address.proxy());
来为当前链路准备代理服务
```
/** Prepares the proxy servers to try. */
 private void resetNextProxy(HttpUrl url, Proxy proxy) {
   if (proxy != null) {
     // If the user specifies a proxy, try that and only that.
     proxies = Collections.singletonList(proxy);
   } else {
     // Try each of the ProxySelector choices until one connection succeeds. If none succeed
     // then we'll try a direct connection below.
     proxies = new ArrayList<>();
     List<Proxy> selectedProxies = address.proxySelector().select(url.uri());
     if (selectedProxies != null) proxies.addAll(selectedProxies);
     // Finally try a direct connection. We only try it once!
     proxies.removeAll(Collections.singleton(Proxy.NO_PROXY));
     proxies.add(Proxy.NO_PROXY);
   }
   nextProxyIndex = 0;
 }
```
如果proxy 不为null  就把代理添加到集合proxies中。

如果proxy 为null 创建一个Proxy.NO_PROXY 添加到集合proxies中。

next() 方法中：
```
public Route next() throws IOException {
    // Compute the next route to attempt.
    if (!hasNextInetSocketAddress()) {
      if (!hasNextProxy()) {
        if (!hasNextPostponed()) {
          throw new NoSuchElementException();
        }
        return nextPostponed();
      }
      lastProxy = nextProxy();
    }
    lastInetSocketAddress = nextInetSocketAddress();

    Route route = new Route(address, lastProxy, lastInetSocketAddress);
    if (routeDatabase.shouldPostpone(route)) {
      postponedRoutes.add(route);
      // We will only recurse in order to skip previously failed routes. They will be tried last.
      return next();
    }

    return route;
  }


```
三个if else 判断
先看if (!hasNextInetSocketAddress()) 是否 有socket地址，没有的话
再看  if (!hasNextProxy()) 是否有代理（Proxy.NO_PROXY）也算有。有的话 缓存到lastProxy中。
最后 if (!hasNextPostponed())  是否有推迟的链路 如果没有了 就会报出NoSuchElementException 有就取出第一个。
 最后会取出地址然后创建个Route对象返回。

> 所以最后通过routeSelector对象可以去取出一个链路。

### RealConnection
接下来我们再看这个对象。
 这个对象里面做了一些 底层的请求操作感兴趣的朋友可以自己看看。我们主要讲StreamAllocation中关于RealConnection 的几个方法。

```

 private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
     int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
     throws IOException, RouteException {
   while (true) {
     RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
         connectionRetryEnabled);

     // If this is a brand new connection, we can skip the extensive health checks.
     synchronized (connectionPool) {
       if (candidate.successCount == 0) {
         return candidate;
       }
     }

     // Otherwise do a potentially-slow check to confirm that the pooled connection is still good.
     if (candidate.isHealthy(doExtensiveHealthChecks)) {
       return candidate;
     }

     connectionFailed(new IOException());
   }
 }

 /**
  * Returns a connection to host a new stream. This prefers the existing connection if it exists,
  * then the pool, finally building a new connection.
  */
 private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
     boolean connectionRetryEnabled) throws IOException, RouteException {
   Route selectedRoute;
   synchronized (connectionPool) {
     if (released) throw new IllegalStateException("released");
     if (stream != null) throw new IllegalStateException("stream != null");
     if (canceled) throw new IOException("Canceled");

     RealConnection allocatedConnection = this.connection;
     if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
       return allocatedConnection;
     }

     // Attempt to get a connection from the pool.
     RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);
     if (pooledConnection != null) {
       this.connection = pooledConnection;
       return pooledConnection;
     }

     selectedRoute = route;
   }

   if (selectedRoute == null) {
     selectedRoute = routeSelector.next();
     synchronized (connectionPool) {
       route = selectedRoute;
     }
   }
   RealConnection newConnection = new RealConnection(selectedRoute);
   acquire(newConnection);

   synchronized (connectionPool) {
     Internal.instance.put(connectionPool, newConnection);
     this.connection = newConnection;
     if (canceled) throw new IOException("Canceled");
   }

   newConnection.connect(connectTimeout, readTimeout, writeTimeout, address.connectionSpecs(),
       connectionRetryEnabled);
   routeDatabase().connected(newConnection.route());

   return newConnection;
 }

```
这里有两个方法。findHealthyConnection 查找健康的链路 返回值是RealConnection.
里面调用了findConnection的方法。
因为从线程池里拿出来的链接。所以对线程池加了同步锁。
判断当前connection是否为null 是否有新的流 如果没有 就返回当前链接。 如果没有，就 RealConnection pooledConnection = Internal.instance.get(connectionPool, address, this);

这个方法中获取新的链接  如果 没有获取到 就新建个链接。



### HttpStream
Http流。这个对象会封装http 请求。 这个是一个接口。有两个实现类
- Http2xStream
- Http1xStream
 这个类封装了 请求的所有参数 还有返回的所有数据
例如以下的方法。
- createRequestBody
- writeRequestHeaders
- writeRequestBody
- finishRequest
- openResponseBody

我们再这个地方才真正的创建http 请求参数。然后发送请求接受返回值。
 对于okHttp的使用方法。我会再接下来的博客中写来。
