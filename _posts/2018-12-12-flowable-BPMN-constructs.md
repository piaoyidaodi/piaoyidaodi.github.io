---
layout: post
title: "Flowable文档学习笔记--BPMN架构"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 7. BPMN2.0架构

BPMN 2.0标准对所有相关方都是好事。最终用户不会受到依赖于专有解决方案的供应商的影响。像Flowable这样的开源框架，可以实现一个与大型供应商具有相同功能的解决方案。得益于BPMN2.0标准，从这样一个大型供应商解决方案向Flowable的过渡可以轻松顺畅地进行。

然而，标准的缺点在于它是不同公司之间的许多讨论和妥协的结果。作为一个阅读流程定义的BPMN2.0 XML的开发人员，有时候感觉某些构造或方法非常麻烦。由于Flowable将开发简化作为首要任务，因此我们引入了称为*Flowable BPMN extensions*的东西。这些扩展是用以简化某些构造，且不属于BPMN 2.0规范的新的构造或方法。

尽管BPMN 2.0规范明确指出它是为自定义扩展而设计的，但我们确保：
- 首先这种自定义扩展必须对标准的处理方式进行简单的转换。因此，当您决定使用自定义扩展时不必担心。
- 使用自定义扩展时，应始终给出新的XML元素，属性等以明确的`flowable:`命名空间前缀。Flowable引擎还支持activiti：名称空间前缀。

### 7.1 Events

事件（Events）用于模拟在流程生命周期中发生的事情。事件在BPMN 2.0中总是可视化为**圆圈**，事件的主要类别分为*catching*和*throwing*事件。
- Catching：当流程执行到达事件时，它将等待触发发生。触发器的类型由XML中的内部图标或类型声明定义。捕捉事件通过未填充的内部图标（仅为白色）在视觉上区别于抛出事件。
- Throwing：当流程执行到达事件时，触发器被触发。触发器的类型由XML中的内部图标或类型声明定义。通过填充黑色的内部图标在视觉上与捕获事件区分。

#### 7.1.1 Event Definitions

事件定义定义了事件的语义。没有事件定义，事件“没有什么特别之处”。例如，没有事件定义的启动事件无法指定究竟如何启动流程。如果我们向start事件添加一个事件定义（例如，一个timer事件定义），我们声明何种类型的事件启动流程（在一个timer事件定义的情况下就是到达某个时间点）。

#### 7.1.2 Timer Event Definitions

Timer事件是由定义的计时器触发的事件。它们可以用作start event，intermediate event或boundary event。时间事件的行为取决于使用的业务日程（business calendar）。每个Timer事件都有一个默认的业务日程，但业务日程也可以作为Timer事件定义的一部分给出。

```xml
<timerEventDefinition flowable:businessCalendarName="custom">
    ...
</timerEventDefinition>
```

其中businessCalendarName指向流程引擎配置中的业务日程。省略业务日程时，将使用默认业务日程。

Timer定义必须具有以下一个元素：
- `timeDate`：此格式以ISO8601格式指定固定日期，并在日期下此时将触发触发器。例如：

```xml
<timerEventDefinition>
    <timeDate>2011-03-11T12:13:14</timeDate>
</timerEventDefinition>
```

- `timeDuration`：要指定计时器在触发之前应运行多长时间，可以将timeDuration指定为timerEventDefinition的子元素。使用的格式是ISO8601格式。例如（间隔持续10天）：

```xml
<timerEventDefinition>
    <timeDuration>P10D</timeDuration>
</timerEventDefinition>
```

- `timeCycle`：指定重复间隔，这对于定期启动流程或为过期用户任务发送多个提醒非常有用。

时间周期可以是两种格式定义：ISO8601标准规定的持续时间的格式。如（3个重复间隔，每个持续10小时）：可以指定timeCycle上的可选属性endDate，或者在时间表达式的末尾指定如下：`R3/PT10H/${EndDate}`。当到达endDate时，应用程序将停止为此任务创建其他作业。它接受静态ISO8601标准值，例如`2015-02-25T164211+00:00`，或变量，例如`${EndDate}`。

```xml
<timerEventDefinition>
    <timeCycle flowable:endDate="2015-02-25T16:42:11+00:00">R3/PT10H</timeCycle>
</timerEventDefinition>
```

```xml
<timerEventDefinition>
    <timeCycle>R3/PT10H/${EndDate}</timeCycle>
</timerEventDefinition>
```

