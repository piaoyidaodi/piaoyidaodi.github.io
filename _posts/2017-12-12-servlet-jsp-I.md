---
layout: post
title: "Head First Servlets and JSP -- I"
categories: Servlet&JSP
tag: Java-Web
---
> `Servlet & JSP`基础，概述、Web应用体系结构、MVC入门、Servlet的请求和响应等。

### 1. 前言与概述

#### 1.1 HTTP基础概念

1. TCP负责确保文件完整的达到目的地，IP负责路由数据到目的地。

2. HTML是HTTP响应的一部分，**HTTP是无状态连接**。

3. HTTP请求常用`GET`和`POST`方法，此外还包括不常用的方法如`HEAD、TRACE、PUT、DELETE、OPTION和CONNECT`方法。

4. `GET`：要求服务器获得一个资源并发回。
- `GET`总字符数有限；
- `GET`发送的数据追加在URL后，明文；
- 使用`POST`用户不能对一个表单提交建立书签。

5. `POST`：数据放在HTTP请求体内。

6. HTTP响应包括一个首部和一个体：
- 首部信息告诉浏览器使用了什么协议，请求是否成功，以及体中包括何种类型的内容。
- 体包含了让浏览器显示的具体内容

7. 端口16位，一个服务器最多可运行65536个不同的服务器应用，其中0-1023端口保留。
- Telnet：23
- FTP：21
- HTTP：80；HTTPS：443

#### 1.2 Web服务器

1. Web服务器只提供静态页面，不对页面进行处理；Web服务器将请求交给辅助应用，然后取得这个应用的响应，再把它发送给客户。

2. Web服务器无法：提供动态内容和在服务器保存数据：
- 动态内容：Web服务器与“辅助应用”通信并生成非静态的即时页面。
- 保存数据：Web服务器将请求发送给“辅助应用”，并由该应用生成一个响应。

3. CGI：
- CGI代表公共网关接口，使用Perl，C，Python和PHP编写CGI程序。

### 2. Web应用体系结构

#### 2.1 容器

1. 何为容器？
- Servlet没有`main()`方法，它们受控于一个Java应用，这个Java应用称为容器；
- Web服务器获得**指向servlet的请求**->Web服务器将请求交给**servlet的容器**->容器向servlet提供HTTP请求和响应，调用servlet的方法。

2. 容器提供的服务：
- 通信支持：利用容器提供的方法，可轻松让servlet与Web服务器通信。无需自己建立ServerSocket，监听端口，创建流等。
- 生命周期管理：容器控制着servlet的创建和销毁。如类加载、实例化、初始化、servlet方法调用、以及销毁后的垃圾回收。
- 多线程支持：容器会自动为它接收的每个servlet请求创建一个新的Java线程。
- 生命方式实现安全：利用容器可以使用XML部署描述文件来配置安全性，而不必将其硬编码进servlet类代码中。
- JSP支持。
 > 容器可以让你更专注于自己的业务逻辑，而不用考虑为线程管理，安全性和网络通信编写代码。将底层服务交由容器处理。

3. 容器处理请求简介：
- 用户点击一个指向一个servlet的URL;
- 容器“看出来”这个请求要的是一个servlet，所以容器创建两个对象：`HttpServletResponse`和`HttpServletRequest`；
- 容器根据请求中的URL找到对应的servlet，为该请求**创建或分配一个线程，并将请求和响应对象传递给这个servlet线程**；
- 容器调用servlet的`service()`方法，并根据请求的不同类型，`service()`方法会调用`doGet()`或`doPost()`方法，如假设请求为HTTP GET请求；
- `doGet()`方法生成动态页面，并把这个页面填入响应对象；
- 线程结束，容器把响应对象转化为HTTP响应，把它发回客户，然后删除请求和响应对象。

4. 代码分析：

  ```java
import javax.servlet.*;  
import javax.servlet.http.*;
import java.io.*;
public class Ch2Servlet extends HttpServlet {
    public void doGet(HttpServletRequest request,HttpServletResponse response)throws IOException {
        PrintWriter out = response.getWriter();
        java.util.Date today = new java.util.Date();
        out.println(“<html> “ +
                    “<body>” +
                    “<h1 style=”text-align:center>” +
                    “HF\’s Chapter2 Servlet</h1>” +
                    “<br>” + today +
                    “</body>” +
                    “</html>”);
    }
}
  ```
