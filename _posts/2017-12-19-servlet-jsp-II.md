---
layout: post
title: "Head First Servlets and JSP -- II"
categories: Servlet & JSP
tag: Java-Web
---
> `Servlet & JSP`基础，Web应用的属性和监听者、会话状态等。

### 5. Web应用

#### 5.1 配置文件示例

  ``` xml
<servlet>
  <init-param>
    <param-name>DriftEmail</param-name>
    <param-value>piaoyidaodi@gmail.com</param-value>
  </init-param>
</servlet>
<context-param>
  <param-name>GeGeEmail</param-name>
  <param-value>piaoyidaodi@gmail.com</param-value>
</context-param>
<listener>
  <listener-class>com.example.MyServletContextListener</listener-class>
</listener>
  ```
  ``` java
    // ServletConfig
    out.println(getServletConfig().getInitParameter("DriftEmail"));
    // ServletContext
    out.println(getServletContext().getInitParameter("DriftEmail"));
  ```

#### 5.2 ServletConfig

1. 容器初始化一个servlet时，会为这个servlet建一个**唯一的`ServletConfig`**，容器从配置描述文件**读出servlet初始化参数**，并把这些参数交给`ServletConfig`对象，然后把`ServletConfig`对象传递给servlet的`init()`方法。

2. servlet初始化参数只在`init()`时调用一次：
- servlet读取部署描述文件，包括servlet初始化参数；
- 容器为servlet创建一个ServletConfig实例；
- 容器为servlet初始化参数创建String键值对；
- 容器向ServletConfig提供键值对初始化参数的引用；
- 容器创建servlet类的一个实例；
- 容器调用servlet的`init()`方法，传入`ServletConfig`引用。

3. `ServletConfig`主要用于提供初始化参数。

4. 通过设置请求对象属性`request.setAttribute("styles",result)`，得到请求的任何其他servlet或接收转发请求的JSP都可以使用这些属性。

#### 5.3 ServletContext

1. 应用中**所有的`servlet`和`JSP`**都自动能访问上下文初始化参数。

2. 在分布式环境中，每个JVM都有一个`ServletContext`。

3. `ServletContext`的作用范围更大，最常见的用途可能是存储数据库查找名。`ServletContext`是`JSP`或`servlet`与容器及Web应用其他部分的一个连接。

4. 可以把`ServletConfig`和`ServletContext`初始化参数认为是部署时常量，运行时只能获得不能设置。

5. `ServletContext`的获取：
- 直接调用`getServletContext()`方法。
- `ServletConfig`拥有该servlet的`ServletContext`引用，只有在servlet类没有扩展**`GenericServlet`及其子类**时，才会使用这种方式。

#### 5.4 `ServletContextListener`监听器

1. 监听器唯一的用途就是**初始化应用**。通过`web.xml`部署监听器描述，容器通过检查类注意到监听器接口或多个接口，以此明确监听什么类型的事件。

2. `ServletContextListener`监听`ServletContext`最关键的两个事件：创建和撤销。监听器类

3. 要监听`ServletContext`事件，需要编写一个实现`ServletContextListener`的监听器类（实现其中的`contextInitialized()`和`contextDestroyed()`方法），把它放在`WEB-INF/classes`目录中，并在`web.xml`中放置`<listener>`元素告诉容器。

4. `ServletContext`的`getAttribute()`方法返回类型为`Object`，需要对结果强制类型转换。`getInitParameter()`方法返回值为`String`。

5. `ServletContextListener`的工作流程：
- 容器读取配置文件，包括`<listener>`和`<context-param>`元素。
- 容器为该Web应用创建一个`ServletContext`对象。
- 容器为每一个上下文初始化参数创建键值对，并将键值对的引用交给`ServletContext`对象。
- 容器创建一个`ServletContextListener`类实例。
- 容器调用监听者`contextInitialized()`方法，传入一个`ServletContextEvent`对象，通过该对象获取一个`ServletContext`引用，并得到上下文初始化参数。
- 容器初始化某个对象，并放入`ServletContext`请求属性中。

#### 5.5 常见监听器

1. `javax.servlet.ServletContextAttributeListener`
- **事件类型参数**：`ServletContextAttributeEvent`
- **作用**：一个Web应用上下文中是否增加、删除或替换了一个属性。

2. `javax.servlet.http.HttpSessionListener`
- **事件类型参数**：`HttpSessionEvent`
- **作用**：期望了解有多少个并发用户，即想跟踪活动的会话。

3. `javax.servlet.ServletRequestListener`
- **事件类型参数**：`ServletRequestEvent`
- **作用**：监听每一次请求，以便建立日志记录。