如果同时指定了两者，则系统将使用指定endDate属性的。目前，只有BoundaryTimerEvents和CatchTimerEvent支持EndDate功能。

此外，还可以使用cron表达式指定时间周期，示例展示了从整小时开始每隔5分钟触发：`0 0/5 * * * ?`。

持续循环时间更适合处理相对定时器，相对定时器是根据某个特定时间点（例如，用户任务启动的时间）计算的，而cron表达式可以处理绝对定时器，这对于timer start events很有用。

您可以将表达式用于计时器事件定义，通过这样做时您可以根据流程变量影响计时器定义。对于适当的计时器类型，流程变量必须包含ISO8601或cron类型字符串。另外，对于持续时间，可以使用返回java.time.Duration的类型变量或表达式。

```xml
<boundaryEvent id="escalationTimer" cancelActivity="true" attachedToRef="firstLineSupport">
  <timerEventDefinition>
    <timeDuration>${duration}</timeDuration>
  </timerEventDefinition>
</boundaryEvent>
```

只有在启用异步executor时才会触发计时器（在flowable.cfg.xml中必须将asyncExecutorActivate设置为true，因为默认情况下禁用异步执行程序）。

#### 7.1.3 Error Event Definitions

**注意：**BPMN错误与Java异常不同。事实上，两者没有任何共同之处。BPMN错误事件是一种建模业务异常的方法。Java异常以其自己的特定方式处理。

```xml
<endEvent id="myErrorEndEvent">
  <errorEventDefinition errorRef="myError" />
</endEvent>
```

#### 7.1.4 Signal Event Definitions

signal事件是引用一个有名字信号的事件。信号是全局范围的事件（broadcast semantics），并传递给所有活动的处理程序（等待流程实例/捕获信号事件）。

使用signalEventDefinition元素声明信号事件定义。属性signalRef的引用signal子元素。以下是一个流程的摘录，其中信号事件被中间事件抛出并捕获，其中signalEventDefinition引用了同一个signal元素。

```xml
<definitions... >
    <!-- declaration of the signal -->
    <signal id="alertSignal" name="alert" />

    <process id="catchSignal">
        <intermediateThrowEvent id="throwSignalEvent" name="Alert">
            <!-- signal event definition -->
            <signalEventDefinition signalRef="alertSignal" />
        </intermediateThrowEvent>
        ...
        <intermediateCatchEvent id="catchSignalEvent" name="On Alert">
            <!-- signal event definition -->
            <signalEventDefinition signalRef="alertSignal" />
        </intermediateCatchEvent>
        ...
    </process>
</definitions>
```

**抛出Signal事件**

信号可以由流程实例使用BPMN构造抛出，也可以使用JavaAPI以编程方式抛出。org.flowable.engine.RuntimeService上的以下方法可用于以编程方式抛出信号：

```java
RuntimeService.signalEventReceived(String signalName);
RuntimeService.signalEventReceived(String signalName, String executionId);
```

signalEventReceived(String signalName)和signalEventReceived(String signalName, String executionId)之间的区别在于第一种方法将信号全局抛出到所有订阅处理程序（broadcast semantics），第二种方法仅将信号传递给特定的execution。

**捕捉Signal事件**

Signal事件可以被中间捕获信号事件或信号边界事件捕获。

**查询Signal事件订阅**

可以查询已订阅的特定Signal事件的所有execution：

```java
List<Execution> executions = runtimeService.createExecutionQuery()
      .signalEventSubscriptionName("alert")
      .list();
```

然后我们可以使用signalEventReceived(String signalName, String executionId)方法将信号传递给这些execution。

**Signal事件范围**

默认情况下，信号是广播流程引擎（broadcast process engine wide）。这意味着您可以在流程实例中抛出signal事件，而具有不同流程定义的其他流程实例可以对此事件的发生做出反应。

然而，有时希望仅在同一过程实例内对信号事件作出反应。例如，用例是当两个或多个活动互斥时的流程实例中的同步机制。

要限制信号事件的范围，请将（非BPMN2.0标准）scope属性添加到信号事件定义中：`<signal id="alertSignal" name="alert" flowable:scope="processInstance"/>`，此属性的默认值为“global”。

#### 7.1.5 Message Event Definitions

消息事件是引用有名称的消息的事件。消息具有名称和有效负载。与信号不同，消息事件始终指向单个接收器。

使用messageEventDefinition元素声明消息事件定义。属性messageRef引用message子元素。以下是一个过程的摘录，其中两个消息事件由start事件和中间捕获消息事件声明和引用。

