---
layout: post
title: "Flowable文档学习笔记————总体介绍"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 1. 总体介绍

### 1.1 注意事项

1. Flowable是Activiti的一个分支，由于Activiti主创人员发生分歧，而拉出来单干的代表。丰富和完善了Activiti6中的漏洞和功能。

2. Flowable 6.4.0至少需要JDK8及其以上的支持。

3. 所有在包名中含有`impl`的类，是接口的内部实现，应只能用于内部使用。

### 1.2 Flowable概览

Flowable是一个用Java编写的轻量级业务流程引擎。Flowable流程引擎允许您部署BPMN2.0流程定义（用于定义流程的行业XML标准）、创建流程定义的实例、进行查询、访问活动或历史流程实例及相关数据等等。

Flowable在将其添加到您的应用程序/服务/体系结构时是非常灵活的。您可以通过将Flowable库（可用作JAR）添加到您的应用程序或服务中嵌入引擎。由于它是JAR，因此您可以轻松地将其添加到任何Java环境中：JavaSE；servlet容器，如Tomcat或Jetty，Spring；JavaEE服务器，例如JBoss或WebSphere等。或者，您可以使用Flowable REST API通过HTTP进行通信。还有几个Flowable应用程序（Flowable Modeler，Flowable Admin，Flowable IDM和Flowable Task），它们提供了用于处理流程和任务的开箱即用的示例UI。

配置Flowable的所有方法的共同点是core引擎，可以将其视为公开API以管理和执行业务流程的服务集合。

### 1.2 Flowable简单配置使用

#### 1.2.1 简单示例的Maven依赖配置

```xml
<dependencies>
  <dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-engine</artifactId>
    <version>6.4.0</version>
  </dependency>
  <!-- h2,mysql -->
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.3.176</version>
  </dependency>
  <dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>1.7.21</version>
  </dependency>
  <!-- log4j -->
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.21</version>
  </dependency>
  <!-- log4j2 -->
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>X.X.X</version>
  </dependency>
</dependencies>
```

#### 1.2.2 简单的概念和流程

- -> ProcessDefinition：使用BPMN2.0格式（XML）编制，任一个ProcessDefinition可以开启多个Process实例，可以认为是流程的蓝图。
- --> Deploy ProcessDefinition：存储XML文件到数据库；ProcessDefinition被解析为一个内部的可执行对象模型，用于生成流程实例。

- -> ProcessEngineConfiguration：用于创建、配置和微调ProcessEngine，多数情况使用xml文件配置，但是也可以通过编程实现。配置该对象至少需要使用JDBC连接数据库；
- --> ProcessEngine：核心类接口的实现对象。

- -> RepositoryService：从ProcessEngine中获取，并传入xml文件使用deploy()方法执行，生成ProcessDefinition。
- -> RuntimeService：从ProcessEngine中获取，并可通过startProcessInstanceByKey()方法，传入xml中的流程id，以及map形式的参数。调用此方法时将会在start event中创建execution。
- -> TaskService：从ProcessEngine中通过getTaskService()获取。用于获取和管理Task。
- -> JavaDelegate：流程中的某项工作，可能是流程之外的Java业务逻辑。


**注意事项**

1. 需要配置默认的log文件以及log库文件，如log4j2等。
2. ProcessDefinition概述：**圆圈**代表任务开始；**矩形**代表用户任务或一项外部操作；**四边形中间带X**表示需要决定的决策门。
3. 在FLowable中Transactionality（事务）在保证数据一致性和处理并发问题中扮演很重要的作用。Flowable的API调用默认都是同步的，这表明事务将会在方法调用返回后开始并提交。当开始一个process时，将会有一个数据库事务从process开始直到下一个等待状态（wait state），当事务处于等待状态中时将没有CPU和内存资源的使用。


**ProcessDefinition简单例子**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:xsd="http://www.w3.org/2001/XMLSchema"
  xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
  xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
  xmlns:omgdi="http://www.omg.org/spec/DD/20100524/DI"
  xmlns:flowable="http://flowable.org/bpmn"
  typeLanguage="http://www.w3.org/2001/XMLSchema"
  expressionLanguage="http://www.w3.org/1999/XPath"
  targetNamespace="http://www.flowable.org/processdef">

  <process id="holidayRequest" name="Holiday Request" isExecutable="true">

    <startEvent id="startEvent"/>
    <sequenceFlow sourceRef="startEvent" targetRef="approveTask"/>

    <userTask id="approveTask" name="Approve or reject request" flowable:candidateGroups="managers"/>
    <sequenceFlow sourceRef="approveTask" targetRef="decision"/>

    <exclusiveGateway id="decision"/>
    <sequenceFlow sourceRef="decision" targetRef="externalSystemCall">
      <conditionExpression xsi:type="tFormalExpression">
        <![CDATA[
          ${approved}
        ]]>
      </conditionExpression>
    </sequenceFlow>
    <sequenceFlow  sourceRef="decision" targetRef="sendRejectionMail">
      <conditionExpression xsi:type="tFormalExpression">
        <![CDATA[
          ${!approved}
        ]]>
      </conditionExpression>
    </sequenceFlow>

    <serviceTask id="externalSystemCall" name="Enter holidays in external system"
        flowable:class="org.flowable.CallExternalSystemDelegate"/>
    <sequenceFlow sourceRef="externalSystemCall" targetRef="holidayApprovedTask"/>

    <userTask id="holidayApprovedTask" name="Holiday approved" flowable:assignee="${employee}"/>
    <sequenceFlow sourceRef="holidayApprovedTask" targetRef="approveEnd"/>

    <serviceTask id="sendRejectionMail" name="Send out rejection email"
        flowable:class="org.flowable.SendRejectionMail"/>
    <sequenceFlow sourceRef="sendRejectionMail" targetRef="rejectEnd"/>

    <endEvent id="approveEnd"/>

    <endEvent id="rejectEnd"/>

  </process>

</definitions>
```
