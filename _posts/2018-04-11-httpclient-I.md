---
layout: post
title: "Apache HttpClient I"
categories: Log4j2
tag: Java-Libs
---
> `Apache HttpClient 4.5.5 Tutorial - I`

## 前言

### 1. HttpClient的范围

- 基于HTTPCore的Client部分的HTTP传输库。
- 基于传统的阻塞IO。
- 内容不可知。

### 2. HttpClient不包括什么

HttpClient不是一个浏览器，是一个客户端Http传输库。HttpClient的目的是发送和接收HTTP消息。HttpClient不会处理内容，执行HTML中嵌入的javascript，猜测content type，如果不明确设置，也不会重格式化request、重写URI的位置，或其他与HTTP传输无关的功能。

## 1. 基本原理

### 1.1 Request execution

HttpClient最重要的功能是执行HTTP方法。执行一个HTTP方法会涉及到一个或多个HTTP请求和HTTP响应的交换，通这常由HttpClient在内部处理。用户需要提供一个请求对象来执行，HttpClient需要将请求传送给目标服务器返回相应的响应对象，或者如果执行失败则抛出异常。

很自然地，HttpClient API的主要入口点是定义上述内容的HttpClient接口。

下面是一个最简单的请求执行过程示例：

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    <...>
} finally {
    response.close();
}
```

#### 1.1.1 Http请求

所有HTTP请求都有一个请求行，其中包含方法名称，请求URI和HTTP协议版本。

HttpClient支持HTTP/1.1规范中定义的所有HTTP方法：`GET，HEAD，POST，PUT，DELETE，TRACE和OPTIONS`。每种方法类型都有一个特定的类：`HttpGet，HttpHead，HttpPost，HttpPut，HttpDelete，HttpTrace和HttpOptions`。

Request-URI是一个统一资源标识符，用于标识应用请求的资源。HTTP请求URI由协议，主机名，可选端口，资源路径，可选查询和可选片段组成。如：

```java
HttpGet httpget = new HttpGet("http://www.google.com/search?hl=en&q=httpclient&btnG=Google+Search&aq=f&oq=");
```

HttpClient提供了一个URIBuider工具类以简化请求URI的创建和更改：

```java
URI uri = new URIBuilder()
        .setScheme("http")
        .setHost("www.google.com")
        .setPath("/search")
        .setParameter("q", "httpclient")
        .setParameter("btnG", "Google Search")
        .setParameter("aq", "f")
        .setParameter("oq", "")
        .build();
HttpGet httpget = new HttpGet(uri);
System.out.println(httpget.getURI());
```

#### 1.1.2 Http响应

HTTP响应是服务器收到并解释请求消息后发回给客户端的消息。该消息的第一行由协议版本、数字状态代码及其相关的文本短语组成。如`HTTP/1.1 200 OK`。

#### 1.1.3 处理消息头

HTTP消息可以包含许多描述消息属性的消息头，例如内容长度，内容类型等。 HttpClient提供了检索，添加，删除和枚举消息头的方法。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

输出为：

```text
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
2
```

最有效地获取所有给定类型的消息头的方法是使用HeaderIterator接口。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

输出为：

```text
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
```

还提供了方便的方法将HTTP消息解析为单独的头元素。

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", 
"c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", 
"c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(response.headerIterator("Set-Cookie"));

while (it.hasNext()) {
    HeaderElement elem = it.nextElement(); 
    System.out.println(elem.getName() + " = " + elem.getValue());
    NameValuePair[] params = elem.getParameters();
    for (int i = 0; i < params.length; i++) {
        System.out.println(" " + params[i]);
    }
}
```

输出为：

```text
c1 = a
path=/
domain=localhost
c2 = b
path=/
c3 = c
domain=localhost
```

#### 1.1.4 HTTP实体

HTTP消息可以携带与请求或响应相关联的内容实体。实体可以在一些请求和某些响应中找到，因为它们是可选的。使用实体的请求被称为实体封装请求。HTTP规范定义了两个实体封装请求方法：POST和PUT。响应通常被认为会包含内容实体。此规则有例外情况，例如对HEAD方法的响应和204 No Content，304 Not Modified，205 Reset Content的响应。

