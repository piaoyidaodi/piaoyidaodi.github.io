---
layout: post
title: "Apache HttpClient III"
categories: Log4j2
tag: Java-Libs
---
> `Apache HttpClient 4.5.5 Tutorial - III`

## 3. HTTP状态管理

最初，HTTP被设计为一种无状态的，面向请求/响应的协议，它没有为跨越多个逻辑相关的请求/响应交换的有状态会话做特殊规定。随着HTTP协议越来越流行，并且越来越多的系统开始将它用于从来没有打算用于的应用程序，例如作为电子商务应用程序的传输。因此，对状态管理的支持成为必然。

那时，网景通信是一家领先的网络客户端和服务器软件开发商，它们在其产品基于专有规范的基础上实现了对HTTP状态管理的支持。后来，Netscape试图通过发布规范草案来标准化该机制。这些努力有助于通过RFC标准定义的正式规范。但是，大量应用程序中的状态管理仍主要基于Netscape草案，但与官方规范不兼容。所有Web浏览器的主要开发人员都被迫保持与这些应用程序的兼容性，这些应用程序极大地促使了现有标准的碎片化。

### 3.1 HTTP cookies

HTTP cookie是HTTP代理与目标服务器之间可以交换，以保持会话的token或短包状态信息。Netscape的工程师曾经把它称为一个“魔术饼干”。

HttpClient使用Cookie接口来表示一个抽象的cookie token。最简单形式的HTTP cookie只是一个键值对。通常，HTTP cookie还包含许多属性，例如有效的域，指定应用此cookie的源服务器上URL的子集的路径以及Cookie有效的最长时间。

SetCookie接口表示，由原始服务器发送给HTTP代理以维持会话状态的Set-Cookie响应头。

ClientCookie接口通过额外的客户端特定功能扩展了Cookie接口，例如能够完全按照原始服务器指定的方式检索原始Cookie属性。这对于生成Cookie头很重要，因为某些Cookie规范要求只有在Set-Cookie头中指定Cookie头时，Cookie头才应包含某些属性。

以下是创建客户端cookie对象的示例：

```java
BasicClientCookie cookie = new BasicClientCookie("name", "value");
// Set effective domain and path attributes
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
// Set attributes exactly as sent by the server
cookie.setAttribute(ClientCookie.PATH_ATTR, "/");
cookie.setAttribute(ClientCookie.DOMAIN_ATTR, ".mycompany.com");
```

### 3.2 Cookie规范

CookieSpec接口代表cookie管理规范。cookie管理规范会强制执行：

- 解析Set-Cookie头的规则。

- 解析的cookie的验证规则。

- 给定主机，端口和原始路径的Cookie头格式。

HttpClient附带几个CookieSpec实现：

- **标准严格（Standard strict）**：符合RFC6265第4节定义的良好行为配置文件的语法和语义的状态管理策略。

- **标准（Standard）**：符合RFC6265第4节定义的更宽松的配置文件的状态管理策略，该配置文件旨在与不符合良好行为的配置文件的现有服务器进行互操作。

- **Netscape草案（过时）**：此政策符合Netscape Communications发布的原始草案规范。除非绝对有必要与遗留代码兼容，否则应该避免使用它。

- **RFC 2965（过时）**：符合RFC2965定义的过时状态管理规范的状态管理策略。请勿在新应用程序中使用。

- **RFC 2109（过时）**：符合RFC2109定义的过时状态管理规范的状态管理策略。请勿在新应用程序中使用。

- **浏览器兼容性（过时）**：此政策致力于模拟旧版浏览器应用程序的（错误）行为（如Microsoft Internet Explorer和Mozilla FireFox）。请不要在新的应用程序中使用。

- **默认（Default）**：默认cookie策略是一种综合性策略，根据与HTTP响应一起发送的cookie的属性（例如版本属性，现在已过时），选择符合RFC2965，RFC2109或Netscape草案的兼容实现。在下一个次要版本的HttpClient中，此策略将被弃用，以支持标准（RFC6265兼容）实现。

