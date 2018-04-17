---
layout: post
title: "Apache HttpClient IV"
categories: Log4j2
tag: Java-Libs
---
> `Apache HttpClient 4.5.5 Tutorial - IV`

## 5. Fluent API

### 5.1 易于使用的facade API

从版本4.2开始，HttpClient基于fluent接口的概念提供了一个易于使用的facade API。Fluent Facade API只公开了HttpClient的最基本功能，适用于不需要HttpClient的全部灵活性的简单用例。例如，Fluent Facade API可以让用户不必处理连接管理和资源释放。

以下是通过HC fluent API执行的HTTP请求的几个示例：

```java
// Execute a GET with timeout settings and return response content as String.
Request.Get("http://somehost/")
        .connectTimeout(1000)
        .socketTimeout(1000)
        .execute().returnContent().asString();

// Execute a POST with the 'expect-continue' handshake, using HTTP/1.1,
// containing a request body as String and return response content as byte array.
Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .version(HttpVersion.HTTP_1_1)
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT)
        .execute().returnContent().asBytes();

// Execute a POST with a custom header through the proxy containing a request body
// as an HTML form and save the result to the file
Request.Post("http://somehost/some-form")
        .addHeader("X-Custom-header", "stuff")
        .viaProxy(new HttpHost("myproxy", 8080))
        .bodyForm(Form.form().add("username", "vip").add("password", "secret").build())
        .execute().saveContent(new File("result.dump"));
```

还可以直接使用Executor来执行特定安全上下文中的请求，从而将验证细节缓存并重新用于后续请求。

```java
Executor executor = Executor.newInstance()
        .auth(new HttpHost("somehost"), "username", "password")
        .auth(new HttpHost("myproxy", 8080), "username", "password")
        .authPreemptive(new HttpHost("myproxy", 8080));

executor.execute(Request.Get("http://somehost/"))
        .returnContent().asString();

executor.execute(Request.Post("http://somehost/do-stuff")
        .useExpectContinue()
        .bodyString("Important stuff", ContentType.DEFAULT_TEXT))
        .returnContent().asString();
```

#### 5.1.1 管理响应

Fluent Facade API通常可以减轻用户不必处理连接管理和资源释放的风险。然而，在大多数情况下，这需要在内存中缓冲响应消息的内容。强烈建议使用`ResponseHandler`进行HTTP响应处理，以避免必须缓冲内存中的内容。

```java
Document result = Request.Get("http://somehost/content")
        .execute().handleResponse(new ResponseHandler<Document>() {

    public Document handleResponse(final HttpResponse response) throws IOException {
        StatusLine statusLine = response.getStatusLine();
        HttpEntity entity = response.getEntity();
        if (statusLine.getStatusCode() >= 300) {
            throw new HttpResponseException(
                    statusLine.getStatusCode(),
                    statusLine.getReasonPhrase());
        }
        if (entity == null) {
            throw new ClientProtocolException("Response contains no content");
        }
        DocumentBuilderFactory dbfac = DocumentBuilderFactory.newInstance();
        try {
            DocumentBuilder docBuilder = dbfac.newDocumentBuilder();
            ContentType contentType = ContentType.getOrDefault(entity);
            if (!contentType.equals(ContentType.APPLICATION_XML)) {
                throw new ClientProtocolException("Unexpected content type:" +
                    contentType);
            }
            String charset = contentType.getCharset();
            if (charset == null) {
                charset = HTTP.DEFAULT_CONTENT_CHARSET;
            }
            return docBuilder.parse(entity.getContent(), charset);
        } catch (ParserConfigurationException ex) {
            throw new IllegalStateException(ex);
        } catch (SAXException ex) {
            throw new ClientProtocolException("Malformed XML document", ex);
        }
    }
});
```

## 6. Http缓存

### 6.1 总体概念

HttpClient Cache提供了一个与HTTP/1.1兼容的缓存层，可以与HttpClient一起使用（与浏览器缓存相当）。该实现遵循责任链设计模式，其中缓存HttpClient实现可以用于替代默认非缓存HttpClient实现；完全可以从缓存中满足的请求不会执行实际的原始请求。使用GET和`If-Modified-Since`和/或`If-None-Match`请求头，可以尽可能使用原始地址自动验证陈旧的高速缓存条目。

一般而言，HTTP/1.1缓存设计为语义透明；也就是说，缓存不应该改变客户端和服务器之间请求-响应交换的含义。因此，将缓存的HttpClient放入现有的兼容客户端-服务器关系中应该是安全的。尽管从HTTP协议的角度来看，缓存模块是客户端的一部分，但实现的目标是与透明缓存代理的要求相兼容。

最后，缓存HttpClient包括支持由RFC5861指定的缓存控制扩展（stale-if-error和stale-while-revalidate）。

当缓存HttpClient执行请求时，它会经历以下流程：

- 检查请求是否基本符合HTTP 1.1协议的并尝试更正请求。

- 刷新将被该请求认定无效的任何缓存条目。

- 确定当前请求是否可以从缓存中获取。如果不是，直接将请求传递给原始服务器，并在适当的情况下缓存后返回响应。

- 如果请求是缓存可加载的，它将尝试从缓存中读取。如果它不在缓存中，则调用原始服务器并缓存响应（如果适用）。