4. `javax.servlet.ServletRequestAttributeListener`
- **事件类型参数**：`ServletRequestAttributeEvent`
- **作用**：期望了解是否增加、删除或替换一个请求属性。

5. `javax.servlet.HttpSessionBindingListener`
- **事件类型参数**：`HttpSessionBindingEvent`
- **作用**：拥有一个属性类（该类的对象将被放在一个属性中），期望了解该类型的实例是否被绑定或删除在一个会话中。

6. `javax.servlet.HttpSessionAttributeListener`
- **事件类型参数**：`HttpSessionBindingEvent`
- **作用**：期望了解是否增加、删除或替换一个会话属性。

7. `javax.servlet.ServletContextListener`
- **事件类型参数**：`ServletContextEvent`
- **作用**：监听是否创建或撤销了一个上下文。

8. `javax.servlet.HttpSessionActivationListener`
- **事件类型参数**：`HttpSessionEvent`
- **作用**：拥有一个属性类，期望该类型实例所绑定的会话迁移到另一个JVM时得到通知。

#### 5.6 属性与参数

1. 属性：（`ServletContextAttributeListener, ServletRequestAttributeListener, HttpSessionAttributeListener`）
- **类型**：应用上下文、请求、会话。
- **设置方法**：`setAttribute(String name, Object value)`
- **返回类型**：`Object`
- **获取方法**：`getAttribute(String name)`

2. 参数：
- **类型**：应用上下文初始化参数、请求参数、Servlet初始化参数（无会话参数的说法）。
- **设置方法**：在配置文件中设置应用和Servlet初始化参数。
- **返回类型**：`String`
- **获取方法**：`getInitParameter(String name)`

3. 上下文属性**不是**线程安全的。
- 同步服务方法会防止同一个servlet中的其他线程访问上下文属性，但是不能阻止另外一个不同servlet的访问。
- **只有当处理同一个上下文属性的所有其他代码都对`ServletContext`同步时**，才可以保护上下文属性。

4. 如果同一个客户同时有多个请求，会话属性**不是**线程安全的。

5. `SingleThreadModel`（**废弃的API**），如果一个servlet实现了该接口，可以保证不会在该servlet的服务方法中并发的执行两个线程。实现`SingleThreadModel`和同步服务方法的效果是一样的，低效且无保护作用。

6. 只有**请求属性**和**局部变量**是线程安全的。servlet只有一个实例，但可以有多个线程。

#### 5.7 请求属性和请求分派

1. 控制器和模型通信，得到视图建立相应所需的数据，因为只针对这个请求，所以放在请求作用域中。接管请求使用`RequestDispatcher`。

2. `RequestDispatcher`有两个方法：`forward()`（常用）和`include()`。这两个方法都取**请求和响应对象**作为参数。
- `forward()`方法，将请求分派以后就不会在处理这个请求和响应。
- `include()`方法，在请求分派处理后，会继续处理请求和响应。

3. 获取`RequestDispatcher`的两个方法：**从`ServletRequest`得到，从`ServletContext`得到**。
- `RequestDispatcher dispatcher=request.getRequestDispatcher("result.jsp")`，以一个`String`作为路径参数，此参数**是否以`/`开头都可以**。
- `RequestDispatcher dispatcher=getServletContext.getRequestDispatcher("/result.jsp")`，以一个`String`作为路径参数，此参数**必须以`/`开头**。

4. 使用`OutputStream`的`flush()`方法，会导致响应发送到客户，响应完成。

#### 5.8 属性作用域总结

1. `Context`：
- **可访问性**：Web应用所有部分，包括servlet, JSP, ServletContextListener, ServletContextAttributeListener。
- **作用域**：应用的整个生命周期。随着服务器或应用的关闭而销毁。
- **适用于...**：希望整个应用共享的资源，如**数据库、email地址**等。

2. `HttpSession`：
- **可访问性**：访问这个特定会话的所有Servlet和JSP。需要注意，会话可能从单个客户请求扩展到跨同一个客户的多个请求，这些请求可能指向不同的servlet。（打开多个淘宝商品页面）
- **作用域**：会话的生命周期。会话可以通过编程撤销，也可以因为超时而撤销。
- **适用于...**：与客户会话有关的资源和数据，不单包括单个请求的资源。它要与客户完成一个持续的会话，如购物车。

3. `Request`：
- **可访问性**：应用中能直接访问请求对象的所有部分。简单来说主要是**`RequestDispatcher`将请求转发到的JSP和servlet，以及与请求相关的监听者**。
- **作用域**：请求的声明周期。持续到servlet的`service()`方法结束，就是线程处理该请求的整个生命周期。
- **适用于...**：将模型信息从控制器传送到视图，或特定于单个客户请求的任何其他数据。

