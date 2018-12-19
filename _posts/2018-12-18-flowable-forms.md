---
layout: post
title: "Flowable文档学习笔记--Forms"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 8. Forms

Flowable提供了一种方便灵活的方式来为业务流程的手动步骤添加表单（form）。我们支持两种使用表单的策略：带有表单定义（使用表单设计器创建）的内部表单和外部表单。对于外部表单策略，可以使用表单属性（在版本5中的ExplorerWeb应用程序中支持），或者是指向可以使用自定义编码解析的外部表单引用的表单键定义。

### 8.1 Form definition

有关表单定义和Flowable表单引擎的完整信息，请参阅FormEngine用户指南。可以使用FlowableFormDesigner创建表单定义，该FlowableFormDesigner是FlowableModelerWeb应用程序的一部分，也可以通过JSON编辑器手动创建。FormEngine用户指南以全长形式描述了表单定义JSON的结构。支持以下表单字段类型：
- Text：呈现为文本字段
- Multiline text：呈现为文本区域字段
- Number：呈现为文本字段，但仅允许数字值
- Checkbox：呈现为复选框字段
- Date：呈现为日期字段
- Dropdown：呈现为选择字段，并在字段定义中配置选项值
- Radio buttons：呈现为单选字段，并且在字段定义中配置的选项值
- People：呈现为选择字段，可以选择已验证用户表中的人员
- Group of people：呈现为选择字段，其中可以选择已验证群组表中的组
- Upload：呈现为上传字段
- Expression：呈现为标签，允许您使用JUEL表达式在标签文本中使用变量和/或其他动态值

Flowable任务应用程序能够从表单定义JSON中渲染出html表单。您还可以使用FlowableAPI自行获取表单定义JSON。如`	FormModel RuntimeService.getStartFormModel(String processDefinitionId, String processInstanceId)`或`FormModel TaskService.getTaskFormModel(String taskId)`。

FormModel对象是一个可以代表表单定义JSON的Java对象。

为了使用表达定义启动流程实例，可使用API：`ProcessInstance RuntimeService.startProcessInstanceWithForm(String processDefinitionId, String outcome, Map<String, Object> variables, String processInstanceName)`。当在流程定义的（一个）起始事件上定义表单定义时，此方法可用于启动流程实例，并在起始表单中填入值。Flowable任务应用程序使用此方法也可以使用表单启动流程实例。所有表单值都需要在变量映射中传递，并且可以提供可选的表单结果字符串和流程实例名称。

以类似的方式，可以使用以下API调用使用表单完成用户任务：`void TaskService.completeTaskWithForm(String taskId, String formDefinitionId, String outcome, Map<String, Object> variables)`。

### 8.2 Form properties

与业务流程相关的所有信息都包含在流程变量本身中或通过流程变量引用。Flowable支持将复杂的Java对象存储为流程变量，如Serializable对象，JPA实体或将整个XML文档作为String。

启动流程和完成用户任务是使用人参与流程的地方。与人沟通需要在某些UI技术中呈现表单。为了简化多个UI技术，流程定义包含了可以将流程变量中复杂的Java类型对象转换为Map<String，String>属性的操作逻辑。

然后，任何UI技术都可以使用FlowableAPI方法在这些属性之上构建表单并公开属性信息。这些属性可以为过程变量提供专用（且更有限）的视图。例如，FormData返回值中提供了显示表单所需的属性，`StartFormData FormService.getStartFormData(String processDefinitionId)`或`TaskFormdata FormService.getTaskFormData(String taskId)`

默认情况下，内置表单引擎会查看属性以及流程变量。因此，如果任务表单属性与流程变量匹配1-1，则无需声明它们。例如，使用以下声明：`<startEvent id ="start"/>`，当execution到达startEvent时，所有进程变量都可用，但是`formService.getStartFormData(String processDefinitionId).getFormProperties()`因为没有定义特定的映射，所以将为空。

在上述情况下，所有提交的属性都将存储为流程变量。这意味着只需在表单中添加新的输入字段，就可以存储新的变量。

属性是从流程变量派生的，但它们并非必须必存储为流程变量。例如，流程变量可以是类Address的JPA实体。UI技术使用的表单属性StreetName可以与表达式`#{address.street}`链接。

类似地，用户应该在表单中提交的属性可以作为流程变量或嵌套属性存储在其中一个流程变量中，其具有UEL值表达式，例如， `#{address.street}`。

类似地，提交属性的默认行为是作为流程变量存储，除非另外指定formProperty声明。类型转换也可以作为表单属性和流程变量之间处理的一部分来应用。如

```xml
<userTask id="task">
  <extensionElements>
    <flowable:formProperty id="room" />
    <flowable:formProperty id="duration" type="long"/>
    <flowable:formProperty id="speaker" variable="SpeakerName" writable="false" />
    <flowable:formProperty id="street" expression="#{address.street}" required="true" />
  </extensionElements>
</userTask>
```

