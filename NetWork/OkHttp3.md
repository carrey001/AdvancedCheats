## OkHttp3源码分析
---
最近rxJava retrofit 比较火爆。我们看到源码可以知道。retrofit是基于OKHttp的封装。所以我们首先 来看一下OkHttp 是怎么发送请求的。

>文中所取的名称均为作者脑补。如有不正确或者不准确的地方望私信或者评论

**OkHttp3中比较重要的几个类**

- OKHttpClient 客户端
- Request 请求
- Dispatcher  请求分派者
- StreamAllocation


### OKHttpClient

OKHttpClient是使用构建者模式来创建对象的。我们可以看到一个内部类Builder。
```
public static final class Builder {
    Dispatcher dispatcher;//请求分配者
    Proxy proxy; //代理
    List<Protocol> protocols;//协议
    List<ConnectionSpec> connectionSpecs;
    final List<Interceptor> interceptors = new ArrayList<>();//拦截器
    final List<Interceptor> networkInterceptors = new ArrayList<>();//网络拦截器
    ProxySelector proxySelector;//代理选择
    CookieJar cookieJar;//cookie
    Cache cache;//缓存
    InternalCache internalCache;//内部缓存接口
    SocketFactory socketFactory; //socket工厂
    SSLSocketFactory sslSocketFactory; //加密socket工厂
    TrustRootIndex trustRootIndex;//CA证书
    HostnameVerifier hostnameVerifier;//主机名认证
    CertificatePinner certificatePinner;
    Authenticator proxyAuthenticator;//代理身份凭证
    Authenticator authenticator;//身份凭证
    ConnectionPool connectionPool;//连接池
    Dns dns;
    boolean followSslRedirects;
    boolean followRedirects;
    boolean retryOnConnectionFailure;
    int connectTimeout;
    int readTimeout;
    int writeTimeout;
    ...

```
构建OKHttpClient的时候把你需要的参数和配置放到builder 里面 最后可生成你需要的OKHttpClient.
### Request
基本用法。当我们获得OKHttpClient的对象后就可以发送请求了。使用方法Call call=client.newCall(request);新建一个call 对象来进行发送请求操作。
```
OkHttpClient client = new OkHttpClient.Builder()
               .build();

Request request =new Request.Builder().url("http://www.github.com")
                .build();
    Call call = client.newCall(request);
    Response response=call.execute();

```
这里面我们需要个Request对象。这个对象我们也是通过构造者模式来创建。

```
public static class Builder {
   private HttpUrl url;//请求的url
   private String method;//请求的方法 默认get
   private Headers.Builder headers;//请求头
   private RequestBody body;//请求踢
   private Object tag;//标记。用于查找当前请求用

```
request对象配置了我们发送请求的参数。我们再build 的时候直接 构造出需要的request对象。

client.newCall(request)时候会创建一个对象RealCall。通过名字我们大致可以了解到这个类是真正的请求。
```
/**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return new RealCall(this, request);
  }

```
RealCall 继承Call  在execute（） 和enqueue（CallBack call）  做了具体的处理。

```
@Override
public Response execute() throws IOException {
  synchronized (this) {//同步请求
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain(false);
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    client.dispatcher().finished(this);
  }
}

@Override
public void enqueue(Callback responseCallback) {
  enqueue(responseCallback, false);//异步请求
}

void enqueue(Callback responseCallback, boolean forWebSocket) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));
}

```

从源码我们可以看到RealCall 的同步请求方法会调用
client.dispatcher().executed(this);
 getResponseWithInterceptorChain(false) 这个方法下面会说
异步请求方法enqueue中调用了  client.dispatcher().enqueue(new AsyncCall(responseCallback, forWebSocket));我们看到在Dispatcher对象中也有相同的两个方法executed 和enqueue。我们看看在Dispatcher 中到底做了什么操作。

### Dispatcher

类描述：Policy on when async requests are executed.
一个异步请求的执行策略。
查看下Dispatcher中的变量。基本上都是为了用于控制并发的请求

```
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();


```
- maxRequests = 64: 最大并发请求数为64
- maxRequestsPerHost = 5: 每个主机最大请求数为5
- ExecutorService：消费者池（也就是线程池）
- Deque<AsyncCalls> readyAsyncCalls：异步请求缓存队列
- Deque<AsyncCalls> runningAsyncCalls：异步正在运行的任务列表
- Deque<RealCall> runningSyncCalls:同步请求队列

