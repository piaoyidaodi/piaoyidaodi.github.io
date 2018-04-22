---
layout: post
title: "Apache HttpClient II"
categories: HttpClient
tag: Java-Libs
---
> `Apache HttpClient 4.5.5 Tutorial - II`

## 2. 连接管理

### 2.1 连接持久性

建立从一个主机到另一个主机的连接的过程非常复杂，并且涉及两个端点之间的多个分组交换，这可能非常耗时。连接的握手开销可能很大，特别是对于小型HTTP消息。如果打开的连接可以重新用于执行多个请求，则可以实现更高的数据吞吐量。

HTTP/1.1规定HTTP连接可以重复用于每个默认的多个请求。符合HTTP/1.0的端点也可以使用一种机制来显式传达它们的首选项以保持连接的活动状态并将其用于多个请求。如果后续请求需要连接到同一个目标主机，则HTTP代理也可以保持空闲连接在一段时间内保持活动状态。保持连接活动的能力通常被称为连接持久性。HttpClient完全支持连接持久性。

### 2.2 HTTP连接路由

HttpClient能够直接或通过可能涉及多个中间连接（也称为中继）的路由建立到目标主机的连接。HttpClient将路由的连接区分为普通（plain），隧道（tunneled）和分层（layered）。使用多个中间代理来隧道连接到目标主机被称为代理链接。

通过连接目标或第一个也是唯一的代理来建立普通路由。隧道路由通过连接到第一个隧道并通过代理链对目标进行隧道建立。没有代理的路由不能被隧道传输。分层路由通过对现有连接上分层协议来建立。协议只能在通向目标的隧道上进行分层，或者通过不通过代理的直接连接进行分层。

#### 2.2.1 路线计算
`RouteInfo`接口用于表示，包含一个或多个中间步骤或跳跃的到目标主机的确定路由的信息。`HttpRoute`是`RouteInfo`的具体实现，它不能被改变（是不可变的）。`HttpTracker`是一个可变的`RouteInfo`实现，由HttpClient在内部用来跟踪到最终路由目标的剩余跳转。在向路由目标成功执行下一跳之后，可以更新`HttpTracker`。`HttpRouteDirector`是一个辅助类，可用于计算路由中的下一步。这个类由HttpClient在内部使用。

`HttpRoutePlanner`是一个接口，表示根据执行上下文计算到给定目标的完整路由策略。HttpClient提供了两个默认的`HttpRoutePlanner`实现。`SystemDefaultRoutePlanner`基于java.net.ProxySelector。默认情况下，它将从系统属性或运行应用程序的浏览器中获取JVM的代理设置。`DefaultProxyRoutePlanner`实现不使用任何Java系统属性，也不使用任何系统或浏览器代理设置。它总是通过相同的默认代理来计算路由。

#### 2.2.2 Secure HTTP连接
如果在两个连接端点之间传输的信息不能被未经授权的第三方读取或篡改，则HTTP连接可以被认为是安全的。SSL/TLS协议是确保HTTP传输安全性的最广泛使用的技术。但是，也可以采用其他加密技术。通常，HTTP传输通过SSL/TLS加密连接。

### 2.3 HTTP连接管理器

#### 2.3.1 管理连接和连接管理器

HTTP连接是复杂的，有状态的，线程不安全的对象，需要妥善管理才能正常工作。HTTP连接一次只能由一个执行线程使用。HttpClient使用一个特殊的实体来管理对HTTP连接的访问​​，称为HTTP连接管理器，并由接口`HttpClientConnectionManager`表示。HTTP连接管理器的目的是，提供新的HTTP连接的工厂，管理持久连接的生命周期，并同步对持久连接的访问​​，以确保一次只有一个线程可以访问连接。内部HTTP连接管理器与`ManagedHttpClientConnection`实例一起工作，作为管理连接状态和控制I/O操作执行的真实连接的代理。如果托管连接被释放或被其消费者明确关闭，则底层连接将从其代理中分离出来并返回给管理器。即使服务使用者仍然持有对代理实例的引用，它不能再有意或无意地执行任何I/O操作或更改真实连接的状态。

这是从连接管理器获取连接的示例：

```java
HttpClientContext context = HttpClientContext.create();
HttpClientConnectionManager connMrg = new BasicHttpClientConnectionManager();
HttpRoute route = new HttpRoute(new HttpHost("localhost", 80));
// Request new connection. This can be a long process
ConnectionRequest connRequest = connMrg.requestConnection(route, null);
// Wait for connection up to 10 sec
HttpClientConnection conn = connRequest.get(10, TimeUnit.SECONDS);
try {
    // If not open
    if (!conn.isOpen()) {
        // establish connection based on its route info
        connMrg.connect(conn, route, 1000, context);
        // and mark it as route complete
        connMrg.routeComplete(conn, route, context);
    }
    // Do useful things with the connection.
} finally {
    connMrg.releaseConnection(conn, null, 1, TimeUnit.MINUTES);
}
```

