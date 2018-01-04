---
layout: post
title: "Head First Servlets and JSP -- III"
categories: Servlet&JSP
tag: Java-Web
---
> `Servlet & JSP`基础，JSP基础，EL基础等，JSTL基础等。

### 7. JSP

#### 7.1 概述

1. 容器查看JSP，把它转化为Java源代码（.java），并编译成为完整的Java servlet类，容器会加载这个servlet类，实例化并初始化，为每一个请求建立一个单独的线程，并调用`service()`方法。

#### 7.2 JSP语法

1. JSP脚本：`<% ... %>`
- scriptlet语法：`<% out.println(); %>`
- scriptlet脚本是常规Java，**语句后有分号**。

2. JSP指令（一共3个）：`<%@ ... %>`
- page指令语法：`<%@ page import="java.util.*" session="false" %>`，定义页面特定的属性。
- page的属性共13个，其中前6个为重点**import, isThreadSafe, isErrorPage, errorPage, contentType, isELIgnored**, session, language, extends, buffer, autoFlush, info, pageEncoding。
- taglib指令语法：`<%@ taglib tagdir="/WEB-INF/tags/cool" prefix="cool" %>`，定义JSP可使用的标签库。
- include指令语法：`<%@ include file="wickedHeader.html" %>`，定义在转换时增加到当前页面的文本和代码。
- **指令语法后没有分号**。

3. JSP表达式：`<%= ... %>`
- **表达式语法后没有分号**。
- 表达式会成为`out.print()`的参数，容器拿到内容后把它作为参数打印到隐式响应`PrintWriter out`。
- 不能把返回类型为`void`的方法作为表达式。

4. JSP声明：`<%! ... %>`
- 声明语法：`<%! int count=0; %>`
- **声明语法后有分号**。
- 声明静态及实例变量和方法。

5. JSP表达式语言：`${something}`

6. JSP动作：
- 标准动作：`<jsp:include page="wickedFooter.jsp"/>`
- 其他动作：`<c:set var="rate" value="32"/>`

7. JSP注释：`<%-- --%>`

8. 作用小结：
- JSP指令：用于向容器提供特殊的指示。
- JSP脚本：就是普通的Java代码，会原样放在所生成servlet的服务方法中。
- JSP表达式：结果总会成为`print()`方法的参数。
- JSP声明：生成servlet类变量。

#### 7.3 JSP转换为Servlet

1. 容器把所有**scriptlet代码和表达式代码**放入一个通用的服务方法中，可以认为是一个全面符合的`doGet/doPost`中。

2. JSP隐式对象（9个）：
- JSPWriter - out
- HttpServletRequest - request
- HttpServletResponse - response
- HttpSession - session
- ServletContext - application
- ServletConfig - config
- Throwable - exception ： 只有指定的错误页面才能使用该隐式对象
- PageContext - pageContext ： 封装了其他隐式对象
- Object - page

3. JSP转化为Servlet的3个重要方法，如果方法名前面没有下划线就是可覆盖的方法：
- `jspInit()`：该方法由`init()`方法调用。
- `jspDestroy()`：该方法由`destroy()`方法调用。
- `_jspService()`：该方法由`service()`方法调用。

4. JSP的生命周期：
- 部署一个.jsp文件，此时容器读取配置文件但不对jsp做任何处理。
- 首个调用此.jsp文件，容器将.jsp转换为一个servlet类的.java源文件。**此阶段会在发现JSP语法错误**。
- 容器尝试将此.java源文件编译为一个.class文件。**此阶段会捕获Java语言语法错误**。
- **转换和编译只发生一次**；加载class文件；实例化servlet，调用jspInit()方法，成为真正的servlet；新建线程处理该请求。

5. JSP配置：`<web-app><servlet><servlet-name><jsp-file>`

6. JSP中的属性：
- application.setAttribute(), request.setAttribute(), session.setAttribute()
- pageContext.setAttribute()