- **忽略Cookie（Ignore cookies）**：所有Cookie都被忽略。

强烈建议在新应用程序中使用Standard或Standard strict策略。过时的规范只能用于与旧系统的兼容性。对于过时的规范的支持将在下一个主要版本的HttpClient中被删除。

### 3.3 选择cookies策略

如果需要，可以在HTTP客户端上设置Cookie策略并在HTTP请求级别上覆盖Cookie策略。

```java
RequestConfig globalConfig = RequestConfig.custom().setCookieSpec(CookieSpecs.DEFAULT).build();
CloseableHttpClient httpclient = HttpClients.custom().setDefaultRequestConfig(globalConfig).build();
RequestConfig localConfig = RequestConfig.copy(globalConfig).setCookieSpec(CookieSpecs.STANDARD_STRICT).build();
HttpGet httpGet = new HttpGet("/");
httpGet.setConfig(localConfig);
```

### 3.4 自定义cookies策略

为了实现自定义Cookie策略，应该创建一个`CookieSpec`接口的自定义实现，创建一个`CookieSpecProvider`实现来创建和初始化自定义规范的实例，并使用HttpClient注册工厂。自定义规范注册后，可以按照与标准Cookie规范相同的方式激活。

```java
PublicSuffixMatcher publicSuffixMatcher = PublicSuffixMatcherLoader.getDefault();

Registry<CookieSpecProvider> r = RegistryBuilder.<CookieSpecProvider>create()
        .register(CookieSpecs.DEFAULT,
                new DefaultCookieSpecProvider(publicSuffixMatcher))
        .register(CookieSpecs.STANDARD,
                new RFC6265CookieSpecProvider(publicSuffixMatcher))
        .register("easy", new EasySpecProvider())
        .build();

RequestConfig requestConfig = RequestConfig.custom()
        .setCookieSpec("easy")
        .build();

CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultCookieSpecRegistry(r)
        .setDefaultRequestConfig(requestConfig)
        .build();
```

### 3.5 Cookie持久化

HttpClient可以使用实现`CookieStore`接口的任何持久性cookie存储的物理表示。名为`BasicCookieStore`的默认`CookieStore`实现是一个由java.util.ArrayList支持的简单实现。存储在`BasicClientCookie`对象中的cookie在容器对象被垃圾收集时会丢失。用户可以根据需要提供更复杂的实现。

```java
// Create a local instance of cookie store
CookieStore cookieStore = new BasicCookieStore();
// Populate cookies if needed
BasicClientCookie cookie = new BasicClientCookie("name", "value");
cookie.setDomain(".mycompany.com");
cookie.setPath("/");
cookieStore.addCookie(cookie);
// Set the store
CloseableHttpClient httpclient = HttpClients.custom()
        .setDefaultCookieStore(cookieStore)
        .build();
```

### 3.6 HTTP状态管理和执行上下文

在HTTP请求执行过程中，HttpClient将以下与状态管理相关的对象添加到执行上下文中：

- `Lookup`实例代表实际的cookie规范注册表。在本地上下文中设置的该属性的值优先于默认值。

- 表示实际Cookie规范的`CookieSpec`实例。

- `CookieOrigin`实例代表原始服务器的实际细节。

- 代表实际cookie存储的`CookieStore`实例。在本地上下文中设置的该属性的值优先于默认值。

本地`HttpContext`对象可用于在执行请求之前自定义HTTP状态管理上下文，或者在请求执行后检查其状态。也可以使用单独的执行上下文来实现每个用户（或每个线程）的状态管理。在本地上下文中定义的cookie规范注册表和cookie存储将优先于在HTTP客户端级别中设置的默认规则。