```xml
<definitions id="definitions"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:flowable="http://flowable.org/bpmn"
  targetNamespace="Examples"
  xmlns:tns="Examples">

  <message id="newInvoice" name="newInvoiceMessage" />
  <message id="payment" name="paymentMessage" />

  <process id="invoiceProcess">

    <startEvent id="messageStart" >
    	<messageEventDefinition messageRef="newInvoice" />
    </startEvent>
    ...
    <intermediateCatchEvent id="paymentEvt" >
    	<messageEventDefinition messageRef="payment" />
    </intermediateCatchEvent>
    ...
  </process>

</definitions>
```

**抛出Message事件**

对于嵌入的流程引擎，Flowable不关心实际接收消息。这将取决于环境并且需要特定于平台的活动，例如连接到JMS（Java消息服务）Queue/Topic或处理Webservice或REST请求。因此，消息的接收是您必须实现的应用程序或基础结构的一部分。

在应用程序中收到消息后，您必须决定如何处理它。如果消息应触发新流程实例的启动，请在运行时服务提供的以下方法之间进行选择：

```java
ProcessInstance startProcessInstanceByMessage(String messageName);
ProcessInstance startProcessInstanceByMessage(String messageName, Map<String, Object> processVariables);
ProcessInstance startProcessInstanceByMessage(String messageName, String businessKey,
    Map<String, Object> processVariables);
```

这些方法使用引用的消息启动一个流程实例。如果消息需要由现有的流程实例接收，则首先必须将消息关联到特定流程实例（请参阅下一节），然后触发需执行的等待execution。运行时服务提供以下方法，用于根据消息事件订阅触发execution：

```java
void messageEventReceived(String messageName, String executionId);
void messageEventReceived(String messageName, String executionId, HashMap<String, Object> processVariables);
```

**查询Message事件订阅**

在消息启动事件的情况下，消息事件订阅与特定的流程定义相关联。可以使用ProcessDefinitionQuery查询此类消息订阅：

```java
ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery()
      .messageEventSubscription("newCallCenterBooking")
      .singleResult();
```

由于特定消息订阅只能有一个流程定义，因此查询始终返回零或一个结果。如果更新了流程定义，则只有最新版本的流程定义才能订阅消息事件。

在中间捕获消息事件的情况下，消息事件订阅与特定execution相关联。可以使用ExecutionQuery查询此类消息事件订阅：

```java
Execution execution = runtimeService.createExecutionQuery()
      .messageEventSubscriptionName("paymentReceived")
      .variableValueEquals("orderId", message.getOrderId())
      .singleResult();
```

此类查询称为关联查询，通常需要有关流程的知识（在这种情况下，给定orderId最多只有一个流程实例）。

#### 7.1.6 Start Events

启动事件指示流程的开始位置。启动事件的类型（流程在消息到达时、在特定时间间隔时开始等等），定义流程的启动方式，在事件的可视化表示中显示为一个小图标。在XML表示中，类型由子元素的声明给出。

开始事件总是被捕获：从概念上讲，事件（在任何时候）等待直到某个触发发生。在start事件中，可以指定以下Flowable特定属性：
- `initiator`：标识当流程启动时的特殊变量名称，该变量名称将存储经过身份验证的用户ID。 例如：`<startEvent id="request" flowable:initiator="initiator" />`。

必须使用IdentityService.setAuthenticatedUserId(String)方法设置经过身份验证的用户，并使用try-finally块进行包括，如下所示：

```java
try {
  identityService.setAuthenticatedUserId("bono");
  runtimeService.startProcessInstanceByKey("someProcessKey");
} finally {
  identityService.setAuthenticatedUserId(null);
}
```

此代码已粘贴到Flowable应用程序中，因此它与Forms结合使用。

#### 7.1.7 None Start Event

None Start Event在技术上意味着未指定启动流程实例的触发器。这意味着引擎无法预测何时必须启动流程实例。通过调用startProcessInstanceByXXX方法之一的API启动流程实例时，将使用none start事件，`ProcessInstance processInstance = runtimeService.startProcessInstanceByXXX();`

**XML表示**
none start event的XML表示是没有任何子元素的正常启动事件声明（其他启动事件类型都具有声明启动类型的子元素）。
```xml
<startEvent id =“start”name =“my start event”/>
```

**自定义扩展**
formKey：用户在启动新流程实例时必须填写的表单定义的引用。更多信息可以在表单部分找到示例：`<startEvent id="request" flowable:formKey="request" />`


PASS
