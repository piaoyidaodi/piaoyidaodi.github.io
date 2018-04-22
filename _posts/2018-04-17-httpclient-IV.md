---
layout: post
title: "Apache HttpClient IV"
categories: HttpClient
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

## 7. 高级主题

### 7.1 自定义客户端连接

在某些情况下，为了能够处理非标准的，不符合规范的行为，可能有必要定制HTTP消息的传输方式，使用超出HTTP参数范围的形式进行传输。例如，对于Web爬虫，可能需要强制HttpClient接受格式错误的响应头以挽救消息的内容。

通常，插入自定义消息解析器或自定义连接实现的过程涉及几个步骤：

- 提供一个定制的`LineParser`/`LineFormatter`接口实现。根据需要实现消息解析/格式化逻辑。

```java
class MyLineParser extends BasicLineParser {

    @Override
    public Header parseHeader(
            CharArrayBuffer buffer) throws ParseException {
        try {
            return super.parseHeader(buffer);
        } catch (ParseException ex) {
            // Suppress ParseException exception
            return new BasicHeader(buffer.toString(), null);
        }
    }

}
```

- 提供一个自定义的`HttpConnectionFactory`实现。根据需要将默认请求writer和/或响应解析器替换为自定义的。

```java
HttpConnectionFactory<HttpRoute, ManagedHttpClientConnection> connFactory =
        new ManagedHttpClientConnectionFactory(
            new DefaultHttpRequestWriterFactory(),
            new DefaultHttpResponseParserFactory(
                    new MyLineParser(), new DefaultHttpResponseFactory()));
```

- 配置HTTPClient使用自定义连接工厂。

```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(
    connFactory);
CloseableHttpClient httpclient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();
```

### 7.2 有状态HTTP连接

虽然HTTP规范假定会话状态信息总是以HTTP cookie的形式嵌入到HTTP消息中，因此HTTP连接始终是无状态的，但这种假设在现实生活中并不总是成立。在某些情况下，HTTP连接是使用特定的用户身份或特定的安全上下文创建的，因此无法与其他用户共享，并且只能由同一用户重用。此类有状态HTTP连接的示例是NTLM身份验证连接和带有客户端证书身份验证的SSL连接。

#### 7.2.1 管理用户token

HttpClient依靠`UserTokenHandler`接口来确定给定的执行上下文是否是用户特定的。如果上下文是用户特定的，则该处理程序返回的token对象应该唯一标识当前用户，或者如果上下文不包含任何特定于当前用户的资源或细节，则为null。用户token将用于确保特定用户资源不会与其他用户共享或重新使用。