HttpClient根据内容的来源区分了三种实体：

- 流式传输（streamed）：内容是从流接收的，或者是即时生成的。特别是，这个类别包括从HTTP响应收到的实体。流式实体通常不可重复。

- 自包含（self-contained）：内容在内存中或通过独立于连接或其他实体的方式获得。独立实体通常是可重复的。这种类型的实体将主要用于包含HTTP请求的实体。

- 包装（wrapping）：内容是从另一个实体获得的。

当从HTTP响应中流出内容时，这种区别对connection管理很重要。对于由应用程序创建并仅使用HttpClient发送的请求实体，流式和自包含之间的区别并不重要。在这种情况下，建议将不可重复的实体视为流式传输，并将那些可重复的实体视为自包含。

**Repeatable实体**

一个实体可以是可重复的，这意味着它的内容可以被多次读取。这只适用于自包含的实体（如ByteArrayEntity或StringEntity）。

**使用HTTP实体**

由于实体可以表示二进制和字符内容，因此它支持字符编码。

在执行带有封装内容的请求时或请求成功并且响应主体用于将结果发送回客户端时将创建HTTP实体。

要从HTTP实体读取内容，可以通过`HttpEntity#getContent()`方法检索输入流，该方法返回java.io.InputStream，或者可以向`HttpEntity#writeTo(OutputStream)`方法提供输出流，在所有的内容被写入给定的流后将返回。

当实体接收到传入消息时，可以使用方法`HttpEntity#getContentType()`和`HttpEntity#getContentLength()`方法读取公共metadata，如Content-Type和Content-Length消息头（如果它们可用）。由于Content-Type消息头可以包含MIME类型（如text/plain或text/html）的字符编码，可使用`HttpEntity#getContentEncoding()`方法读取此信息。如果消息头不可用，则返回-1的长度，对于内容类型则返回NULL。如果Content-Type头部可用，则会返回一个Header对象。

为外发消息创建实体时，此元数据必须由实体的创建者提供。

```java
StringEntity myEntity = new StringEntity("important message", ContentType.create("text/plain", "UTF-8"));

System.out.println(myEntity.getContentType());
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

输出为：

```text
Content-Type: text/plain; charset=utf-8
17
important message
17
```

#### 1.1.5 确保底层你资源的释放

为了确保正确释放系统资源，必须关闭与实体相关的内容流或响应本身，示例如下：

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        try {
            // do something useful
        } finally {
            instream.close();
        }
    }
} finally {
    response.close();
}
```

关闭内容流和关闭响应的区别在于前者会尝试通过使用实体内容来保持底层连接的活动状态，而后者会立即关闭并丢弃连接。

请注意，使用`HttpEntity#writeTo(OutputStream)`方法将实体完全写出后，也需要确保正确释放系统资源。如果此方法通过调用`HttpEntity#getContent()`来获取`java.io.InputStream`的实例，则还需要在finally子句中关闭该流。

当使用流式实体时，可以使用`EntityUtils#consume(HttpEntity)`方法确保实体内容已被完全使用，并且底层流已关闭。