7. PageContext引用：
- 静态字段：`APPLICATION_SCOPE, PAGE_SCOPE, REQUEST_SCOPE, SESSION_SCOPE`
- 用法：`pageContext.setAttribute(), pageContext.getAttribute(), pageContext.findAttribute()`
- `findAttribute()`查找路径：请求作用域 -> 会话作用域 -> 应用作用域。
- PageContext引用可得到任意作用域的属性，包括从页面作用域得到绑定的PageContext属性

8. JSP禁用脚本元素（脚本、表达式、声明）：`<web-app><jsp-config><scripting-invalid>`

9. JSP禁用EL：配置文件中使用`<web-app><jsp-config><el-ignored>`；使用`<%@ page isELIgnored="true" %>`。

#### 7.4 page, request, session, application作用域

1. page作用域：
- jsp默认的作用域是page（页面作用域），这个作用域中的对象只能在该页面中使用，不允许在其他页面使用，即只要页面发生跳转就会消失。我们可以通过调用pageContext这个隐含的对象的getAttribute()和setAttribute()方法去获取和设置需要传递、共享具有这种范围类型的数据。（pageContext对象还提供了访问其他范围对象的getAttribute方法）。
- page范围内的对象，在客户端每次请求JSP页面时创建，在页面向客户端发送回响应或请求被转发（forward）到其他的资源后被删除。

2. request作用域：
- request（请求作用域）作用于，从HTTP请求到服务器处理结束返回响应的整个过程。要注意的是，因为每一个客户请求都是不同的，所以对于每一个新的请求（刷新页面也算），都要重新创建和删除这个范围内的对象。

3. session作用域：
- session（会话作用域）的对象作用于浏览器打开到浏览器关闭发出的所有请求。Session的作用范围为用户和服务器持续连接的一段时间，但与服务器断线，这个属性就无效。
- 当浏览器发出第一个请求时，就认为session的作用时间已经开始了，但是它的结束时间还是不太好判断，所以我们可以类似于处理“系统响应超时”这种情况的方法，设置：如果一定的时间内客户端没有反应，则认为会话结束。Tomcat的默认值为120分钟，但这个值也可以通过HttpSession的setMaxInactiveInterval()方法来设置最大时长。

4. application作用域：
- application（应用程序作用域）中的对象作用于整个应用程序，从服务器一开始执行服务，一直到服务器关闭为止。从这看来，application的作用范围最广，作用的时间也最长。所以使用时要特别注意不然可能会造成服务器负载越来越重的情况。

> 注意：根据jsp规范，用于某个对象的名称必须在所有作用域中都是唯一的。也就是说，如果application作用域中有一个名为user的对象，而且在request作用域中用相同的名称保存着另一个对象，那么容器可能会移除第一个对象，尽管很少有容器会执行这项规则，但是为了使您的项目更加完善，还是应该确保在任何地方都是用唯一的名称，除非所保存的对象为同一个。

### 8. 无脚本页面

#### 8.1 JavaBean标准动作

1. 使用`<jsp:useBean>`声明和初始化一个bean属性，`<jsp:getProperty>`得到bean属性的性质值，`<jsp:setProperty>`设置bean属性的性质值。

2. `<jsp:useBean id="person" type="foo.Person" class="foo.Employee" scope="request" />`
- id对应servlet中的名`req.setAttribute("person",p)`；type接口类、抽象类等引用类类型；class实例化类类型；scope标识属性作用域，默认为page。
- 如果标签中找不到对应为person的属性，就会创建一个新的。
- 如果只有type而没有class，则page作用域内的bean必须已经存在，即person属性必须存在；使用class，则class不能为抽象类，且有无参公共构造器。

3. `<jsp:getProperty name="person" property="name" />`
- name对应id；property标识属性中的性质名，Person类中的定义。

4. `<jsp:setProperty name="person" property="name" value="Jon" />`

5. `<jsp:useBean><jsp:setProperty /></jsp:useBean>`体
- 使用`<jsp:useBean>`体可有条件的运行代码，只有找不到对应bean属性而创建一个新bean时才会运行体中的代码。

6. 请求直接到JSP，使用param属性：
- `<jsp:setProperty name="person" property="name" param="userName">`
- 如果让请求的参数名与bean性质名全部匹配，可取消param属性，property为*。