### 6. 会话状态

#### 6.1 会话管理

1. `HttpSession`对象可以保存同一个客户**多个请求的会话状态**，即与一个特定客户的整个会话期间`HttpSession`会持久存储。对于会话期间客户的所有请求、得到的所有信息都可以用`HttpSession`对象保存。

2. **HTTP是无状态连接**，因此对于容器而言，每个请求都来自于一个新的客户。

3. 客户需要唯一的会话ID标示客户，最常用cookie交换会话ID。
- 对于客户的第一个请求，容器会生成一个唯一的会话ID，并通过响应返回给客户。
- 客户在以后的每个请求中发回该会话ID，容器看到后会进行匹配会话。

#### 6.2 会话获取

1. 新建和获取会话ID，使用`request.getSession()`。其中还包括**容器执行的**生成会话ID、创建新的cookie对象、把会话ID放入cookie、把cookie设置为响应的一部分等工作。

2. 获取会话的两种方法：
- 从`request`中；
- 从**会话事件对象**中。

#### 6.3 禁用cookie

1. 禁用cookie：客户会忽略`Set-Cookie`响应首部。

2. `URL`重写：
- `URL`重写能获取置于`cookie`中的会话ID，并把会话ID附加到**访问应用的各个`URL`后**。
- `URL`重写需要对响应中发送的所有`URL`进行显式编码，`response.encodeURL()`。

3. 当使用`getSession()`而没有获取到会话时，就需要建立一个新会话。由于不清楚`cookie`是否工作，所以使用`cookie`和`URL`重写两种做法。

4. 请求重定向的`URL`重写：`response.encodeRedirectURL()`。

5. `URL`重写的注意事项：
- 使用`URL`重写当且仅当**作为会话的一部分的所有页面都是动态生成的**。
- `URL`重写是自动的，前提是**通过响应对象完成对`URL`的编码**，`URL`编码只与响应有关。

6. 要点：
- 在写至`response`的HTML中，`URL`重写会把会话ID增加到其中**所有的URL**之后。
- 会话ID作为请求`URL`最后的额外信息再通过请求返回。
- 如果客户不接受`cookie`，`URL`重写会自动发生，但是必须显示对`URL`编码。
- 静态页面无法使用自动`URL`重写。

#### 6.4 会话的销毁

1. `HttpSession`方法：
- `getAttribute()`，`setAttribute()`，`removeAttribute()`，`getCreationTime()`，`getLastAccessedTime()`。
- `getMaxInactiveInterval()`：指定对于此会话客户请求的最大间隔时间（秒数）。
- `invalidate()`：结束会话，并解除所有绑定属性。

2. 会话死亡的三种方式：
- 超时。
- 在会话对象中使用`invalidate()`方法。
- 应用结束（崩溃或取消部署）。

3. 会话超时的设置方法：
- 在配置文件中使用`<web-app><session-config><session-timeout>`设置（单位为分）。
- 在会话中调用`session.setMaxInactiveInterval()`设置（单位为秒）。

4. `cookie`的作用：
- `cookie`实际就是客户和服务器之间交换的一小段键值对数据；服务器把`cookie`交给客户，客户再在请求中发回这个`cookie`。
- 默认`cookie`在客户离开浏览器后删除，也可以设置让`cookie`活的更长。
- 向响应增加一个`cookie`时，要传递一个**Cookie对象**。
- `HttpServletRequest`：`getCookies()`。
- `HttpServletResponse`：`addCookies()`，不存在`setCookies()`。

#### 6.5 会话的生命周期事件

1. 生命周期：（创建会话、撤销会话）
- 事件：`HttpSessionEvent`
- 监听器：`HttpSessionListener`

2. 属性：（增、删、改一个属性）
- 事件：`HttpSessionBindingEvent`
- 监听器：`HttpSessionAttributeListener`

3. 迁移：（会话准备钝化、会话已经激活）
- 事件：`HttpSessionEvent`
- 监听器：`HttpSessionActivationListener`

4. 会话属性：（会话属性类绑定、会话属性类解绑）
- 事件：`HttpSessionBindingEvent`
- 监听器：属性类实现`HttpSessionBindingListener`

5. 会话迁移（分布式多个VM）：
- 多个VM，只有`HttpSession`对象及其属性会从一个VM**移动到**另一个VM。
- 对于Web应用的一个给定的会话ID，不论应用分布在多少个VM上，**只有一个`HttpSession`对象**。