5. 注意事项：
- 实际中，**大多数servlet都会覆盖`doGet()`或`doPost()`方法**。
- **大多数servlet都是HttpServlet**。
- servlet可以获得容器创建的请求和响应的对象的引用request和response。
- servlet从response中可得到一个`PrintWriter`，使用这个PrintWriter将HTML文本输入到响应对象。

#### 2.2 部署文件（DD）

1. servlet名称映射：
- 文件路径名：servlet类存在服务器上的路径名，类路径名。
- 部署名：秘密内部名，可以是servlet类名或类文件相对路径或不相干名。
- 公共URL名：客户所知道的名字，写在HTML中，公共URL名放在HTTP请求中发送给服务器。

2. servlet名映射有助于改善应用的灵活性和安全性：
- 通过映射servlet名，而不是把真实文件和路径名写入代码，能提供很大的灵活性，轻松移动文件，不需要跟踪文件的移动。
- 通过映射servlet名，可以隐藏服务器的真实路径和目录结构，防止用户直接访问。

3. 部署描述文件将URL映射到servlet：
- 将servlet部署到Web容器时，会创建一个简单的XML文档，称为部署描述文件（DD），部署描述文件告诉容器如何运行servlet和JSP。
- 使用两个XML元素将URL映射到servlet：`<servlet><servlet-name><servlet-class>`内部名->**完全限定类名(无.class扩展名)**；`<servlet-mapping><servlet-name><url-pattern>`内部名->**公共URL名**。

4. 部署描述文件的其他作用：**安全角色、错误页面、标记库、初始配置信息**。
- 尽量少改动已经过测试的源代码；
- 即使没有源代码，也可以对应用的功能进行调整；
- 不用重新编译和测试任何代码，也可以让应用适用不同的资源（如数据库）；
- 可以更容易的维护动态安全信息，如访问控制列表和安全角色；
- 非程序员也可以修改部署Web应用。

#### 2.3 MVC（Model-View-Controller）

1. MVC的关键是，**业务逻辑要与表示分离**，而且要**在两者之间放上别的东西**，这样业务逻辑本身就能作为一个可重用的Java类存在。（业务逻辑可用于其他视图，如JavaSwing）

2. MVC就是把业务逻辑从servlet中抽出来，放入一个“模型”中，所谓**模型就是一个可重用的普通Java类**，模型是业务数据和处理该数据的方法的组合。

3. MVC
- `Model`：包含具体的业务逻辑和状态，即模型知道用什么规则来得到和更新状态，系统中只有此部分与数据库通信。
- `View`：负责表示方面，它从控制器得到模型的状态（控制器会把模型数据放在视图能找到的一个地方），视图还要获取用户输入，并交给控制器。
- `Controller`：从请求获得用户输入，并明确这些输入对模型的影响，告诉模型自行更新。

#### 2.4 J2EE

1. J2EE是一种超级规范，它结合了其他一些规范，J2EE1.4服务器
- 包括servlet2.4，JSP2.0等**Web容器**，用于Web组件
- 包括Enterprise JavaBean（EJB）2.1规范等**EJB容器**，用于业务组件

2. 一个完全兼容的J2EE应用服务器必须有一个Web容器和一个EJB容器（以及其他一些部件）。Tomcat只与J2EE规范中有关Web容器的部分兼容。

3. J2EE应用服务器包括一个Web容器和一个EJB容器；Tomcat是一个Web容器，而不是一个完整的J2EE应用服务器；J2EE1.4服务器包括Servlet2.4规范、JSP2.0规范及EJB2.1规范。

4. 独立的Web容器通常配置为与一个Web服务器（如Apache）协作，不过Tomcat容器本身就能作为一个基本的HTTP服务器。但是在HTTP服务器功能方面，Tomcat没有Apache那么健壮，所以最常见的非EJB Web应用通常会结合使用Apache和Tomcat，Apache作为HTTPWeb服务器，Tomcat作为Web服务器。

