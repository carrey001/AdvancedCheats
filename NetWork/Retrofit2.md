# Retrofit 2源码分析
---

<h2>[Retrofit官方文档](http://square.github.io/retrofit/)</h2>

> Retrofit是一种对OKHttp3的封装。简化了OKHttp3的请求。并支持rxjava。

## Retrofit.java

Retrofit的构造方法是包权限是默认的，所以使用了构造者模式来创建Retrofit.

```
Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
    List<Converter.Factory> converterFactories, List<CallAdapter.Factory> adapterFactories,
    Executor callbackExecutor, boolean validateEagerly) {
  this.callFactory = callFactory;
  this.baseUrl = baseUrl;
  this.converterFactories = unmodifiableList(converterFactories); // Defensive copy at call site.
  this.adapterFactories = unmodifiableList(adapterFactories); // Defensive copy at call site.
  this.callbackExecutor = callbackExecutor;
  this.validateEagerly = validateEagerly;
}

```

Retrofit.Builder:的所有成员变量都是Retrofit生成对象所需要的参数。

```
public static final class Builder {
    private Platform platform;
    private okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private List<Converter.Factory> converterFactories = new ArrayList<>();
    private List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
    private Executor callbackExecutor;
    private boolean validateEagerly;

```
我们看到Builder.build();方法中有 baseUrl 不可以为null，其他的为null 则赋默认值。

```
public Retrofit build() {
     if (baseUrl == null) {
       throw new IllegalStateException("Base URL required.");
     }

     okhttp3.Call.Factory callFactory = this.callFactory;
     if (callFactory == null) {
       callFactory = new OkHttpClient();
     }

     Executor callbackExecutor = this.callbackExecutor;
     if (callbackExecutor == null) {
       callbackExecutor = platform.defaultCallbackExecutor();
     }

     // Make a defensive copy of the adapters and add the default Call adapter.
     List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
     adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

     // Make a defensive copy of the converters.
     List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

     return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
         callbackExecutor, validateEagerly);
   }
 }


```

再Retrofit中使用create方法来创建接口对象。

```
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }

```
 Utils.validateServiceInterface(service); 判断service 必须是接口文件，并且没有父类。

 后面代理会生成对象。Proxy.newProxyInstance(...);
InvocationHandler()根据几种情况创建出对象。

ServiceMethod serviceMethod = loadServiceMethod(method);

OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);

这两句来创建 ServiceMethod 和Call 对象。
在ServiceMethod 对象中有toRequest方法来构造出此次请求的请求数据。
```
/** Builds an HTTP request from method arguments. */
 Request toRequest(Object... args) throws IOException {
   RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
       contentType, hasBody, isFormEncoded, isMultipart);

   @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
   ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

   int argumentCount = args != null ? args.length : 0;
   if (argumentCount != handlers.length) {
     throw new IllegalArgumentException("Argument count (" + argumentCount
         + ") doesn't match expected count (" + handlers.length + ")");
   }

   for (int p = 0; p < argumentCount; p++) {
     handlers[p].apply(requestBuilder, args[p]);
   }

   return requestBuilder.build();
 }

```
ServiceMethod.toRequest方法再哪里执行呢？我们得到call 对象时就可以执行网络请求了。同步的是execute        
异步是enqueue。在这两种网络请求中都会调用一个方法createRawCall()，在这个方法的第一行我们就可以看到serviceMethod.toRequest执行了。
```
private okhttp3.Call createRawCall() throws IOException {
   Request request = serviceMethod.toRequest(args);
   okhttp3.Call call = serviceMethod.callFactory.newCall(request);
   if (call == null) {
     throw new NullPointerException("Call.Factory returned null.");
   }
   return call;
 }

```       
在 enqueue 和execute 两个方法中 大部分是容错操作。最后都会调用一个方法来获得response.

```
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

```
parseResponse  解析Response。code 200-300 之间的 为网络成功。最后为了获取实体类。会调用T body = serviceMethod.toResponse(catchingBody); 在serviceMethod中是拿到responseConverter 来转换成需要的实体。
这里的responseConverter  就是再Retrofit.class中配置的Converter.

```
 T toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }

```