```java
CloseableHttpClient httpclient = <...>

Lookup<CookieSpecProvider> cookieSpecReg = <...>
CookieStore cookieStore = <...>

HttpClientContext context = HttpClientContext.create();
context.setCookieSpecRegistry(cookieSpecReg);
context.setCookieStore(cookieStore);
HttpGet httpget = new HttpGet("http://somehost/");
CloseableHttpResponse response1 = httpclient.execute(httpget, context);
<...>
// Cookie origin details
CookieOrigin cookieOrigin = context.getCookieOrigin();
// Cookie spec used
CookieSpec cookieSpec = context.getCookieSpec();
```

## 4. HTTP认证
HttpClient完全支持由HTTP标准规范定义的认证方案以及许多广泛使用的非标准认证方案，如NTLM和SPNEGO。

### 4.1 用户凭据

任何用户身份验证过程都需要一组可用于建立用户身份的凭证。用最简单的形式，用户凭证可以只是一个用户名/密码对。`UsernamePasswordCredentials`表示由安全主体和明文形式的密码组成的一组凭据。这个实现对于由HTTP标准规范定义的标准认证方案是足够的。

```java
UsernamePasswordCredentials creds = new UsernamePasswordCredentials("user", "pwd");
System.out.println(creds.getUserPrincipal().getName());
System.out.println(creds.getPassword());
```

标准输出为`user, pwd`。

`NTCredentials`是一个特定于Microsoft Windows的实现，除了用户名/密码对之外，还包括一组额外的Windows特定属性，例如用户域的名称。在Microsoft Windows网络中，同一用户可以属于多个域，每个域都有一组不同的授权。

```java
NTCredentials creds = new NTCredentials("user", "pwd", "workstation", "domain");
System.out.println(creds.getUserPrincipal().getName());
System.out.println(creds.getPassword());
```

标准输出为`DOMAIN/user, pwd`。

### 4.2 认证方案

`AuthScheme`接口表示抽象的面向质询-响应的认证方案。该认证方案将支持以下功能：

- 解析并处理目标服务器发送的质询，以响应受保护资源的请求。

- 提供处理后的质询的属性：认证方案类型及其参数，例如此认证方案适用的领域（如果可用）。

- 针对给定的凭证集和HTTP请求生成授权字符串，以响应实际的授权质询。

请注意，身份验证方案可能有状态并涉及一系列质询-响应交换。

HttpClient附带有几个AuthScheme实现：

- **基本（Basic）**：RFC2617中定义的基本认证方案。该认证方案不安全，因为凭证以明文形式传输。尽管不安全基本身份验证方案，但如果与TLS/SSL加密结合使用，则完全可以满足要求。

- **摘要（Digest）**：摘要式身份验证方案在RFC2617中定义。摘要式身份验证方案比Basic更安全，对于那些不希望通过TLS/SSL加密实现完全传输安全性开销的应用程序来说，它可能是一个不错的选择。

- **NTLM**：NTLM是Microsoft开发的专用身份验证方案，针对Windows平台进行了优化。据信NTLM比Digest更安全。

- **SPNEGO**：SPNEGO（Simple and Protected GSSAPI Negotiation Mechanism）是一种GSSAPI“伪机制”，用于协商许多可能的实际机制之一。SPNEGO最明显的用途是在Microsoft的HTTP Negotiate认证扩展中。可协商的子机制包括Active Directory支持的NTLM和Kerberos。目前HttpClient只支持Kerberos子机制。

- **Kerberos**：Kerberos身份验证实现。

### 4.3 凭证提供程序

凭证提供程序旨在维护一组用户凭证并能够为特定的认证范围生成用户凭证。身份验证范围由主机名，端口号，领域名称和身份验证方案名称组成。当向凭证提供程序注册凭证时，可以提供通配符（任何主机，任何端口，任何领域，任何方案）而不是具体的属性值。如果无法找到直接匹配，凭证提供程序将希望能够找到特定范围的最接近的匹配项。

HttpClient可以处理实现了`CredentialsProvider`接口的凭证提供程序的任何物理表示。名为`BasicCredentialsProvider`的默认`CredentialsProvider`实现是一个由java.util.HashMap支持的简单实现。