如果需要，可以通过调用`ConnectionRequest#cancel()`来过早终止连接请求。这将解除在`ConnectionRequest#get()`方法中阻塞的线程。

#### 2.3.2 简单连接管理

`BasicHttpClientConnectionManager`是一个简单的连接管理器，一次只能维护一个连接。即使这个类是线程安全的，它也应该只被一个执行线程使用。`BasicHttpClientConnectionManager`将努力用相同的路由为后续请求复用连接。但是，如果持久化的连接路由与连接请求的路由不匹配，它将关闭现有连接，并重新打开给定路由。如果连接已被分配，则抛出java.lang.IllegalStateException。该连接管理器实现应该在EJB容器中使用。

#### 2.3.3 连接池管理

`PoolingHttpClientConnectionManager`是一个更复杂的实现，用于管理客户端连接池并能够处理来自多个执行线程的连接请求。连接以每个路由为基础进行汇总。通过从池中租用而不是创建全新的连接，可以为在管理器池中具有可用连接的路由请求提供服务。

`PoolingHttpClientConnectionManager`保持每个路由和合计连接的最大限制。默认情况下，此实现将为每个给定路由创建不超过2个并发连接，总共不会有20个连接。对于许多真实的应用程序来说，这些限制可能会过于严格，特别是如果他们使用HTTP作为其服务的传输协议。

此示例显示连接池参数如何调整：

```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
// Increase max total connection to 200
cm.setMaxTotal(200);
// Increase default max connection per route to 20
cm.setDefaultMaxPerRoute(20);
// Increase max connections for localhost:80 to 50
HttpHost localhost = new HttpHost("locahost", 80);
cm.setMaxPerRoute(new HttpRoute(localhost), 50);

CloseableHttpClient httpClient = HttpClients.custom().setConnectionManager(cm).build();
```

#### 2.3.4 关闭连接管理器

当不再需要HttpClient实例并且即将离开作用域时，关闭其连接管理器以确保管理器中所有保持活动状态的连接都关闭，并释放由这些连接分配的系统资源非常重要。

```java
CloseableHttpClient httpClient = <...>
httpClient.close();
```

### 2.4 多线程请求异常

当使用`PoolingClientConnectionManager`等连接池化的连接管理器时，HttpClient可使用多个执行线程同时执行多个请求。

`PoolingClientConnectionManager`将根据其配置分配连接。如果给定路由的所有连接都已租用，则连接请求将被阻塞，直到有连接释放回池。可以通过将`'http.conn-manager.timeout'`设置为正值来确保连接管理器不会无限期地阻塞连接请求操作。如果连接请求在给定时间内无法被处理，则会抛出`ConnectionPoolTimeoutException`。

```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(cm)
        .build();

// URIs to perform GETs on
String[] urisToGet = {
    "http://www.domain1.com/",
    "http://www.domain2.com/",
    "http://www.domain3.com/",
    "http://www.domain4.com/"
};

// create a thread for each URI
GetThread[] threads = new GetThread[urisToGet.length];
for (int i = 0; i < threads.length; i++) {
    HttpGet httpget = new HttpGet(urisToGet[i]);
    threads[i] = new GetThread(httpClient, httpget);
}

// start the threads
for (int j = 0; j < threads.length; j++) {
    threads[j].start();
}

// join the threads
for (int j = 0; j < threads.length; j++) {
    threads[j].join();
}
```

尽管HttpClient实例是线程安全的并且可以在多个执行线程之间共享，但强烈建议每个线程维护自己的专用HttpContext实例。

```java
static class GetThread extends Thread {

    private final CloseableHttpClient httpClient;
    private final HttpContext context;
    private final HttpGet httpget;

    public GetThread(CloseableHttpClient httpClient, HttpGet httpget) {
        this.httpClient = httpClient;
        this.context = HttpClientContext.create();
        this.httpget = httpget;
    }

    @Override
    public void run() {
        try {
            CloseableHttpResponse response = httpClient.execute(
                    httpget, context);
            try {
                HttpEntity entity = response.getEntity();
            } finally {
                response.close();
            }
        } catch (ClientProtocolException ex) {
            // Handle protocol errors
        } catch (IOException ex) {
            // Handle I/O errors
        }
    }
}
```

### 2.5 连接回收策略

经典阻塞I/O模型的一个主要缺点是网络套接字只能在I/O操作发生阻塞时才对I/O事件做出反应。当连接释放回管理器时，它可以保持活动状态，但它无法监视套接字的状态并对任何I/O事件做出反应。如果连接在服务器端关闭，则客户端连接无法检测连接状态的更改（并通过关闭套接字来适当地作出反应）。

