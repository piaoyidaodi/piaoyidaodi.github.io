---
layout: post
title: "Head First Servlets and JSP -- IV"
categories: Servlet&JSP
tag: Java-Web
---
> `Servlet & JSP`基础，定制标签基础等。

### 10. 定制标签开发

#### 10.1 标签文件调用可重用内容

1. 代码示例：
    ```html
    <%@ attribute name="subTitle" required="true" rtexprvalue="true"%>
    <%@ tag body-content="tagdependent" %>
    <img src="image.jpg"/>
    <strong>${subTitle}</strong>
    <strong><jsp:doBody/></strong>
    ```
    ```html
    <%@ taglib prefix="myTags" tagdir="/WEB-INF/tags" %>
    <html><body>
    <myTags:Header subTitle="Hello">
        World！
    </myTags:Header>
    </body></html>
    ```

2. 建立标签文件的简单方法：
- 取一个被包含文件重命名为`.tag`扩展名；
- 将标签文件放入`WEB-INF`目录下`tags`目录中；
- 在JSP中放taglib指令（拥有tagdir属性），并调用该标记.

3. 对于标签文件，发送的是标签属性而不是请求参数，所有标签属性都只有标签作用域。`<jsp:include>`中的`<jsp:param>`的值会作为请求参数。**对于Web应用来说`<jsp:param>`的名/值对就如同表单提交得来的一样**。

4. 定制标签(包括JSTL)属性需要在TLD中指定，但是标签文件（.tag）属性不再TLD中指定。

5. 为标签文件声明body-content类型只有一个方法，使用标签文件指令：tag指令。**标签文件中的tag指令相当于JSP页面中的page指令**。body-content默认为`scriptless`，可选为`empty`（标签体为空）和`tagdependent`（标签体看做纯文本）。

6. 标签文件(.tag)查找位置：
- 直接在WEB-INF/tags目录中查找；
- 在WEB-INF/tags的子目录中查找；
- 在WEB-INF/lib下JAR文件的META-INF/tags目录中查找；
- 在WEB-INF/lib下JAR文件的META-INF/tags的子目录中查找；
- 如果标签文件部署在一个JAR中，这个标签文件必须有一个TLD。

7. 注意事项：
- 标签文件(.tag)最后作为JSP的一部分，可以使用隐式对象，标签文件要使用**JSPContext**。
- 标签文件中可以放入脚本；但如果一个标签要调用标签文件，这个标签的体中是不能脚本的。

### 11. 部署Web应用

1. 通过WAR文件部署Web应用，可以在`META-INF/MANIFEST.MF`文件中声明库依赖，这样在部署时就能检查容器是否找到应用依赖的包和类。

2. 只要把文件放在`WEB-INF`下就能避免用户直接访问静态资源；或者如果部署为WAR文件，可以把不允许直接访问的文件放在META-INF/tags下。

3. 容器在查找`WEB-INF/lib`中的jar文件时，会首先查找`WEB-INF/classes`目录中的类。

4. servlet映射中的URL模式可能完全是虚拟的。url的映射分为**完全匹配（匹配名必须以/开头）、目录匹配（匹配名必须以/开头，总是以/*结束）、扩展名匹配（必须以\*开头，后接一个加点的扩展名）**。容器会按照以上顺序查找匹配，如果目录匹配出现多匹配情况，则会选择最长匹配。

5. 配置错误页面，可在配置文件中的error-page元素下配置`exception-type/error-code`元素和location元素，其中前者不能同时使用。

6. 如果希望在部署时加载servlet，而不是等到第一个请求到来时才加载，可以在配置文件中使用`load-on-startup`元素。通过设置该元素的值非负，在部署时加载servlet，且该值越小加载越早；若多个servlet具有相同的等级，则按照在配置文件中声明的顺序加载。

### 12. 过滤器

与servlet非常类似，过滤器就是java组件，请求发送到servlet之前可以用过滤器截获和处理请求，另外servlet结束工作后，在相应发给客户之前，可以用过滤器处理相应。