```java
CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(
    new AuthScope("somehost", AuthScope.ANY_PORT), 
    new UsernamePasswordCredentials("u1", "p1"));
credsProvider.setCredentials(
    new AuthScope("somehost", 8080), 
    new UsernamePasswordCredentials("u2", "p2"));
credsProvider.setCredentials(
    new AuthScope("otherhost", 8080, AuthScope.ANY_REALM, "ntlm"), 
    new UsernamePasswordCredentials("u3", "p3"));

System.out.println(credsProvider.getCredentials(
    new AuthScope("somehost", 80, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("somehost", 8080, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("otherhost", 8080, "realm", "basic")));
System.out.println(credsProvider.getCredentials(
    new AuthScope("otherhost", 8080, null, "ntlm")));
```

标准输出为：`[principal: u1][principal: u2]null[principal: u3]`。

### 4.4 HTTP认证和执行上下文

HttpClient依靠`AuthState`类来跟踪有关认证过程状态的详细信息。HttpClient在HTTP请求执行过程中创建两个`AuthState`实例：一个用于目标主机身份验证，另一个用于代理身份验证。如果目标服务器或代理需要用户身份验证，则相应的`AuthScope`实例将使用在验证过程中使用的`AuthScope`，`AuthScheme`和`Crednetials`。可以检查`AuthState`以便找出请求的认证类型，是否找到匹配的`AuthScheme`实现以及凭证提供程序是否设法找到给定认证范围的用户凭证。

在HTTP请求执行过程中，HttpClient将以下与认证相关的对象添加到执行上下文中：

- 表示实际认证方案注册表的`Lookup`实例。在本地上下文中设置的该属性的值优先于默认值。

- `CredentialsProvider`实例表示实际的凭证提供程序。在本地上下文中设置的该属性的值优先于默认值。

- 表示实际目标验证状态的`AuthState`实例。在本地上下文中设置的该属性的值优先于默认值。

- 表示实际代理验证状态的`AuthState`实例。在本地上下文中设置的该属性的值优先于默认值。

- 代表实际认证数据缓存的`AuthCache`实例。在本地上下文中设置的该属性的值优先于默认值。

本地`HttpContext`对象可用于在请求执行之前自定义HTTP认证上下文，或者在请求执行后检查其状态：

```java
CloseableHttpClient httpclient = <...>

CredentialsProvider credsProvider = <...>
Lookup<AuthSchemeProvider> authRegistry = <...>
AuthCache authCache = <...>

HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);
context.setAuthSchemeRegistry(authRegistry);
context.setAuthCache(authCache);
HttpGet httpget = new HttpGet("http://somehost/");
CloseableHttpResponse response1 = httpclient.execute(httpget, context);
<...>

AuthState proxyAuthState = context.getProxyAuthState();
System.out.println("Proxy auth state: " + proxyAuthState.getState());
System.out.println("Proxy auth scheme: " + proxyAuthState.getAuthScheme());
System.out.println("Proxy auth credentials: " + proxyAuthState.getCredentials());
AuthState targetAuthState = context.getTargetAuthState();
System.out.println("Target auth state: " + targetAuthState.getState());
System.out.println("Target auth scheme: " + targetAuthState.getAuthScheme());
System.out.println("Target auth credentials: " + targetAuthState.getCredentials());
```

### 4.5 认证数据的缓存

从版本4.1开始，HttpClient会自动缓存已成功验证的主机的信息。请注意，必须使用相同的执行上下文执行相关的逻辑请求，以便将缓存的身份验证数据从一个请求传播到另一个请求。只要执行环境超出范围，身份验证数据就会丢失。

### 4.6 抢先认证

HttpClient不支持抢先认证，因为如果误用或使用不当，抢先认证可能会导致严重的安全问题，例如以明文形式向未授权的第三方发送用户凭证。因此，希望用户在特定应用环境的环境中评估抢先认证与安全风险的潜在优劣。