- 表单属性room将作为字符串映射到流程变量room
- 表单属性duration将作为java.lang.Long映射到流程变量duration
- 表单属性speaker将映射到流程变量SpeakerName。它只能在TaskFormData对象中使用。如果提交了属性speaker，则会抛出FlowableException。类似于属性readable="false"，属性可以从FormData中排除，但仍然可以在提交中处理。
- 表单属性street将作为String映射到流程变量中的Javabean属性street中。如果未提供属性，则required="true"将在提交期间抛出异常。

也可以提供类型元数据作为从方法`StartFormData FormService.getStartFormData(String processDefinitionId)`和`TaskFormdata FormService.getTaskFormData(String taskId)`返回的FormData的一部分。

我们支持以下表单属性类型：
- string（org.flowable.engine.impl.form.StringFormType）
- long（org.flowable.engine.impl.form.LongFormType）
- double（org.flowable.engine.impl.form.DoubleFormType）
- enum（org.flowable.engine.impl.form.EnumFormType）
- date（org.flowable.engine.impl.form.DateFormType）
- boolean（org.flowable.engine.impl.form.BooleanFormType）

对于声明的每个表单属性，将通过`List<FormProperty> formService.getStartFormData(String processDefinitionId).getFormProperties()`和`List<FormProperty> formService.getTaskFormData(String taskId).getFormProperties()`提供以下FormProperty信息。

```java
public interface FormProperty {
  /** the key used to submit the property in {@link FormService#submitStartFormData(String, java.util.Map)}
   * or {@link FormService#submitTaskFormData(String, java.util.Map)} */
  String getId();
  /** the display label */
  String getName();
  /** one of the types defined in this interface like e.g. {@link #TYPE_STRING} */
  FormType getType();
  /** optional value that should be used to display in this property */
  String getValue();
  /** is this property read to be displayed in the form and made accessible with the methods
   * {@link FormService#getStartFormData(String)} and {@link FormService#getTaskFormData(String)}. */
  boolean isReadable();
  /** is this property expected when a user submits the form? */
  boolean isWritable();
  /** is this property a required input field */
  boolean isRequired();
}
```

```xml
<startEvent id="start">
  <extensionElements>
    <flowable:formProperty id="speaker"
      name="Speaker"
      variable="SpeakerName"
      type="string" />

    <flowable:formProperty id="start"
      type="date"
      datePattern="dd-MMM-yyyy" />

    <flowable:formProperty id="direction" type="enum">
      <flowable:value id="left" name="Go Left" />
      <flowable:value id="right" name="Go Right" />
      <flowable:value id="up" name="Go Up" />
      <flowable:value id="down" name="Go Down" />
    </flowable:formProperty>

  </extensionElements>
</startEvent>
```

所有这些信息都可以通过API访问。可以使用`formProperty.getType().getName()`获取类型名称。甚至日期模式也可以使用`formProperty.getType().getInformation("datePattern")`，并且可以使用`formProperty.getType().getInformation("values")`访问枚举值.

以下XML片段可在自定义app中用于呈现流程开始：
```xml
<startEvent>
  <extensionElements>
    <flowable:formProperty id="numberOfDays" name="Number of days" value="${numberOfDays}" type="long" required="true"/>
    <flowable:formProperty id="startDate" name="First day of holiday (dd-MM-yyy)" value="${startDate}" datePattern="dd-MM-yyyy hh:mm" type="date" required="true" />
    <flowable:formProperty id="vacationMotivation" name="Motivation" value="${vacationMotivation}" type="string" />
  </extensionElements>
</userTask>
```

### 8.3 External form rendering

API还允许您在FlowableEngine外部执行自己的任务表单渲染。

本质上，呈现表单所需的所有数据都是在以下两种服务方法之一中组合的：`StartFormData FormService.getStartFormData(String processDefinitionId)`和`TaskFormdata FormService.getTaskFormData(String taskId)`。

提交表单属性可以使用`ProcessInstance FormService.submitStartFormData(String processDefinitionId, Map<String, String> properties)`和`void FormService.submitTaskFormData(String taskId, Map<String, String> properties)`完成。

要了解表单属性如何映射到流程变量，请参阅表单属性。

您可以将任何表单模板资源放在您部署的业务档案中（如果您希望将它们与该流程存储版本化）。它将作为部署中的资源提供，您可以使用以下命令检索：`String ProcessDefinition.getDeploymentId()`和`InputStream RepositoryService.getResourceAsStream(String deploymentId, String resourceName)`；这可以是您的模板定义文件，您可以使用它来在自己的应用程序中呈现/显示表单。

您还可以使用此功能来访问任务表单之外的部署资源以用于任何其他目的。

API通过`String FormService.getStartFormData(String processDefinitionId).getFormKey()`和`String FormService.getTaskFormData(String taskId).getFormKey()`公开属性`<userTask flowable：formKey = "..."`。您可以使用它来存储部署中模板的全名（例如org/flowable/example/form/my-custom-form.xml），但这根本不需要。例如，您还可以在表单属性中存储通用键，并应用算法或转换以获取需要使用的实际模板。当您想要为不同的UI技术呈现不同的表单时，这可能很方便。如，一种用于正常屏幕大小的Web应用程序的表单，一种用于移动电话的小屏幕的表单，甚至可以用于IM表单或电子邮件表单的模板。