HttpClient试图通过在使用连接执行HTTP请求之前，测试连接是否是陈旧来缓解这个问题，即如果它在服务器端被关闭，则不再有效。过时的连接检查不是100%可靠的。对于空闲连接，每个套接字模型都不夹杂单一线程的唯一可行解决方案是使用专用监视器线程，用于回收由于长时间不活动而被认为已过期的连接。监视线程可以周期性调用`ClientConnectionManager#closeExpiredConnections()`方法关闭所有过期的连接并从池中回收关闭的连接。它还可以选择调用`ClientConnectionManager#closeIdleConnections()`方法关闭在给定时间段内闲置的所有连接。

```java
public static class IdleConnectionMonitorThread extends Thread {

    private final HttpClientConnectionManager connMgr;
    private volatile boolean shutdown;

    public IdleConnectionMonitorThread(HttpClientConnectionManager connMgr) {
        super();
        this.connMgr = connMgr;
    }

    @Override
    public void run() {
        try {
            while (!shutdown) {
                synchronized (this) {
                    wait(5000);
                    // Close expired connections
                    connMgr.closeExpiredConnections();
                    // Optionally, close connections
                    // that have been idle longer than 30 sec
                    connMgr.closeIdleConnections(30, TimeUnit.SECONDS);
                }
            }
        } catch (InterruptedException ex) {
            // terminate
        }
    }
    public void shutdown() {
        shutdown = true;
        synchronized (this) {
            notifyAll();
        }
    }

}
```

### 2.6 连接Keep-Alive策略

HTTP规范没有规定持久性连接可能会持续多久，并且应该保持活动状态。一些HTTP服务器使用非标准的Keep-Alive头部来向客户端传达他们希望在服务器端保持连接活跃的时间段（以秒为单位）。如果可用的话，HttpClient使用这些信息。如果Keep-Alive头部不存在于响应中，则HttpClient假定连接可以无限期地保持活动状态。但是，通常使用的许多HTTP服务器被配置为在一段时间不活动之后删除持性久连接，以便节约系统资源，但通常不会通知客户端。如果默认策略过于乐观，则可能需要提供自定义Keep-Alive策略。

```java
ConnectionKeepAliveStrategy myStrategy = new ConnectionKeepAliveStrategy() {

    public long getKeepAliveDuration(HttpResponse response, HttpContext context) {
        // Honor 'keep-alive' header
        HeaderElementIterator it = new BasicHeaderElementIterator(
                response.headerIterator(HTTP.CONN_KEEP_ALIVE));
        while (it.hasNext()) {
            HeaderElement he = it.nextElement();
            String param = he.getName();
            String value = he.getValue();
            if (value != null && param.equalsIgnoreCase("timeout")) {
                try {
                    return Long.parseLong(value) * 1000;
                } catch(NumberFormatException ignore) {
                }
            }
        }
        HttpHost target = (HttpHost) context.getAttribute(
                HttpClientContext.HTTP_TARGET_HOST);
        if ("www.naughty-server.com".equalsIgnoreCase(target.getHostName())) {
            // Keep alive for 5 seconds only
            return 5 * 1000;
        } else {
            // otherwise keep alive for 30 seconds
            return 30 * 1000;
        }
    }

};
CloseableHttpClient client = HttpClients.custom().setKeepAliveStrategy(myStrategy).build();
```

### 2.7 连接socket工厂

HTTP连接在内部使用java.net.Socket对象来处理通过线路传输的数据。但是，它们依赖于`ConnectionSocketFactory`接口来创建，初始化和连接套接字。这使得HttpClient的用户可以在运行时提供特定于应用程序的套接字初始化代码。`PlainConnectionSocketFactory`是创建和初始化普通（未加密）套接字的默认工厂。

创建套接字和将其连接到主机的过程是分离的，这样套接字在连接操作中被阻塞时可以关闭。

```java
HttpClientContext clientContext = HttpClientContext.create();
PlainConnectionSocketFactory sf = PlainConnectionSocketFactory.getSocketFactory();
Socket socket = sf.createSocket(clientContext);
int timeout = 1000; //ms
HttpHost target = new HttpHost("localhost");
InetSocketAddress remoteAddress = new InetSocketAddress(InetAddress.getByAddress(new byte[] {127,0,0,1}), 80);
sf.connectSocket(timeout, socket, target, remoteAddress, null, clientContext);
```

#### 2.7.1 SSL

`LayeredConnectionSocketFactory`是`ConnectionSocketFactory`接口的扩展。分层套接字工厂能够在现有平面套接字上的创建分层套接字。套接字层主要用于通过代理创建安全套接字。HttpClient附带实现SSL/TLS层的`SSLSocketFactory`。请注意HttpClient不使用任何自定义加密功能。它完全依赖于标准的Java加密（JCE）和安全套接字（JSEE）扩展。

