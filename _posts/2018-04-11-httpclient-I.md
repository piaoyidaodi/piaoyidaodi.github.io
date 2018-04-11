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