然而，可能会出现只需要检索整个响应内容的一小部分的情况，且使用剩余内容并使连接可重用的性能损失太高，在这种情况下，可以通过关闭来终止内容流响应。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        int byteOne = instream.read();
        int byteTwo = instream.read();
        // Do not need the rest
    }
} finally {
    response.close();
}
```

连接将不会被重用，但它所拥有的所有层次的资源将被正确释放。

#### 1.1.6 使用实体内容

推荐使用实体内容的方法是使用其`HttpEntity#getContent()`或`HttpEntity#writeTo(OutputStream)`方法。 HttpClient也附带了EntityUtils类，该类提供了几种静态方法来更容易地从实体读取内容或信息。 不用直接读取`java.io.InputStream`，通过使用此类中的方法就可以检索整个内容体中的字符串或字节数组。但是，**强烈建议不要使用EntityUtils，除非响应实体来自受信任的HTTP服务器，并且知道其长度大小**。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        long len = entity.getContentLength();
        if (len != -1 && len < 2048) {
            System.out.println(EntityUtils.toString(entity));
        } else {
            // Stream content out
        }
    }
} finally {
    response.close();
}
```

在某些情况下，可能需要不止一次使用实体内容。在这种情况下，实体内容必须以某种方式进行缓冲，无论是在内存中还是在磁盘上。最简单的方法是使用`BufferedHttpEntity`类包装原始实体。这将导致原始实体的内容被读入内存缓冲区。在所有其他方面，实体包装将具有原始包装。

```java
CloseableHttpResponse response = <...>
HttpEntity entity = response.getEntity();
if (entity != null) {
    entity = new BufferedHttpEntity(entity);
}
```

#### 1.1.7 生成实体内容

HttpClient提供了几个类，可以用来通过HTTP连接有效地流出内容。这些类的实例可以与封装实体请求（如POST和PUT等）关联，以便将实体内容封装到传出的HTTP请求中。HttpClient为大多数常用数据容器（如字符串，字节数组，输入流和文件）提供了几个类：StringEntity，ByteArrayEntity，InputStreamEntity和FileEntity。

```java
File file = new File("somefile.txt");
FileEntity entity = new FileEntity(file,
ContentType.create("text/plain", "UTF-8"));
HttpPost httppost = new HttpPost("http://localhost/action.do");
httppost.setEntity(entity);
```

请注意InputStreamEntity非repeatable的，因为它只能从底层数据流读取一次。通常建议实现一个自定义的自包含HttpEntity类，而不是使用通用的InputStreamEntity，比如FileEntity。

**HTML表单**

例如，许多应用程序需要模拟提交HTML表单的过程，以便登录到Web应用程序或提交输入数据。HttpClient提供实体类UrlEncodedFormEntity来支持该过程。

```java
List<NameValuePair> formparams = new ArrayList<NameValuePair>();
formparams.add(new BasicNameValuePair("param1", "value1"));
formparams.add(new BasicNameValuePair("param2", "value2"));
UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, Consts.UTF_8);
HttpPost httppost = new HttpPost("http://localhost/handler.do");
httppost.setEntity(entity);
```

UrlEncodedFormEntity实例将使用所谓的URL编码来对参数进行编码并生成以下内容：`param1=value1&param2=value2`。

**内容分块**

通常建议让HttpClient根据正在传输的HTTP消息的属性选择最合适的传输编码。 但是，可以通过将`HttpEntity#setChunked()`设置为true来通知HttpClient，首选分块编码。请注意，HttpClient将仅使用此标志作为线索。当使用不支持块编码的HTTP协议版本（例如HTTP/1.0）时，此值将被忽略。

```java
StringEntity entity = new StringEntity("important message",
        ContentType.create("plain/text", Consts.UTF_8));
entity.setChunked(true);
HttpPost httppost = new HttpPost("http://localhost/acrtion.do");
httppost.setEntity(entity);
```

#### 1.1.8 响应操纵器

处理响应的最简单和最方便的方法是使用ResponseHandler接口，该接口包含`handleResponse(HttpResponse response)`方法。这种方法完全免除了用户担心连接管理问题。当使用ResponseHandler时，无论请求执行成功还是导致异常，HttpClient都会自动确保连接释放回连接管理器。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/json");