我们看到Dispatcher里面有三个队列。所以所有的请求最后都会进入者三个队列然后。再被执行。我们看看再Dispatcher中的两个方法executed 和enqueue 到底做了些什么。

```

/** Used by {@code Call#execute} to signal it is in-flight. */
synchronized void executed(RealCall call) {
  runningSyncCalls.add(call);
}


synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

```
 果然 同步的时候executed方法直接把RealCall 加入了runningSyncCalls的队列中。
 异步的时候enqueue 会判断正在运行的队列runningAsyncCalls是否超过最大请求数。来塞到运行或者等待队列。然后会调用  executorService().execute(call);
 其实是执行AsyncCall中的execute()方法。

 ```

     @Override
     protected void execute() {
       boolean signalledCallback = false;
       try {
         Response response = getResponseWithInterceptorChain(forWebSocket);
         if (canceled) {
           signalledCallback = true;
           responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
         } else {
           signalledCallback = true;
           responseCallback.onResponse(RealCall.this, response);
         }
       } catch (IOException e) {
         if (signalledCallback) {
           // Do not signal the callback twice!
           logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
         } else {
           responseCallback.onFailure(RealCall.this, e);
         }
       } finally {
         client.dispatcher().finished(this);
       }
     }

 ```

 这里也看到了同步调用的一个方法    Response response = getResponseWithInterceptorChain(forWebSocket);调用过这个方法后直接获得了Response.所以我们看看这个方法中做了那些操作。

```
private Response getResponseWithInterceptorChain(boolean forWebSocket) throws IOException {
   Interceptor.Chain chain = new ApplicationInterceptorChain(0, originalRequest, forWebSocket);
   return chain.proceed(originalRequest);
 }

 class ApplicationInterceptorChain implements Interceptor.Chain {
   @Override
   public Response proceed(Request request) throws IOException {
     // If there's another interceptor in the chain, call that.
     if (index < client.interceptors().size()) {
       Interceptor.Chain chain = new ApplicationInterceptorChain(index + 1, request, forWebSocket);
       Interceptor interceptor = client.interceptors().get(index);
       Response interceptedResponse = interceptor.intercept(chain);
       if (interceptedResponse == null) {
        throw new NullPointerException("application interceptor " + interceptor
            + " returned null");
      }

       return interceptedResponse;
     }
     // No more interceptors. Do HTTP.
     return getResponse(request, forWebSocket);
   }
 }

```
在getResponseWithInterceptorChain方法中会创建一个对象。
ApplicationInterceptorChain 然后执行这个对象的proceed方法。再这个方法中会获取client中的Interceptor来执行。最终会执行 getResponse的这个方法。最终的网络请求方法会再这里执行。 下面看看这个方法的主要请求

```
Response getResponse(Request request, boolean forWebSocket) throws IOException {
    // Copy body metadata to the appropriate request headers.
        .....设置请求头
    // Create the initial HTTP engine. Retries and redirects need new engine for each attempt.
    //创建请求引擎
    engine = new HttpEngine(client, request, false, false, forWebSocket, null, null, null);

    int followUpCount = 0;
    while (true) {
      if (canceled) {
        engine.releaseStreamAllocation();
        throw new IOException("Canceled");
      }

      boolean releaseConnection = true;
      //发请求
      try {
        engine.sendRequest();
        engine.readResponse();
        releaseConnection = false;
      } catch (Exception e) {
      //异常处理 包括重试等。。。。
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          StreamAllocation streamAllocation = engine.close();
          streamAllocation.release();
        }
      }
      //获取结果
      Response response = engine.getResponse();
      Request followUp = engine.followUpRequest();
        //释放资源
      if (followUp == null) {
        if (!forWebSocket) {
          engine.releaseStreamAllocation();
        }
        return response;
      }

      StreamAllocation streamAllocation = engine.close();

      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (!engine.sameConnection(followUp.url())) {
        streamAllocation.release();
        streamAllocation = null;
      }

      request = followUp;
      engine = new HttpEngine(client, request, false, false, forWebSocket, streamAllocation, null,
          response);
    }
  }
```