`UserTokenHandler`接口的默认实现使用`Principal`类的实例来表示HTTP连接的状态对象，如果它可以从给定的执行上下文中获取的话。 `DefaultUserTokenHandler`将使用基于用户规则连接的身份验证方案（如NTLM）或开启客户机身份验证的SSL会话。如果两者都不可用，则返回null token。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    Principal principal = context.getUserToken(Principal.class);
    System.out.println(principal);
} finally {
    response.close();
}
```

如果默认用户不满足他们的需求，用户可以提供自定义实现：

```java
UserTokenHandler userTokenHandler = new UserTokenHandler() {

    public Object getUserToken(HttpContext context) {
        return context.getAttribute("my-token");
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setUserTokenHandler(userTokenHandler)
        .build();
```

#### 7.2.2 持久化状态连接

请注意，只有在执行请求时将相同的状态对象绑定到执行上下文，才能重用携带状态对象的持久连接。因此，确保同一个上下文由同一用户重用并执行后续HTTP请求，或者在执行请求之前将用户token绑定到上下文非常重要。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context1 = HttpClientContext.create();
HttpGet httpget1 = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response1 = httpclient.execute(httpget1, context1);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
Principal principal = context1.getUserToken(Principal.class);

HttpClientContext context2 = HttpClientContext.create();
context2.setUserToken(principal);
HttpGet httpget2 = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response2 = httpclient.execute(httpget2, context2);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

### 7.3 使用F`utureRequestExecutionService`

使用`FutureRequestExecutionService`，您可以安排http调用并将响应视为`Future`。这是很有用的，比如，多次调用Web服务。使用`FutureRequestExecutionService`的优点是可以使用多个线程同时调度请求，设置任务超时或在不再需要响应时取消它们。

`FutureRequestExecutionService`使用继承`FutureTask`的`HttpRequestFutureTask`封装请求。该类允许你在取消任务的同时并跟踪各种指标，如请求持续时间。

#### 7.3.1 创建`FutureRequestExecutionService`

`futureRequestExecutionService`的构造函数接受任何现有的httpClient实例和一个ExecutorService实例。配置两者时，最重要的是将最大连接数与你要使用的线程数设置一致。当线程数多于连接数时，由于没有可用的连接，连接可能会开始超时。如果连接数比线程多，`futureRequestExecutionService`将不会全部使用它们。

```java
HttpClient httpClient = HttpClientBuilder.create().setMaxConnPerRoute(5).build();
ExecutorService executorService = Executors.newFixedThreadPool(5);
FutureRequestExecutionService futureRequestExecutionService =
    new FutureRequestExecutionService(httpClient, executorService);
```

#### 7.3.2 安排请求

要安排请求，只需提供一个`HttpUriRequest`，`HttpContext`和一个`ResponseHandler`。由于请求由执行服务处理，因此必须使用`ResponseHandler`。

```java
private final class OkidokiHandler implements ResponseHandler<Boolean> {
    public Boolean handleResponse(
            final HttpResponse response) throws ClientProtocolException, IOException {
        return response.getStatusLine().getStatusCode() == 200;
    }
}

HttpRequestFutureTask<Boolean> task = futureRequestExecutionService.execute(
    new HttpGet("http://www.google.com"), HttpClientContext.create(),
    new OkidokiHandler());
// blocks until the request complete and then returns true if you can connect to Google
boolean ok=task.get();
```

#### 7.3.3 取消任务

计划任务可能会被取消。如果任务尚未执行，仅仅排队等待执行，它将永远不会执行。如果正在执行并且`mayInterruptIfRunning`参数设置为true，则会在请求上调用`abort()`；否则将被忽略响应，但请求将被允许正常完成。对`task.get()`的任何后续调用都将失败并出现`IllegalStateException`。应该注意的是取消任务只是释放客户端资源。该请求实际上可以在服务器端正常处理。

```java
task.cancel(true)
task.get() // throws an Exception
```

#### 7.3.4 回调

除了手动调用`task.get()`之外，还可以使用`FutureCallback`实例在请求完成时获取回调。这与`HttpAsyncClient`中使用的接口相同。

```java
private final class MyCallback implements FutureCallback<Boolean> {

    public void failed(final Exception ex) {
        // do something
    }

    public void completed(final Boolean result) {
        // do something
    }

    public void cancelled() {
        // do something
    }
}

HttpRequestFutureTask<Boolean> task = futureRequestExecutionService.execute(
    new HttpGet("http://www.google.com"), HttpClientContext.create(),
    new OkidokiHandler(), new MyCallback());
```

#### 7.3.5 指标
`FutureRequestExecutionService`通常用于执行大量Web服务调用的应用程序。为了便于例如监控或配置调优，`FutureRequestExecutionService`会跟踪几个指标。

每个`HttpRequestFutureTask`都提供了获取任务计划，开始和结束时间的方法。此外还有，请求和任务持续时间。这些指标汇总在`FutureRequestExecutionMetrics`实例中的`FutureRequestExecutionService`中，该实例可通过`FutureRequestExecutionService.metrics()`进行访问。

```java
task.scheduledTime() // returns the timestamp the task was scheduled
task.startedTime() // returns the timestamp when the task was started
task.endedTime() // returns the timestamp when the task was done executing
task.requestDuration // returns the duration of the http request
task.taskDuration // returns the duration of the task from the moment it was scheduled

FutureRequestExecutionMetrics metrics = futureRequestExecutionService.metrics()
metrics.getActiveConnectionCount() // currently active connections
metrics.getScheduledConnectionCount(); // currently scheduled connections
metrics.getSuccessfulConnectionCount(); // total number of successful requests
metrics.getSuccessfulConnectionAverageDuration(); // average request duration
metrics.getFailedConnectionCount(); // total number of failed tasks
metrics.getFailedConnectionAverageDuration(); // average duration of failed tasks
metrics.getTaskCount(); // total number of tasks scheduled
metrics.getRequestCount(); // total number of requests
metrics.getRequestAverageDuration(); // average request duration
metrics.getTaskAverageDuration(); // average task duration
```