> 严格意义上Web服务器只负责处理HTTP协议，只能发送静态页面的内容。而JSP，ASP，PHP等动态内容需要通过CGI、FastCGI、ISAPI等接口交给其他程序去处理。这个其他程序就是应用服务器。  
> 比如Web服务器包括Nginx，Apache，IIS等。而应用服务器包括WebLogic，JBoss等。应用服务器一般也支持HTTP协议，因此界限没这么清晰。但是应用服务器的HTTP协议部分仅仅是支持，一般不会做特别优化，所以很少有见Tomcat直接暴露给外面，而是和Nginx、Apache等配合，只让Tomcat处理JSP和Servlet部分。

### 3. MVC迷你教程

1. 构建一个小Web应用：
- 分析用户视图（浏览器显示的东西），以及高层体系结构；
- 创建用于开发这个项目的开发环境；
- 创建用于部署这个项目的部署环境；
![Tomcat Deploy](/assets/img/20171212_tomcat.png)
- 对Web应用的各个组件完成迭代式的开发和测试。

2. 构建和测试类模型
- 模型规范：在MVC中，模型指应用的后台，应有自己的工具包：包应当是com.xxx.model；其目录结构应是/WEB-INF/classes/com/xxx/model；
- 为模型构建测试类；
- 构建和测试模型

### 4. Servlet

#### 4.1 servlet生命周期

1. Servlet生命周期重要时刻之`init()`方法：
- **调用**：整个生命周期中只调用一次。在servlet实例创建后，并在servlet提供服务前，容器要对servlet调用`init()`。
- **作用**：使servlet处理客户请求前有机会实施初始化。
- **覆盖**：可能覆盖，如数据库连接或向其他对象注册。

2. Servlet生命周期重要时刻之`service()`方法：
- **调用**：第一个客户请求到来时，容器会开始一个新线程或从线程池分配一个线程，并调用servlet的`service()`方法。
- **作用**：查看HTTP方法，并决定对应操作，如`doGet()，doPost()`。
- **覆盖**：不太可能。

3. Servlet生命周期重要时刻之`doGet()`或`doPost()`方法：
- **调用**：`service()`方法根据请求的HTTP方法调用`doGet()`或`doPost()`。
- **作用**：主要代码区。
- **覆盖**：至少其中之一。

4. Servlet的生命周期:
- **搜索servlet类**：容器启动后，寻找已经部署的Web应用，然后开始搜索servlet类文件；
- **加载servlet类**：可能在容器启动时加载，也可能在第一个客户使用时进行，在servlet没有完全初始化之前不能运行servlet的service()方法；
- servlet的**无参构造函数**实例化为一个普通对象；
- servlet调用`init()`方法初始化servlet；
- servlet**在新线程中或从线程池分配一个线程**调用`service()`方法确定HTTP方法（GET POST）并调用对应的方法；`service()`方法总在自己的栈中调用；
- servlet调用`destroy()`方法。

5. **每个请求**都在一个单独的线程中运行。**任何servlet一般都只有一个实例（如此理解Spring的bean一般都是单例的）**，**容器运行多个线程来处理对一个servlet的多个请求**。

6. 不要在servlet构造函数中写入初始化代码。因为**servlet实例化时，servlet的配置文件可能还没有初始化完成**。

#### 4.2 servletness

1. `ServletConfig`对象：
- 每个servlet都有一个`ServletConfig`对象；
- 用于向servlet传递部署时信息（如数据库），而无需将该信息硬编码入servlet中；
- 用于访问**ServletContext**；
- 参数在部署描述文件中配置。

2. `ServletContext`对象：
- 每个Web应用有一个`ServletContext`；
- 用于访问Web应用参数（也在部署描述文件中配置）；
- 相当于一种应用公告栏，可以在这里放置消息（属性），应用的其他部分可以访问这些信息；
- 用于得到服务器信息，包括容器名和容器版本，以及所支持API的版本等。

3. `HttpServletRequest`接口和`HttpServletResponse`接口增加了与HTTP协议相关的方法，如cookie，首部和会话。

4. `HttpServletRequest`接口和`HttpServletResponse`接口由容器提供，这些类不在API中。

#### 4.3 HTTP方法