ResponseHandler<MyJsonObject> rh = new ResponseHandler<MyJsonObject>() {

    @Override
    public JsonObject handleResponse(
            final HttpResponse response) throws IOException {
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
        Gson gson = new GsonBuilder().create();
        ContentType contentType = ContentType.getOrDefault(entity);
        Charset charset = contentType.getCharset();
        Reader reader = new InputStreamReader(entity.getContent(), charset);
        return gson.fromJson(reader, MyJsonObject.class);
    }
};
MyJsonObject myjson = client.execute(httpget, rh);
```

### 1.2 HttpClient接口

HttpClient接口提供了执行HTTP请求的最基本协定。它不会对请求执行过程施加任何限制或具体细节，并将连接管理，状态管理，认证和重定向处理的细节进行独立实现。这应该可以更容易地使用附加功能来装饰接口，例如响应内容缓存。

通常，HttpClient实现充当许多负责处理HTTP协议特定方面的特殊用途处理程序或策略接口实现的外观，例如重定向或身份验证处理或决定连接持久性和保持活动持续时间。 这使用户可以选择性地使用定制的，特定于应用程序的方式替换这些方面的默认实现。

```java
ConnectionKeepAliveStrategy keepAliveStrat = new DefaultConnectionKeepAliveStrategy() {

    @Override
    public long getKeepAliveDuration(
            HttpResponse response,
            HttpContext context) {
        long keepAlive = super.getKeepAliveDuration(response, context);
        if (keepAlive == -1) {
            // Keep connections alive 5 seconds if a keep-alive value
            // has not be explicitly set by the server
            keepAlive = 5000;
        }
        return keepAlive;
    }

};
CloseableHttpClient httpclient = HttpClients.custom().setKeepAliveStrategy(keepAliveStrat).build();
```

#### 1.2.1 HttpClient线程安全

HttpClient实现被认为是**线程安全的**。建议将此类的同一个实例重用于多个请求执行。

#### 1.2.2 HTTPClient资源释放

当不再需要使用CloseableHttpClient实例并且即将离开作用域时，必须通过调用`CloseableHttpClient#close()`方法关闭与其关联的连接管理器。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
try {
    <...>
} finally {
    httpclient.close();
}
```

### 1.3 HTTP execution context

最初，HTTP被设计为无状态，面向响应-请求的协议。然而，真实世界的应用程序通常需要能够通过几个逻辑相关的请求-响应交换来持久化状态信息。为了使应用程序能够保持处理状态，HttpClient允许HTTP请求在特定的执行上下文（称为HTTP context）内执行。如果在连续的请求之间重复使用相同的上下文，则多个逻辑相关的请求可以参与逻辑会话。HTTP上下文的功能类似于java.util.Map <String，Object>。它只是一个任意命名值的集合。应用程序可以在请求执行之前填充上下文属性，或者在执行完成后检查上下文。

HttpContext可以包含任意对象，因此可能在多个线程之间不安全的共享。建议每个执行线程维护自己的上下文。

在执行HTTP请求的过程中，HttpClient将以下属性添加到执行上下文中：

- 表示到目标服务器的实际连接的`HttpConnection`实例。

- 表示连接目标的`HttpHost`实例。

- 表示完整连接路由的`HttpRoute`实例

- 表示实际HTTP请求的`HttpRequest`实例。执行上下文中的最终HttpRequest对象始终表示精确的消息的状态，因为它被发送到了目标服务器。默认HTTP/1.0和HTTP/1.1使用相对请求URI。但是，如果请求是通过代理以非隧道模式发送的，那么URI将是绝对的。

- 表示实际HTTP响应的`HttpResponse`实例。

- 表示指示实际请求是否已完全传输到连接目标的标志的java.lang.Boolean对象。

- 表示实际请求配置的`RequestConfig`对象。

- 表示请求执行过程中收到的所有重定向位置集合的`java.util.List <URI>`对象。

可以使用HttpClientContext适配器类来简化与上下文状态的交互。

```java
HttpContext context = <...>
HttpClientContext clientContext = HttpClientContext.adapt(context);
HttpHost target = clientContext.getTargetHost();
HttpRequest request = clientContext.getRequest();
HttpResponse response = clientContext.getResponse();
RequestConfig config = clientContext.getRequestConfig();
```

表示逻辑相关会话的多个请求序列应该使用相同的HttpContext实例执行，以确保请求之间的对话上下文和状态信息的自动传播。

在以下示例中，由初始请求设置的请求配置将保留在执行上下文中并传播到共享相同上下文的后续请求。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
RequestConfig requestConfig = RequestConfig.custom().setSocketTimeout(1000).setConnectTimeout(1000).build();

HttpGet httpget1 = new HttpGet("http://localhost/1");
httpget1.setConfig(requestConfig);
CloseableHttpResponse response1 = httpclient.execute(httpget1, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
HttpGet httpget2 = new HttpGet("http://localhost/2");
CloseableHttpResponse response2 = httpclient.execute(httpget2, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

### 1.4 HTTP协议拦截器

HTTP协议拦截器是实现HTTP协议特定方面的例程。通常，协议拦截器需要操作传入消息的一个特定header或一组相关header，或者用一个特定header或一组相关header填充外发消息。协议拦截器也可以操纵封装消息的内容实体——透明内容压缩/解压缩就是一个很好的例子。通常这是通过使用装饰器模式来实现的，其中使用包装器实体类来装饰原始实体。多个协议拦截器可以组合成一个逻辑单元。

协议拦截器可以通过共享信息（如处理状态）和HTTP执行上下文进行协作。协议拦截器可以使用HTTP上下文为一个请求或多个连续的请求存储处理状态。

通常拦截器的执行顺序应该不重要，只要它们不依赖于执行上下文的特定状态。如果协议拦截器具有相互依赖性并因此必须按特定顺序执行，则应将它们以预期执行顺序添加到协议处理器中。

协议拦截器必须实现为线程安全的。与servlet类似，协议拦截器不应使用实例变量，除非同步访问这些变量。

这是如何使用本地上下文在连续请求之间保持处理状态的例子：

```java
CloseableHttpClient httpclient = HttpClients.custom()
        .addInterceptorLast(new HttpRequestInterceptor() {

            public void process(
                    final HttpRequest request,
                    final HttpContext context) throws HttpException, IOException {
                AtomicInteger count = (AtomicInteger) context.getAttribute("count");
                request.addHeader("Count", Integer.toString(count.getAndIncrement()));
            }

        })
        .build();