尽管如此，可以通过预填充认证数据缓存来配置HttpClient进行预先认证。

```java
CloseableHttpClient httpclient = <...>

HttpHost targetHost = new HttpHost("localhost", 80, "http");
CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(
        new AuthScope(targetHost.getHostName(), targetHost.getPort()),
        new UsernamePasswordCredentials("username", "password"));

// Create AuthCache instance
AuthCache authCache = new BasicAuthCache();
// Generate BASIC scheme object and add it to the local auth cache
BasicScheme basicAuth = new BasicScheme();
authCache.put(targetHost, basicAuth);

// Add AuthCache to the execution context
HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);
context.setAuthCache(authCache);

HttpGet httpget = new HttpGet("/");
for (int i = 0; i < 3; i++) {
    CloseableHttpResponse response = httpclient.execute(
            targetHost, httpget, context);
    try {
        HttpEntity entity = response.getEntity();

    } finally {
        response.close();
    }
}
```

### 4.7 NTLM身份验证

从版本4.1起，HttpClient提供对NTLMv1，NTLMv2和NTLM2 Session身份验证的全面支持。人们仍然可以继续使用外部NTLM引擎，例如由Samba项目开发的JCIFS库，作为其Windows互操作性套件程序的一部分。

#### 4.7.1 NTLM连接持久性

与标准的Basic和Digest方案相比，NTLM认证方案在计算开销和性能影响方面明显更昂贵。这可能是微软选择使NTLM身份验证方案处于有状态的主要原因之一。也就是说，一旦通过身份验证，用户身份就会与该连接的整个使用期限相关联。NTLM连接的有状态性使得连接持久性更加复杂，显然这是因为持久性NTLM连接可能不会被具有不同用户身份的用户重新使用。HttpClient附带的标准连接管理器完全能够管理有状态连接。但是，同一会话中逻辑相关的请求使用相同的执行上下文至关重要，这样才能使他们知道当前的用户身份。否则，HttpClient将最终针对每个NTLM保护资源的HTTP请求创建一个新的HTTP连接。有关有状态HTTP连接的详细讨论，请参阅本节。

由于NTLM连接是有状态的，因此通常建议使用相对低廉的方法（如GET或HEAD）触发NTLM身份验证，并重用相同的连接执行更昂贵的方法，特别是那些包含请求实体（如POST或PUT）的方法。