1. HTTP方法：
- `GET`：要求得到所请求URL上的一个资源或文件。
- `POST`：要求服务器接受附加到请求的体信息，并提供所请求URL上的一个东西。
- `HEAD`：只要求得到GET返回结果的首部部分。HEAD可提供所请求URL的有关信息，无返回部分。
- `TRACE`：要求请求消息回送，这样客户能看到另一端接收了什么，以便测试或排错。
- `PUT`：指出要把所包含的信息放在请求URL上。
- `DELETE`：指出删除所请求URL上的一个资源或文件。
- `OPTIONS`：要求得到一个HTTP方法列表，所请求URL上的东西可以对这些HTTP方法做出相应。
- `CONNECT`：要求连接以建立隧道。

2. `POST`有一个体，`GET`对数据参数有限制，并且参数数据只能放在请求行中，传递数据时POST更加安全。

3. `GET`请求可建立书签，`POST`请求不能建立书签；`GET`用于对数据简单的获取，`POST`用于发送数据进行处理。

4. 在编程中，一个幂等操作的特点是**其任意多次执行所产生的影响均与一次执行的影响相同**。
- `GET`只要得到东西，它不会修改服务器上的任何内容，所以根据定义**`GET`是幂等的**。
- `POST`不是幂等的，`POST`体中的提交数据可能用于不可逆转的事务。
  > 根据HTTP规范`GET`是幂等的，但是可以在servlet中实现一个非幂等的`doGet()`方法，即使数据发生了副作用，客户的`GET`请求本身是幂等的。HTTP的`GET`方法和servlet中`doGET()`方法是有区别的。

5. **简单的超链接往往意味着GET；POST方法不是默认的**。

6. 为何从请求中获得一个InputStream？
- `GET`请求除了首部信息没有别的内容；
- `POST`请求有体信息，大多数情况关心的是`request.getParameter()`抽出的参数值，这些值可能很大；还有可能创建一个servlet来处理计算机驱动的请求，其中请求体包含要处理的文本或二进制内容。此时使用`getReader()`或`getInputStream()`方法获取HTTP请求中的**体**而**不包含首部**。

7. 获取端口：
- `getRemotePort()`，服务器问客户端，得到客户的端口；
- `getServletPort()`，所有请求送到一个端口（服务器监听的端口）；
- `getLocalPort()`，每一个线程的本地端口。

#### 4.4 响应

1. 一般会使用响应对象得到一个输出流，并**使用这个流写出HTML或其他内容给客户端**。

2. 大多数情况下，使用相应只是为了向客户**发回数据**；会对响应调用两个方法**setContentType()和getWriter()**；在此之后，只需要完成I/O将HTML写至流；也可以使用相应设置其他首部、发送错误及增加cookie。

3. `ServletContext`中的`getResourceAsStream()`要求首先有一个斜线（"/"），表示Web应用的根。

4. `ContentType`指MIME类型，HTTP响应头中必须包含。如：`text/html,image/jpeg,application/jar`等。

5. 防止内容类型和输入流之间的冲突，先调用`setContentType()`，再调用获得输出流的方法（`getWrite()`或`getOutputStream()`）。

6. `ServletResponse`提供了`ServletOutputStream`用于输出字节，`PrintWriter`用于输出字符数据；`println()`至`PrintWriter`，`write()`写至`ServletOutputStream`。

7. 设置响应首部：
- `setHeader()`，如果响应中已经有同名的首部，则用这个值替换原来的值；否则向响应增加一个新首部和值。
-` addHeader()`，为响应增加一个新的首部和值；或向一个现有的首部增加另外一个值。
- `setIntHeader()`，用提供的整数值替换现有首部的值；或向响应增加一个新首部和值。

#### 4.5 重定向与请求分派

1. 重定向让**浏览器**完成工作，servlet只是调用响应的`sendRedirect()`方法。

2. `sendRedirect()`使用**相对URL**作为参数，参数为**一个String不是一个URL对象**，相对URL有两种类型，前面带斜线`/`和没有斜线的。
- 没有斜线，容器会相对于原请求URL路径建立完整的URL（需要放入HTTP响应的`Location`首部中）。
- 有`/`开头，容器会相对于Web应用本身建立完整的URL。

3. 请求分派在**服务器**中完成。

4. 当请求到达服务器/容器；servlet决定这个请求应当交给Web应用的另一部分；servlet调用`request.getRequestDispathcer("result.jsp")`；浏览器正确响应请求，浏览器地址栏没有变化。
