# HTTP 详解
---
http 是一个应用层面向对象的协议，主要特点
- 支持c/s模式
- 快速访问 通讯快 只需要传递请求方法和路径
- 可以传递任何类型数据对象
- 每次链接只处理一个请求
- 每次处理完就结束

## 协议基础
---

http 采取**请求/响应**模式。客户端向服务器发送一个请求，请求头包含请求的方法、url、协议版本、以及包含请求修饰符、客户信息和内容。服务器以一个状态行为响应，响应内容包括消息协议的版本，成功或者错误编码加上包含服务器信息、实体元信息以及可能的实体内容。

### http请求
![http请求](image/http-1.png)
例如
```
GET /sample.xxx HTTP/1.1
Accept:image/gif.image/jpeg,*/*
Accept-Language:zh-cn
Connection:Keep-Alive
Host:localhost
User-Agent:Mozila/4.0
Accept-Encoding:gzip

username=123456&password=1234

```

- 请求方法：
 - GET *请求获取Request-URI所标识的资源*
 - POST *在Request-URI所标识的资源后附加新的数据*
 - PUT *请求服务器存储一个资源，并用Request-URI作为其标识*
 - DELETE *请求服务器删除Request-URI所标识的资源*
 - TRACE *请求服务器回送收到的请求信息，主要用于测试或诊断*
 - CONNECT *保留将来使用*
 - OPTIONS *请求查询服务器的性能，或者查询与资源相关的选项和需求*

 我们用的最多的是get 和post 请求。

<!-- 通用头域包括请求和响应都支持的头域，包括：**Cache-Control**、**Connection**、**Date**、**Pragma**、**Transfer-Encoding**、**Upgrade**、**Via** 等通用头域 -->

- Cache-Control: 指定请求和响应 遵循的缓存机制。 请求消息或者响应消息中设置 Cache-Control 并不会修改另一个消息处理过程中的缓存处理过程。请求时的缓存指令包括 **no-cache**、**no-store** 、**max-age**、**max-stale**、**min-fresh**、**only-if-cached**。响应消息中的指令包括 **public**、**private**、**no-cache**、**no-store**、**no-transform**、**must-revalidate**、**proxy-revalidate**、**max-age**。各个消息中的指令含义如下：

 - **no-cache**: 指示请求或响应消息不能缓存，下次请求资源必须冲新请求服务器。
 - **no-store**: 防止重要的信息被无意的发布。在请求消息中发送将使得请求和响应消息都不使用缓存。
 - **max-age**：指示客户机可以接收生存期不大于指定时间（以秒为单位）的响应。表示在指定时间内，客户端直接从缓存副本读取资源。
 - **max-stale**:指示客户机可以接收超出超时期间的响应消息。如果指定max-stale消息的值，那么客户机可以接收超出超时期指定值之内的响应消息。　
 - **min-fresh** :指示客户机可以接收响应时间小于当前时间加上指定时间的响应
 - **public** : 指示响应可被任何缓存区缓存。
 - **Private**: 指示对于单个用户的整个或部分响应消息，不能被共享缓存处理。这允许服务器仅仅描述当用户的部分响应消息，此响应消息对于其他用户的请求无效。


- Connection 链接规则，keep-alive 使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接

- Content-Encoding: gzip 告诉客户端，服务端发送的资源是采用gzip编码的，客户端看到这个信息后，应该采用gzip对资源进行解码。

- Content-Type: text/html;charset=UTF-8 告诉客户端，资源文件的类型，还有字符编码，客户端通过utf-8对资源进行解码，然后对资源进行html解析。通常我们会看到有些网站是乱码的，往往就是服务器端没有返回正确的编码。

- Date 头域表示消息发送时间

- Pragma 用来包含实现特定的指令，最常用的是Pragma:no-cache。在HTTP/1.1协议中，它的含义和Cache-Control:no-cache相同。

- Transfer-Encoding 传输时候的编码格式。


- referer 查看网页的http链接 经常能发现一个referer头。 这表示允许客户端指定请求URI的资源地址，可以允许服务器生成回退链表。

- range 下载文件的时候 能看到range 头域。表示可以请求实体的一个或者多个子范围。经常用于多线程下载。
例如 Range:bytes=0-999表示下载该文件的前1000 个字节。


### http响应

典型的响应消息：

```
HTTP/1.1 200OK
Date:Mon,31Dec200104:25:57GMT
Server:Apache/1.3.14(Unix)
Content-type:text/html
Last-modified:Tue,17Apr200106:46:28GMT
Etag:"a030f020ac7c01:1e9f"
Content-length:39725426
Content-range:bytes55******/40279980

```

HTTP/1.1 表示支持的http版本。status-code 表示结果码 例如这个是200。 ok 表示简单的文字描述。
status-code: 不同的数字表示不同的结果.
 - 1xx:信息响应类，表示接收到请求并且继续处理
 - 2xx:处理成功响应类，表示动作被成功接收、理解和接受
 - 3xx:重定向响应类，为了完成指定的动作，必须接受进一步处理
 - 4xx:客户端错误，客户请求包含语法错误或者是不能正确执行
 - 5xx:服务端错误，服务器不能正确执行一个正确的请求

### 网络框架中 需要手动拼接的请求头信息。

- Host
- Path
- 请求方法 ：get post等
- 编码格式和接收的内容 Content-type :接收类型，编码格式
- 压缩的格式 Accept-Encoding: gzip
- 以及一些业务上的请求头。