```java
CloseableHttpClient httpclient = <...>

CredentialsProvider credsProvider = new BasicCredentialsProvider();
credsProvider.setCredentials(AuthScope.ANY,
        new NTCredentials("user", "pwd", "myworkstation", "microsoft.com"));

HttpHost target = new HttpHost("www.microsoft.com", 80, "http");

// Make sure the same context is used to execute logically related requests
HttpClientContext context = HttpClientContext.create();
context.setCredentialsProvider(credsProvider);

// Execute a cheap method first. This will trigger NTLM authentication
HttpGet httpget = new HttpGet("/ntlm-protected/info");
CloseableHttpResponse response1 = httpclient.execute(target, httpget, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}

// Execute an expensive method next reusing the same context (and connection)
HttpPost httppost = new HttpPost("/ntlm-protected/form");
httppost.setEntity(new StringEntity("lots and lots of data"));
CloseableHttpResponse response2 = httpclient.execute(target, httppost, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

### 4.8 SPNEGO/Kerberos认证

SPNEGO（Simple and Protected GSSAPI Negotiation Mechanism）旨在允许当两端都不确定另一端人可以使用/提供什么服务时对服务进行身份验证。最常用于Kerberos身份验证。它可以包装其他机制，但HttpClient中的当前版本仅考虑Kerberos。

- 客户端Web浏览器为资源执行HTTP GET。

- Web服务器返回HTTP401状态和头：`WWW-Authenticate：Negotiate`。

- 客户端生成一个使用base64编码`NegTokenInit`，并使用`Authorization`头重新提交GET：`Authorization：Negotiate <base64 encoding>`。

- 服务器解码`NegTokenInit`，提取支持的`MechTypes`（在我们的例子中只有Kerberos V5），确保它是预期的之一，然后提取`MechToken`（Kerberos令牌）并对其进行验证。如果需要更多处理，则另一个HTTP401将返回到客户端，并在`WWW-Authenticate`头中包含更多数据。客户端获取信息并生成另一个token，将其传回`Authorization`头直至完成。

- 当客户端通过身份验证时，Web服务器应返回HTTP200状态，最终的`WWW-Authenticate`头和页面内容。

#### 4.8.1 HTTPClient中支持的SPNEGO

SPNEGO身份验证方案与Java 1.5及更高版本兼容。但强烈建议使用Java1.6及其以上，因为它更完全支持SPNEGO身份验证。

Sun JRE提供支持类来执行几乎所有的Kerberos和SPNEGO token处理。这意味着很多设置都是针对GSS类的。`SPNegoScheme`是一个简单的类来处理token的编组和读写正确的头文件。

最好的开始方法是获取示例中KerberosHttpClient.java文件，并尝试使其运行。有可能会有很多问题，但如果幸运的话，它会工作，没有太多的问题。它还应该提供一些输出来进行调试。

在Windows中，它应该默认使用已登录的凭据；这可以通过使用`kinit`来覆盖，例如`$JAVA_HOME\bin\kinit testuser@AD.EXAMPLE.NET`，这对测试和调试问题非常有帮助。删除由kinit创建的缓存文件以恢复到Windows Kerberos缓存。

确保在krb5.conf文件中列出domain_realms。这是问题的主要来源。

#### 4.8.2 建立GSS/Java Kerberos

本文档假定您使用Windows，但大部分信息也适用于Unix。

`org.ietf.jgss`类有很多可能的配置参数，主要在`krb5.conf/krb5.ini`文件中。有关格式的更多信息，请参阅`http://web.mit.edu/kerberos/krb5-1.4/krb5-1.4.1/doc/krb5-admin/krb5.conf.html`。

#### 4.8.3 `login.conf`文件

以下配置是在Windows XP中针对IIS和JBoss Negotiation模块的基本设置。

系统属性`java.security.auth.login.config`可用于指向`login.conf`文件。

`login.conf`内容可能如下所示：

```java
com.sun.security.jgss.login {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};

com.sun.security.jgss.initiate {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};

com.sun.security.jgss.accept {
  com.sun.security.auth.module.Krb5LoginModule required client=TRUE useTicketCache=true;
};
```

#### 4.8.4 `krb5.conf/krb5.ini`文件

如果未指定，将使用系统默认值。通过将系统属性`java.security.krb5.conf`指向一个自定义的krb5.conf文件来覆盖（如果需要）。

krb5.conf内容可能如下所示：

```text
[libdefaults]
    default_realm = AD.EXAMPLE.NET
    udp_preference_limit = 1
[realms]
    AD.EXAMPLE.NET = {
        kdc = KDC.AD.EXAMPLE.NET
    }
[domain_realms]
.ad.example.net=AD.EXAMPLE.NET
ad.example.net=AD.EXAMPLE.NET
```

#### 4.8.5 Windows特定配置

要允许Windows使用当前用户，必须将系统属性`javax.security.auth.useSubjectCredsOnly`设置为false，并且应该添加并正确设置Windows注册表键`allowtgtsessionkey`，以允许在Kerberos Ticket-Granting Ticket中发送会话密钥。

在Windows Server 2003和Windows 2000 SP4上，以下是所需的注册表设置：

```text
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\Parameters
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```

在Windows XP SP2上，以下是所需的注册表设置：

```text
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa\Kerberos\
Value Name: allowtgtsessionkey
Value Type: REG_DWORD
Value: 0x01
```