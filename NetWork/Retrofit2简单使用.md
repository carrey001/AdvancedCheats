##　Retrofit 简单使用
---

> [Retrofit 源码分析](Retrofit2.md)

>**[Retrofit官方文档](http://square.github.io/retrofit/)**

### 基本配置
retrofit 把网络请求写在一个接口interface 文件中。
把你的HTTP API改造成java接口。例如

```
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}

```
Retrofit 对象会使用create方法来创建一个service的impl对象。

```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);

```
 然后使用service来创建Call对象,来进行同步或者异步的网络请求。

 ```
 Call<List<Repo>> repos = service.listRepos("octocat");
 repos.execyte();//同步请求
 repos.enqueue(new CallBack<Repo>(){......});//异步请求

 ```

 如果需要取消正在进行中的业务 可以使用call.cancel();
 ```
 call.cancel();

 ```
在retrofit2.0 版本中Converter已经删除。需要我们手动添加。 否则的话，Retrofit只能接收字符串结果。
在生成Retrofit对象时就使用addConverterFactory把他添加进来。


 ```
 Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("http://api.nuuneoi.com/base/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();

service = retrofit.create(APIService.class);

 ```
现在移动端一般使用Gson 服务端一般使用jackson。下面是Square提供的官方Converter modules列表。选择一个最满足你需求的。

```
Gson: com.squareup.retrofit:converter-gson

Jackson: com.squareup.retrofit:converter-jackson

Moshi: com.squareup.retrofit:converter-moshi

Protobuf: com.squareup.retrofit:converter-protobuf

Wire: com.squareup.retrofit:converter-wire

Simple XML: com.squareup.retrofit:converter-simplexml

```

### 请求方法
retrofit是使用注解来注入请求的

**get**

- 没有可变参数

```
@GET("users/list")

@GET("users/list?sort=desc")

```
- 带有可变地址

```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);

```
- 带有查询参数(参数过多可使用map)

```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);

@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);

```
**POST**

- body @Body

```
@POST("users/new")
Call<User> createUser(@Body User user);

```

- 表单提交 @FormUrlEncoded

```
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);

```
- multipart @Multipart

```
@Multipart
@PUT("user/photo")
Call<User> updateUser(@Part("photo") RequestBody photo, @Part("description") RequestBody description);

```
**Header**

- header @Headers

```
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();


@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);


@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)

```

如果想设置通用的head文件 需要使用OKHttp3的拦截器Interceptor。

```
Retrofit retrofit = new Retrofit.Builder()
       .baseUrl("http://api.nuuneoi.com/base/")
       .addConverterFactory(GsonConverterFactory.create())
       .addInterceptor(new Interceptor() {
                   @Override
                   public Response intercept(Chain chain) throws IOException {
                       Headers.Builder builder = new Headers.Builder();
                       builder.add("version", "1,0");
                       builder.add("app-version", "app-version");
                       builder.add("device", "device");
                       builder.add("screen-width", "screen-width");
                       builder.add("screen-height", "screen-width");
                       builder.add("platform", "1");
                       builder.add("os-version", "6.0");
                       builder.add("model", "model");
                       builder.add("access-token", "access-token");
                       Request request = chain.request().newBuilder()
                               .headers(builder.build())
                               .build();
                       return chain.proceed(request);
                   }
               })
       .build();

```

### rxjava

要在Retrofit中使用rxjava则需要配置CallAdapter module

```
Retrofit retrofit = new Retrofit.Builder()
       .baseUrl("http://api.nuuneoi.com/base/")
       .addConverterFactory(GsonConverterFactory.create())
       .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
       .build();

```
则现在就可以使用Observable作为返回了。 使用rxjava来进行线程的操作。
