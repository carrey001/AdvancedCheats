## OkHttp3 使用方法
---
### OkHttp3的简单使用。

创建对象  创建请求参数 执行
#### Client and Request

```

public class OKhttpTest {

    public static void main(String[] args) {

        OkHttpClient okHttpClient = new OkHttpClient.Builder()//设置超时时间
                .readTimeout(10000L, TimeUnit.MILLISECONDS)
                .writeTimeout(10000L, TimeUnit.MILLISECONDS)
//                .cookieJar()设置 cookjar
//                .addInterceptor(new Interceptor() {//设置拦截器 比如说log
//                    @Override
//                    public Response intercept(Chain chain) throws IOException {
//                        Request request = chain.request();
//
////                      log4request(request)///请求的log
//                        Response response = chain.proceed(request);
////                      log4request(response)//结果的log
//
//                        return response;
//                    }
//                })

                .build();

                  // 天气请求接口 返回的是json
        Request request =new Request.Builder()
                .url("http://www.weather.com.cn/data/cityinfo/101010100.html")
                .build();

//         对于https
        Call call = okHttpClient.newCall(request);

        try {
            Response execute = call.execute();
            ResponseBody body = execute.body();
            System.out.println(body.string());//打印出Json
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}


```

这就是基本的http get请求。 上面用的是同步方法。

Client 和request 都是使用builder 模式建立的。
Client调用request会生成一个call对象 这个对象中有同步也有异步的请求。

#### Response 类
response.isSuccess();接口是否成功。
response.body(); 返回ResponseBody
responseBody.string();得到服务器返回的string
responseBody.byteStream();得到服务返回的流对象

#### Post请求

```

    public static void postKey(){

      。。。

        RequestBody requestBody =new FormBody.Builder()
                .add("platform","android")
                .add("name","sss")
                .build();

        Request request =new Request.Builder()
                .url("http://www.weather.com.cn/data/cityinfo/101010100.html")
                .post(requestBody)
                .build();

。。。

    }

```

跟get 的区别 就是再创建request的时候添加一个requestBody再body里面添加键值对。
如果post 提交json 数据 那 需要这样

```
public static void postJson(String json){

       MediaType JSON =MediaType.parse("application/json; charset=utf-8");
...
       RequestBody requestBody =RequestBody.create(JSON,json);

       Request request =new Request.Builder()
               .url("http://www.weather.com.cn/data/cityinfo/101010100.html")
               .post(requestBody)
               .build();

.....
   }

```

跟post 键值对没什么区别。 只是MediaType不同而已。


#### 上传下载文件

上传和下载

打印相应头的head信息

超过1M的body  应使用流来处理。

```
Response execute = client.newCall(request).execute();

   Headers responseHeaders = execute.headers();
   for (int i = 0; i < responseHeaders.size(); i++) {
     System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
   }
```

- post 提交流

```
RequestBody requestBody =new RequestBody() {
           @Override
           public MediaType contentType() {
               return XXXXXX;
           }

           @Override
           public void writeTo(BufferedSink sink) throws IOException {
              sink.write(......);
           }
       };
       Request request =new Request.Builder()
               .url("xxxxx")
               .post(requestBody)
               .build();


```
- post 提交文件

```
Request request =new Request.Builder()
         .url("xxxxx")
         .post(RequestBody.create(MEDIATYPE,file))
         .build();

```

- post 提交表单

```
RequestBody requestBody =new FormBody.Builder()
        .add("platform","android")
        .add("name","sss")
        .build();
        Request request =new Request.Builder()
                 .url("xxxxx")
                 .post(RequestBody.create(MEDIATYPE,file))
                 .build();    

```
- post 提交分块请求 Multipart

MultipartBuilder可以构建复杂的请求体，与HTML文件上传形式兼容。多块请求体中每块请求都是一个请求体，可以定义自己的请求头。这些请求头可以用来描述这块请求，例如他的Content-Disposition。如果Content-Length和Content-Type可用的话，他们会被自动添加到请求头中。

```

 RequestBody requestBody =new MultipartBody.Builder()
                .addPart(new MultipartBody.Part.)
                .addFormDataPart()
                .build();


```

- 相应缓存
为了缓存响应，你需要一个你可以读写的缓存目录，和缓存大小的限制。这个缓存目录应该是私有的，不信任的程序应不能读取缓存内容。
一个缓存目录同时拥有多个缓存访问是错误的。大多数程序只需要调用一次new OkHttp()，在第一次调用时配置好缓存，然后其他地方只需要调用这个实例就可以了。否则两个缓存示例互相干扰，破坏响应缓存，而且有可能会导致程序崩溃。
响应缓存使用HTTP头作为配置。你可以在请求头中添加Cache-Control: max-stale=3600 ,OkHttp缓存会支持。你的服务通过响应头确定响应缓存多长时间，例如使用Cache-Control: max-age=9600。
