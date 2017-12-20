---
layout: post
title: "Head First Servlets and JSP -- III"
categories: Servlet & JSP
tag: Java-Web
---
> `Servlet & JSP`基础，JSP基础等。

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
- taglib指令语法：`<%@ taglib tagdir="/WEB-INF/tags/cool" prefix="cool" %>`，定义JSP可使用的标记库。
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

### 8. 无脚本页面