AtomicInteger count = new AtomicInteger(1);
HttpClientContext localContext = HttpClientContext.create();
localContext.setAttribute("count", count);

HttpGet httpget = new HttpGet("http://localhost/");
for (int i = 0; i < 10; i++) {
    CloseableHttpResponse response = httpclient.execute(httpget, localContext);
    try {
        HttpEntity entity = response.getEntity();
    } finally {
        response.close();
    }
}
```

### 1.5 异常处理

HTTP协议处理器可能会抛出两种类型的异常：`java.io.IOException`，如果发生I/O故障（例如socket超时或socket重置）以及代表HTTP失败（例如违反HTTP协议）的`HttpException`。 通常，**I/O错误被认为是非致命的且可恢复的，而HTTP协议错误被认为是致命的并且不能自动从中恢复**。请注意，HttpClient实现将HttpExceptions重新抛出为ClientProtocolException，它是java.io.IOException的子类。这使得HttpClient的用户可以从单个catch子句处理I/O错误和协议违规。

#### 1.5.1 HTTP传输安全

HTTP协议并不适用于所有类型的应用程序非常重要。HTTP是一种简单的面向请求/响应的协议，最初设计用于支持检索静态或动态生成的内容。它从未打算支持事务操作。例如，如果HTTP服务器成功接收和处理请求，生成响应并将状态码发送回客户端，则HTTP服务器将认为它的合同部分已满足。如果客户端由于读取超时，请求取消或系统崩溃而无法完整接收响应，服务器将不会尝试回退事务。如果客户决定重试相同的请求，服务器将不可避免地不止一次地执行相同的事务。在某些情况下，这可能会导致应用程序数据损坏或应用程序状态不一致。

即使HTTP从未被设计为支持事务处理，但它仍然可以用作满足特定条件的关键任务应用程序的传输协议。为确保HTTP传输层的安全性，系统必须确保应用层上HTTP方法的幂等性。

#### 1.5.2 幂等方法

HTTP/1.1规范定义了一个幂等方法：方法也可以具有“幂等性”的性质，除了错误或期满问题外，N>0个相同请求的副作用与单个请求相同。

换句话说，应用程序应该确保它准备好处理多次执行相同方法的影响。这可以通过例如提供唯一的事务ID以及通过避免执行相同的逻辑操作的其他方式来实现。

请注意，这个问题不是特定于HttpClient的。基于浏览器的应用程序也受到与HTTP方法非幂等性相同的问题的影响。

默认情况下，HttpClient假定只有非实体封装方法（如GET和HEAD）是幂等的，而实体封闭方法（如POST和PUT）是不兼容的。

#### 1.5.3 自动异常恢复

默认情况下，HttpClient会尝试从I/O异常中自动恢复。默认的自动恢复机制仅限于少数已知安全的异常：

- HttpClient不会尝试从任何逻辑或HTTP协议错误（从HttpException类派生的错误）中恢复。

- HttpClient会自动重试那些被认为是幂等的方法。

- 当HTTP请求仍然正被传送到目标服务器时（即请求没有完全传送到服务器），HttpClient会自动重试那些传送异常失败的方法。

#### 1.5.4 请求重试处理

为了启用自定义异常恢复机制，应该提供HttpRequestRetryHandler接口的实现。

```java
HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {

    public boolean retryRequest(
            IOException exception,
            int executionCount,
            HttpContext context) {
        if (executionCount >= 5) {
            // Do not retry if over max retry count
            return false;
        }
        if (exception instanceof InterruptedIOException) {
            // Timeout
            return false;
        }
        if (exception instanceof UnknownHostException) {
            // Unknown host
            return false;
        }
        if (exception instanceof ConnectTimeoutException) {
            // Connection refused
            return false;
        }
        if (exception instanceof SSLException) {
            // SSL handshake exception
            return false;
        }
        HttpClientContext clientContext = HttpClientContext.adapt(context);
        HttpRequest request = clientContext.getRequest();
        boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
        if (idempotent) {
            // Retry if the request is considered idempotent
            return true;
        }
        return false;
    }

};
CloseableHttpClient httpclient = HttpClients.custom().setRetryHandler(myRetryHandler).build();
```

### 1.6 中止请求

在某些情况下，由于目标服务器上的高负载或客户端发出的太多并发请求，HTTP请求无法在预期的时间范围内完成。在这种情况下，可能需要提前终止请求并解除阻塞在I/O操作中的执行线程。通过调用`HttpUriRequest#abort()`方法，HttpClient执行的HTTP请求可以在任何执行阶段中止。此方法是线程安全的，可以从任何线程调用。当一个HTTP请求中止时，它的执行线程（即使当前在I/O操作中被阻塞）将确保通过抛出一个InterruptedIOException来解除阻塞。