7. Bean标签会自动转换String和基本类型的性质，如setProperty()。如果在标签中使用JSP脚本，则不会完成自动转换。

#### 8.2 表达式语言（EL）

1. EL语法格式：`${first.second}`，解决想获取属性的嵌套性质，即性质的性质的问题。

2. `first`可以是一个**EL隐式对象**，也可以是一个**属性**。
- **EL隐式对象**：(pageScope, requestScope, sessionScope, applicationScope)作用域属性的Map；(param, paramValues)请求参数Map；(header, headerValues)请求首部的Map；cookie；(initParam)上下文初始化参数；pageContext（唯一一个非Map，而是pageContext实际引用）。
- **属性**：页面作用域、请求作用域、会话作用域、应用作用域的属性。

3. `.`访问符。
- `.`的左边必须为**Map或bean**。
- `.`的右边必须遵循Java的命名规范。

4. `[]`访问符。
- `[]`的左边可以为**Map, bean, List或数组**。
- `[]`中是String直接量（即**用引号引的串**），可以是一个Map键，或一个bean性质，还可以是List或数组中的索引。
- `[]`中**数组和List**的String索引会强制转换为int。
- **如果`[]`中没有引号，则容器会计算`[]`中的内容，搜索与该名字绑定的属性，并替换为这个属性值（如果有一个同名的隐式对象，那么总使用隐式对象）**。
- 如servlet中`req.setAttribute("Genre","Ambient");`则JSP中`Music is ${musicMap[Genre]}`解析为`Music is ${musicMap["Ambient"]}`。

5. `requestScope`等作用域映射不是真正的对象。使用`requestScope`作用域得到的是请求属性，而不是request性质，通过**pageContext**获取性质。

6. 作用域隐式对象的作用：
- 防止命名冲突。
- request的属性名为一个String，可能不遵守Java命名规范，使用作用域对象，可以使用`[]`操作属性名。

7. Cookie和初始化参数：
- `${cookie.userName.value}`
- `${initParam.mainEmail}`

#### 8.3 EL中的函数和操作符

1. EL中的函数
- 编写有一个公共静态方法的Java类，将类文件放入/WEB-INF/classes目录中。
- 编写一个编辑库描述文件（TLD）。TLD提供了定义函数的Java类与调用函数的JSP之间的映射。TLD文件放在/WEB-INF目录中，扩展名为.tld。
- 在JSP中放一个taglib指令，定义前缀。
- 使用EL调用函数。按照`${prefix:name()}`的形式调用函数。

2. 在算术表达式中，EL把null值看做是0；在逻辑表达式中，EL把null值看做是false。

3. 要在JSP中使用函数，必须使用taglib指令声明一个命名空间。在taglib指令中放一个prefix属性，告诉容器要调用的函数在哪个TLD中可以找到，如`<%@ taglib prefix="mine" uri="/WEB-INF/foo.tld" %>`。

#### 8.4 布局模板

1. `include`指令：在转换时发生
- 在转换前把代码复制到每个JSP中，再编译为生成的servlet。

2. `<jsp:include>`标准动作：在运行时发生
- include标准动作在运行时插入“插入段”的响应。
- 该动作的关键是，容器要根据页面（page）属性创建一个RequestDispatcher，并应用include()方法。所分派/包含的JSP针对同样的请求和响应对象执行，且在同一个线程中运行。
- 容器代码看到include标准动作，并在所生成的servlet代码中插入一个方法调用，这会在运行时动态的将页面响应相合并。

3. `<jsp:forward>`发生转发时，缓冲区会在转发之前清空。

4. `<jsp:param>`定制包含的内容，只能用在`<jsp:include>`或`<jsp:forward>`标准动作中。

### 9. JSP标准标签库（JSTL）

#### 9.1 标准标签库

1. `<c:out value="" >`标签：
- 使用`escapeXml`属性，设置为`true`时，所有XML都转换为浏览器可呈现的形式。其中`<`转换为`&lt`，`>`转换为`&gt`，`&`转换为`&amp`，`'`转换为`&#039`，`"`转换为`&#034`。
- 使用EL输出用户录入的串，可能会导致跨网站脚本攻击。
- 使用`default`属性，若value计算为null时，希望打印的值。