#### 2.7.2 与连接管理器集成

自定义连接套接字工厂可以与特定协议方案（如HTTP或HTTPS）相关联，然后用于创建自定义连接管理器。

```java
ConnectionSocketFactory plainsf = <...>
LayeredConnectionSocketFactory sslsf = <...>
Registry<ConnectionSocketFactory> r = RegistryBuilder.<ConnectionSocketFactory>create().register("http", plainsf).register("https", sslsf).build();

HttpClientConnectionManager cm = new PoolingHttpClientConnectionManager(r);
HttpClients.custom().setConnectionManager(cm).build();
```

#### 2.7.3 自定义SSL/TLS

HttpClient利用`SSLConnectionSocketFactory`创建SSL连接。`SSLConnectionSocketFactory`允许高度的自定义。它可以将`javax.net.ssl.SSLContext`的实例作为参数，并使用它创建自定义配置的SSL连接。

```java
KeyStore myTrustStore = <...>
SSLContext sslContext = SSLContexts.custom().loadTrustMaterial(myTrustStore).build();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext);
```

`SSLConnectionSocketFactory`的定制意味着对SSL/TLS协议的概念有一定程度的熟悉，其详细解释超出了本文的范围。有关javax.net.ssl.SSLContext和相关工具的详细说明，请参阅Java安全套接字扩展（JSSE）参考指南。

#### 2.7.4 主机名验证

除了在SSL/TLS协议级别上执行信任验证和客户端身份验证之外，HttpClient还可以选择验证目标主机名是否与存储在服务器X.509证书内的名称匹配（一旦建立连接）。此验证可以提供服务器认证材料真实性的附加保证。`javax.net.ssl.HostnameVerifier`接口表示主机名验证策略。HttpClient附带两个`javax.net.ssl.HostnameVerifier`实现。重要提示：主机名验证不应与SSL信任验证混淆。

- `DefaultHostnameVerifier`：HttpClient使用的默认实现符合RFC2818。主机名必须与证书中指定的任何替代名称匹配，或者在没有给出替代名称的情况下颁发证书主体所指定的CN。可以在CN和任何主体加入通配符中。

- `NoopH​​ostnameVerifier`：这个主机名验证程序实质上关闭了主机名验证。它接受任何有效的SSL会话并匹配目标主机。

默认HttpClient使用`DefaultHostnameVerifier`实现。如果需要，可以指定一个不同的主机名验证器实现:

```java
SSLContext sslContext = SSLContexts.createSystemDefault();
SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);
```

从版本4.4开始，HttpClient使用由Mozilla Foundation维护的公共后缀列表，以确保SSL证书中的通配符不会被滥用于具有公共顶级域的多个域。HttpClient附带发布时检索到的列表的副本。该列表的最新版本可以在`https://publicsuffix.org/list/`找到。非常建议保存本地副本清单，并从其原始位置每天下载一次。

```java
PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader.load(PublicSuffixMatcher.class.getResource("my-copy-effective_tld_names.dat"));
DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(publicSuffixMatcher);
```

可以使用null匹配器以禁用对公共客户列表的验证：`DefaultHostnameVerifier hostnameVerifier = new DefaultHostnameVerifier(null);`

### 2.8 HttpClient代理设置

即使HttpClient知道复杂的路由方案和代理链，但它仅支持简单的直接或单跳代理连接。

告诉HttpClient通过代理连接到目标主机的最简单方法是设置默认代理参数：

```java
HttpHost proxy = new HttpHost("someproxy", 8080);
DefaultProxyRoutePlanner routePlanner = new DefaultProxyRoutePlanner(proxy);
CloseableHttpClient httpclient = HttpClients.custom().setRoutePlanner(routePlanner).build();
```

也可以使HttpClient使用标准JRE代理选择器来获取代理信息：

```java
SystemDefaultRoutePlanner routePlanner = new SystemDefaultRoutePlanner(
        ProxySelector.getDefault());
CloseableHttpClient httpclient = HttpClients.custom().setRoutePlanner(routePlanner).build();
```

或者，可以提供自定义的`RoutePlanner`实现，以完全控制HTTP路由计算的过程：

```java
HttpRoutePlanner routePlanner = new HttpRoutePlanner() {

    public HttpRoute determineRoute(
            HttpHost target,
            HttpRequest request,
            HttpContext context) throws HttpException {
        return new HttpRoute(target, null,  new HttpHost("someproxy", 8080),"https".equalsIgnoreCase(target.getSchemeName()));
    }
};
CloseableHttpClient httpclient = HttpClients.custom().setRoutePlanner(routePlanner).build();
}
```