### 1.7 重定向处理

HttpClient会自动处理所有类型的重定向，除了那些被HTTP规范明确禁止的要求用户干预的重定向。请参阅其他（状态码303）在POST和PUT上重定向，将根据HTTP规范的要求将请求转换为GET请求。可以使用自定义重定向策略来放宽由HTTP规范施加的对POST方法的自动重定向的限制。

```java
LaxRedirectStrategy redirectStrategy = new LaxRedirectStrategy();
CloseableHttpClient httpclient = HttpClients.custom().setRedirectStrategy(redirectStrategy).build();
```

HttpClient通常不得不在其执行过程中重写请求消息。默认的HTTP/1.0和HTTP/1.1通常使用相对请求URI。同样，原始请求可能会多次从一个位置重定向到另一个位置。最终解释的绝对HTTP位置可以使用原始请求和上下文来构建。`URIUtils#resolve`方法可用于构建用于生成最终请求的绝对URI。此方法包含来自重定向请求或原始请求的最后一个片段标识符。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    HttpHost target = context.getTargetHost();
    List<URI> redirectLocations = context.getRedirectLocations();
    URI location = URIUtils.resolve(httpget.getURI(), target, redirectLocations);
    System.out.println("Final HTTP location: " + location.toASCIIString());
    // Expected to be an absolute URI
} finally {
    response.close();
}
```