- 如果缓存的响应适合作为响应，则构造包含`ByteArrayEntity`的`BasicHttpResponse`并将其返回。否则，尝试针对原始服务器重新验证缓存条目。

- 对于无法重新验证的缓存响应，请调用原始服务器并缓存响应（如果适用）。

当缓存HttpClient收到响应时，它会经历以下流程：

- 检查响应是否遵从协议。

- 确定响应是否可缓存。

- 如果它是可缓存的，则尝试读取配置中允许的最大缓存大小并将其存储在缓存中。

- 如果缓存的响应太大，则重新构建部分消耗的响应，并直接返回而不进行缓存。

需要注意的是，caching HttpClient本身不是HttpClient的不同实现，而是通过将自身作为附加处理组件插入到请求执行管道来工作。

### 6.2 RFC-2616条款

我们认为HttpClient Cache无条件符合RFC-2616。也就是说，无论规范如何指定MUST，MUST NOT，SHOULD或SHOULD NOT都不适用于HTTP缓存，缓存层将试图以满足以上要求的方式行事。这意味着缓存模块在放入时不会产生不正确的行为。

### 6.3 使用示例

以下是设置基本缓存HttpClient的简单示例。按照配置，它将最多存储1000个缓存对象，每个对象的最大主体大小可能为8192字节。 这里选择的数字仅用于举例，并不打算作为规定或建议。

```java
CacheConfig cacheConfig = CacheConfig.custom()
        .setMaxCacheEntries(1000)
        .setMaxObjectSize(8192)
        .build();
RequestConfig requestConfig = RequestConfig.custom()
        .setConnectTimeout(30000)
        .setSocketTimeout(30000)
        .build();
CloseableHttpClient cachingClient = CachingHttpClients.custom()
        .setCacheConfig(cacheConfig)
        .setDefaultRequestConfig(requestConfig)
        .build();

HttpCacheContext context = HttpCacheContext.create();
HttpGet httpget = new HttpGet("http://www.mydomain.com/content/");
CloseableHttpResponse response = cachingClient.execute(httpget, context);
try {
    CacheResponseStatus responseStatus = context.getCacheResponseStatus();
    switch (responseStatus) {
        case CACHE_HIT:
            System.out.println("A response was generated from the cache with " +
                    "no requests sent upstream");
            break;
        case CACHE_MODULE_RESPONSE:
            System.out.println("The response was generated directly by the " +
                    "caching module");
            break;
        case CACHE_MISS:
            System.out.println("The response came from an upstream server");
            break;
        case VALIDATED:
            System.out.println("The response was generated from the cache " +
                    "after validating the entry with the origin server");
            break;
    }
} finally {
    response.close();
}
```

### 6.4 配置

caching HttpClient继承默认非缓存实现的所有配置选项和参数（这包括设置选项，如超时和连接池大小）。对于特定于缓存的配置，可以提供一个`CacheConfig`实例来自定义以下区域的行为：

- **缓存大小**。如果后端存储支持这些限制，则可以指定最大缓存条目数以及最大可缓存响应主体大小。

- **公/私缓存**。默认情况下，缓存模块认为自己是共享（公共）缓存，并且不会为具有Authorization头的请求响应或具有`Cache-Control:private`标记的响应进行缓存。但是，如果缓存仅由一个逻辑用户（与浏览器缓存类似）运行，那么您将需要关闭共享缓存设置。

- **启发式缓存**。根据RFC2616，即使服务器没有明确的缓存控制头，缓存可以缓存若干个缓存条目。此行为在默认情况下处于关闭状态，但如果你正在使用的服务器未设置正确头但仍想缓存响应，则可能需要将其打开。你需要启用启发式缓存，然后指定默认的生命周期和/或上次修改资源以来的一小部分时间。有关启发式缓存的更多详细信息，请参阅HTTP/1.1 RFC的第13.2.2和13.2.4节。

- **后台验证**。缓存模块支持RFC5861的stale-while-revalidate指令，该指令允许在后台发生某些缓存条目重新验证。您可能需要调整设置后台工作线程的最小和最大数量，以及它们在被回收之前的最长可闲置的时间。当没有足够的线程满足需求时，您还可以控制用于重新验证的队列大小。

### 6.4 后端存储

caching HttpClient的默认实现将缓存条目和缓存响应的主体存储在应用程序的JVM的内存中。虽然这提供了高性能，但由于缓存条目的大小限制或生存时间短暂且在应用程序重新启动后无法继续使用，可能不适合您的应用程序。当前版本包括支持使用EhCache和memcached的实现存储缓存条目，这允许将缓存条目存储到磁盘或将其存储在外部进程中。

如果这些选项都不适合您的应用程序，那么可以通过实现`HttpCacheStorage`接口提供您自己的后端存储，然后在构建时向caching HttpClient提供它。在这种情况下，缓存条目将使用您的方案进行存储，但您将重用所有关于HTTP/1.1遵从性和缓存处理的逻辑。一般来说，应该可以使用支持键/值存储（类似于Java Map接口）的任何东西来创建`HttpCacheStorage`实现，并且能够进行原子更新。

最后，通过一些额外的努力，完全有可能建立一个多层缓存层次结构；例如，将一个在内存的caching HttpClient包装在一个存储缓存项的本地磁盘或远程memcached中，遵循类似于虚拟内存，L1/L2处理器缓存等的模式。