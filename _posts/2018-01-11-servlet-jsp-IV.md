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