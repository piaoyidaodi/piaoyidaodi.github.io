---
layout: post
title: "Flowable文档学习笔记--History"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 9. History

History是捕获流程执行期间发生的事情并永久存储它的组件。与运行时数据相反，在完成流程实例后，历史数据也将保留在DB中。有6个history实体：
- HistoricProcessInstances，包含有关当前和过去流程实例的信息。
- HistoricVariableInstances，包含最新的流程变量或任务变量。
- HistoricActivityInstances，包含有关单个activity执行的信息。
- HistoricTaskInstances，包含有关当前和过去（已完成和已删除）任务实例的信息。
- HistoricIdentityLinks，包含有关任务和流程实例的当前和过去身份链接的信息。
- HistoricDetails，包含与历史流程实例、活动实例或任务实例相关的各种信息。

由于DB包含过去和正在进行的实例的历史实体，因此您可能需要考虑如何查询这些表，以便最大限度地减少对运行时流程实例数据的访问，从而保持运行时执行的性能。

### 9.1 Querying history

在API中，可以查询所有6个历史实体。 HistoryService有以下公开方法：`createHistoricProcessInstanceQuery()`，`createHistoricVariableInstanceQuery()`，`createHistoricActivityInstanceQuery()`，`getHistoricIdentityLinksForTask()`，`getHistoricIdentityLinksForProcessInstance()`，`createHistoricDetailQuery()`和`createHistoricTaskInstanceQuery()`。

#### 9.1.1 HistoricProcessInstanceQuery

获取已完成的10个HistoricProcessInstances，且花费了最多时间来完成（最长持续时间）所有流程的定义XXX。

```java
historyService.createHistoricProcessInstanceQuery()
  .finished()
  .processDefinitionId("XXX")
  .orderByProcessInstanceDuration().desc()
  .listPage(0, 10);
```

#### 9.1.2 HistoricVariableInstanceQuery

从所有id为xxx已完成的流程实例中，获取已完成的按照变量名排序的HistoricVariableInstances。

```java
historyService.createHistoricVariableInstanceQuery()
  .processInstanceId("XXX")
  .orderByVariableName.desc()
  .list();
```

#### 9.1.3 HistoricActivityInstanceQuery

从所有使用id为XXX的processDefinition定义的流程中获取已完成的serviceTask类型的最后一个HistoricActivityInstance。

```java
historyService.createHistoricActivityInstanceQuery()
  .activityType("serviceTask")
  .processDefinitionId("XXX")
  .finished()
  .orderByHistoricActivityInstanceEndTime().desc()
  .listPage(0, 1);
```

#### 9.1.4 HistoricDetailQuery

下一个示例，获取已在流程中使用id 123完成的所有变量更新。此查询将仅返回HistoricVariableUpdates。请注意，每次在流程中更新变量时，某个变量名称可能具有多个HistoricVariableUpdate条目。您可以使用orderByTime（变量更新完成的时间）或orderByVariableRevision（更新时的运行时变量的修订版）来查找它们发生的顺序。

```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .processInstanceId("123")
  .orderByVariableName().asc()
  .list()
```

此示例获取在任何任务中或在以123开始流程时提交的所有表单属性。 此查询仅返回HistoricFormProperties。

```java
historyService.createHistoricDetailQuery()
  .formProperties()
  .processInstanceId("123")
  .orderByVariableName().asc()
  .list()
```

最后一个示例获取ID为123的执行任务的所有变量更新。这将返回在任务（任务局部变量）上设置的变量的所有HistoricVariableUpdates，而不返回流程实例上的变量。

```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .taskId("123")
  .orderByVariableName().asc()
  .list()
```

可以使用TaskService或TaskListener中的DelegateTask设置任务局部变量：

```java
taskService.setVariableLocal("123", "myVariable", "Variable value");

public void notify(DelegateTask delegateTask) {
  delegateTask.setVariableLocal("myVariable", "Variable value");
}
```

#### 9.1.5 HistoricTaskInstanceQuery

获得10个已完成的且花费最多时间完成所有任务（最长持续时间）的HistoricTaskInstance。

```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .orderByHistoricTaskInstanceDuration().desc()
  .listPage(0, 10);
```

获取使用包含“invalid”的删除原因，删除的HistoricTaskInstances，且这些任务最后分配给用户kermit。

```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .taskDeleteReasonLike("%invalid%")
  .taskAssignee("kermit")
  .listPage(0, 10);
```

### 9.2 History configuration

通过使用枚举org.flowable.engine.impl.history.HistoryLevel可以以编程方式配置历史记录级别：

```java
ProcessEngine processEngine = ProcessEngineConfiguration
  .createProcessEngineConfigurationFromResourceDefault()
  .setHistory(HistoryLevel.AUDIT.getKey())
  .buildProcessEngine();
```

该级别也可以使用flowable.cfg.xml或spring-context中定义：

```xml
<bean id="processEngineConfiguration" class="org.flowable.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
  <property name="history" value="audit" />
  ...
</bean>
```

可以配置以下历史记录级别：
- none：跳过所有历史存档。这是运行时流程执行的最佳性能，但不会提供历史信息。
- activity：归档所有流程实例和活动实例。在流程实例结束时，顶级流程实例变量的最新值将复制到历史变量实例。没有详细信息将被存档。
- audit：这是默认值。它存档所有流程实例，活动实例，持续保持变量值同步以及提交的所有表单属性，以便通过表单进行的所有用户交互都是可追踪的并且可以进行审计。
- full：这是历史存档的最高级别，因此也是最慢的。此级别存储audit级别中的所有信息以及所有其他可能的详细信息，主要是流程变量更新。

### 9.3 History for audit purposes

至少配置为audit级别时，所有通过方法`FormService.submitStartFormData(String processDefinitionId, Map<String, String> properties)`和`FormService.submitTaskFormData(String taskId, Map<String，String> properties)提交的所有属性将被记录。

可以使用查询API检索表单属性，如下所示：

```java
historyService
      .createHistoricDetailQuery()
      .formProperties()
      ...
      .list();
```

在这种情况下，只返回HistoricFormProperty类型的历史细节。

如果您在调用提交方法之前使用`IdentityService.setAuthenticatedUserId(String)`设置了经过身份验证的用户，那么提交表单的验证用户也可以在历史记录中访问，并且可以使用`HistoricProcessInstance.getStartUserId()`获取开始表单和`HistoricActivityInstance.getAssignee()`获取任务表格。