2. `<c:forEach>`标签：
- 迭代处理数组，集合，Map，逗号分隔的String。
- `<%@ tablib profix="c" uri="http:java.sun.com/jsp/jstl/core" %>`
- `<c:forEach var="movie" items="${movieList}">${movie}</c:forEach>`
- forEach标签可以互相嵌套，var的作用域仅限于标签内部。
- 前缀c只是一个标准约定，用来表示JSTL中一组core标签。

3. `<c:if>`标签：
- `<c:if test="${userType eq 'member'}">doSomething</c:if>`
- 标签和EL中单引号和双引号都可以使用。

4. `<c:choose>`标签和同伴`<c:when>``<c:otherwise>`
- 如果`<c:when>`中的测试都不为ture，`<c:otherwise>`就会默认选择运行。
- `<c:choose><c:when></c:when><c:otherwise></c:otherwise></c:choose>`

5. `<c:set>`标签：
- 有var和target两种设置。**var**用于设置属性变量，**target**用于设置bean和Map，这两个版本都有有体和没有体两种形式。
- var版本：没有体形式**如果value的值计算为null，则该变量会被删除**；有体形式**体会被计算为变量的值**。
- target版本：没有体形式**target必须计算为一个Map或bean，若为null或非Map和bean则容器抛出异常**；有体形式**体会被计算为对象性质的设定值**。target的值应当为EL表达式或一个脚本表达式或`<jsp:attribute>`。
- `<c:set>`中不能同时又var和target属性。
- scope是可选的属性，默认为页面（page）作用域。
- **var所指定的属性不存在，当且仅当value不为null时，才会创建新属性**。

6. `<c:remove>`标签：
- `<c:remove var="" scope="">`
- var属性必须是一个String直接量，不能是表达式；scope是可选的，如果未指定则会从所有作用域删除这个属性。

7. `<c:import>`标签：
- include指令使用file，`<jsp:include>`使用page，`<c:import>`使用url。前两种机制不能超出当前容器范围，`<c:import>`则可以。
- `<c:param name="" value="">`包含对被包含部分的传入参数。

8. `<c:url>`标签，对URL重写：
- `<c:url value='/XXX.jsp'>`，如果禁用了cookie会在value后增加jsessionid。
- URL通常需要编码，把不安全的字符替代为其他字符，然后再在服务器端完成解码。
- `<c:url value=""><c:param name="" value=""/></c:url>`完成对url的编码。

9. 在配置文件中配置错误页面：
- 普通型错误页面：`<exception-type>java.lang.Throwable</exception-type>`。
- 明确异常错误页面：`<exception-type>java.lang.XXException</exception-type>`。
- HTTP状态码错误页面：`<error-code>java.lang.XXException</error-code>`。

10. 错误页面实际就是一个处理异常的JSP，所以容器为这个页面提供了一个额外的exception对象。

11. `<c:catch>`标签，同时作为try/catch部分：
- 把有风险的EL或标签调用包在`<c:catch>`标签中，异常将被捕获。
- 在`<c:catch>`标签中使用var属性，它会把异常对象放在页面作用域，并按声明的var命名。标签体内无法使用有关异常的任何信息。
- `<c:catch>`的工作方式与try一样，出现异常后，`<c:catch>`体中余下的部分不再运行。

#### 9.2 标签库描述文件（TLD）

1. 创建标签库须知：
- **标签名和语法**：标签名前缀、标签名；语法包括必要和可选的属性，标记是否有体，每个属性的类型以及属性是否可以是一个表达式。
- **库URI**：URI是标签库描述文件的唯一标识符，容器需要这个信息将JSP中使用的标签名映射到实际调用的Java代码。

2. TLD描述了两个主要内容：定制标签和EL函数。

3. 容器会在4个位置查找TLD：
- 直接在WEB-INF目录中查找。
- 直接在WEB-INF的子目录中查找。
- 在WEB-INF/lib下的一个JAR文件中的META-INF目录中查找。
- 在WEB-INF/lib下的一个JAR文件中的META-INF目录的子目录中查找。