---
layout: post
title: "Flowable文档学习笔记————API"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 3. Flowable API

### 3.1 ProcessEngine API和服务

从ProcessEngine中可获取多种包含workflow/BPM方法的服务，ProcessEngine是线程安全的。

![Flowable API](/assets/img/flowable/201812111_flowable_api_services.png)

```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();

RuntimeService runtimeService = processEngine.getRuntimeService();
RepositoryService repositoryService = processEngine.getRepositoryService();
TaskService taskService = processEngine.getTaskService();
ManagementService managementService = processEngine.getManagementService();
IdentityService identityService = processEngine.getIdentityService();
HistoryService historyService = processEngine.getHistoryService();
FormService formService = processEngine.getFormService();
DynamicBpmnService dynamicBpmnService = processEngine.getDynamicBpmnService();
```

ProcessEngines.getDefaultProcessEngine()将在第一次调用时初始化并构建流程引擎，之后始终返回相同的流程引擎。可以使用ProcessEngines.init()和ProcessEngines.destroy()来正确创建和关闭所有流程引擎。

ProcessEngines类将扫描所有flowable.cfg.xml和flowable-context.xml文件。对于所有flowable.cfg.xml文件，流程引擎将以典型的Flowable方式构建：`ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream).buildProcessEngine()`。对于所有flowable-context.xml文件，流程引擎将以Spring方式构建：首先创建Spring应用程序上下文，然后从该应用程序上下文获取流程引擎。

所有服务都是无状态的。这意味着您可以轻松地在集群中的多个节点上运行Flowable，每个节点都转到同一个数据库，而不必担心哪台计算机实际执行了之前的调用。无论在何处执行，任何对任意服务的调用都是幂等的。

**RepositoryService**

RepositoryService可能是使用Flowable引擎时所需的第一个服务。此服务提供用于deployments和process definitions的管理和操作。process definition是BPMN2.0流程的Java对应物。它表示流程的每个步骤结构和行为。deployment是Flowable引擎中的包装单位。deployment可以包含多个BPMN2.0 XML文件和任何其他资源。在一个部署中选择包含的内容取决于开发人员。它可以从单个流程BPMN2.0 XML文件到整个流程包和相关资源（例如，部署hr-processes可以包含与HR流程相关的所有内容）。RepositoryService可以deploy此类包。进行部署意味着将其上载到引擎，在引擎中，所有流程在存储到数据库之前都会进行检查和解析。从那时起，系统就知道部署，现在可以启动部署中包含的任何进程。

此外，此服务允许您：
- 查询引擎已知的部署和流程定义。
- 暂停和激活整个或特定流程定义的部署。暂停意味着不能对它们执行进一步的操作，而激活则相反并且再次启用操作。
- 检索各种资源，例如部署中包含的文件或引擎自动生成的流程图。
- 检索流程定义的POJO版本，该版本可用于使用Java而不是XML来对应流程。

**RuntimeService**

RepositoryService主要是静态信息（数据不会改变，或者至少不是很多），但RuntimeService恰恰相反。它涉及启动流程定义的新流程实例。如上所述，process definition定义流程中不同步骤的结构和行为。流程实例是这种流程定义的一次执行。对于每个流程定义，通常会有许多实例同时运行。RuntimeService也是用于检索和存储流程变量的服务，这是特定于给定流程实例的数据，并且可以由流程中的各种构造使用（例如，决策门通常使用流程变量来确定选择哪个路径来继续流程）。Runtimeservice还允许您查询流程实例和execution，execution是BPMN2.0中“token”概念的表示。基本上，execution是指向流程实例当前所在位置的指针。最后，只要流程实例正在等待外部触发器并且需要继续该流程，就会继续使用RuntimeService。流程实例可以具有各种wait states，并且该服务包含各种操作以向实例发信号，通知实例已接收到外部触发并且可以继续流程实例。

**TaskService**

需要由系统中的人类用户执行的Task是BPM引擎（如Flowable）的核心。围绕任务的所有内容都在TaskService中进行分组，例如：
- 查询分配给用户或组的任务
- 创建新的独立任务，这些是与流程实例无关的任务。
- 操作分配任务的用户或以某种方式参与任务的用户。
- claiming并completing任务。claiming意味着决定某人负责该任务并将完成该任务。completing意味着完成任务的工作。通常，这是填写各种表格。

**IdentityService/FormService/HistoryService/ManagementService/DynamicBpmnService**

IdentityService非常简单。它支持组和用户的管理（创建，更新，删除，查询等）。重要的是要了解Flowable实际上不会在运行时对用户进行任何检查。例如，可以将任务分配给任何用户，但引擎不会验证该用户是否为系统所知。这是因为Flowable引擎还可以与LDAP，Active Directory等服务结合使用。

FormService是一项可选服务。这意味着无此项服务，Flowable也可以正常顺利工作且无任何功能丧失。该服务引入了开始表单和任务表单的概念。开始表单是在启动流程实例之前向用户显示的表单，而任务表单是用户想要填终结form时显示的表单。Flowable允许在BPMN2.0流程定义中指定这些表单，此服务以简单的方式提供此数据。但同样这是可选的，因为表单不需要嵌入到流程定义中。

HistoryService提供Flowable引擎收集的所有历史数据。在执行流程时，引擎可以保存大量数据（这是可配置的），例如流程实例启动时间，哪些人执行了哪些任务，完成任务所需的时间，每个流程实例中遵循的路径，等等。此服务主要提供查询功能以访问此数据。

当使用Flowable编写自定义应用程序时，通常不需要ManagementService。它允许检索有关的数据库表和表元数据的信息。此外，它还公开了作业的查询功能和管理操作。Flowable中的作业用于各种事物，例如计时器，异步延续，延迟暂停/激活等。稍后，将更详细地讨论这些主题。

DynamicBpmnService可用于更改process definition的一部分，而无需重新部署它。例如，您可以更改流程定义中的用户任务受理人，或更改服务任务的类名称。

有关服务操作和引擎API的更多详细信息，请参阅 [javadoc](https://www.flowable.org/docs/javadocs/index.html) 。

### 3.2 查询API

有两种方法可以从引擎查询数据：查询API和本地查询。Query API允许您使用流畅的API完全通过编程进行类型安全的查询。您可以为查询添加各种条件（所有这些条件作为逻辑AND一起应用）并精确排序。 以下代码显示了一个示例：

```java
List<Task> tasks = taskService.createTaskQuery()
    .taskAssignee("kermit")
    .processVariableValueEquals("orderId", "0815")
    .orderByDueDate().asc()
    .list();
```

有时您需要更强大的查询，例如，使用OR运算符的查询或使用Query API无法进行表达。对于这些情况，我们有本地查询，允许您编写自己的SQL查询。返回类型由您使用的Query对象定义，数据映射到正确的对象（Task，ProcessInstance，Execution等）。由于查询将在数据库中触发，因此必须使用在数据库中定义的表名和列名；这需要一些有关内部数据结构的知识，建议谨慎使用本机查询。可以通过API检索表名，以使依赖性尽可能小。

```java
List<Task> tasks = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) +
      " T WHERE T.NAME_ = #{taskName}")
  .parameter("taskName", "gonzoTask")
  .list();

long count = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T1, " +
      managementService.getTableName(VariableInstanceEntity.class) + " V1 WHERE V1.TASK_ID_ = T1.ID_")
  .count();
```

### 3.3 Variables

每个流程实例都需要并使用数据来执行步骤。在Flowable中，此数据称为变量(variables)并存储在数据库中。变量可以在表达式中使用（例如，在决策门中选择正确的传出顺序流），在负责调用外部服务的Java服务任务中（例如，提供输入或存储服务调用的结果）等等。

流程实例可以包含变量（称为流程变量），执行（executions）（指向流程活动位置的特定指针）、用户任务可以包含变量。流程实例可以包含任意数量的变量。每个变量都存储在`ACT_RU_VARIABLE`数据库表的一行中。

所有startProcessInstanceXXX方法都有一个可选参数，用于在创建和启动流程实例时提供变量。例如，从RuntimeService：`ProcessInstance startProcessInstanceByKey(String processDefinitionKey, Map<String, Object> variables)`。变量也可以在流程执行时加入，如：

```java
void setVariable(String executionId, String variableName, Object value);
void setVariableLocal(String executionId, String variableName, Object value);
void setVariables(String executionId, Map<String, ? extends Object> variables);
void setVariablesLocal(String executionId, Map<String, ? extends Object> variables);
```

请注意，可以为给定的execution设置变量为本地变量（local）（请记住，流程实例由execution树组成）。该变量仅在此execution可见，而在execution树较高部分可不见。如果数据不应传播到流程实例级别，或者变量在流程实例中具有某个路径的新值（例如，使用并行路径时），则此操作非常有用。

变量还可以检索，如下所示。请注意，TaskService上存在类似的方法。这意味着任务（task）同execution一样，可以具有仅在任务期间存活的局部变量。

```java
Map<String, Object> getVariables(String executionId);
Map<String, Object> getVariablesLocal(String executionId);
Map<String, Object> getVariables(String executionId, Collection<String> variableNames);
Map<String, Object> getVariablesLocal(String executionId, Collection<String> variableNames);
Object getVariable(String executionId, String variableName);
<T> T getVariable(String executionId, String variableName, Class<T> variableClass);
```

变量（包括本地变量）通常用于Java委托，表达式，execution或任务监听器，脚本等。在这些构造中，当前execution或task对象是可用的，并且它可以用于变量设置和/或检索。 最简单的方法是这些：

```java
execution.getVariables();
execution.getVariables(Collection<String> variableNames);
execution.getVariable(String variableName);

execution.setVariables(Map<String, object> variables);
execution.setVariable(String variableName, Object value);
```

对于历史数据（以及向后兼容性原因），在执行上述任何调用时，将在后台从数据库中获取**所有变量**。这意味着如果你有10个变量，但只通过getVariable("myVariable")得到一个变量，那么在后台其他9个变量将被获取和缓存。这不一定是坏事，因为后续调用不会再次访问数据库。例如，当您的流程定义具有三个顺序服务任务（以及一个数据库事务）时，在第一个服务任务中使用一个调用来获取的所有变量可能会更好。请注意，这对获取和设置变量都适用。

当然，这种操作在使用大量变量时或者只是在想要严格控制数据库查询和流量时是不合适的。通过添加具有可选参数的新方法来对此操作进行更严格的控制，这些方法告诉引擎是否获取并缓存所有变量：

```java
Map<String, Object> getVariables(Collection<String> variableNames, boolean fetchAllVariables);
Object getVariable(String variableName, boolean fetchAllVariables);
void setVariable(String variableName, Object value, boolean fetchAllVariables);
```

当对参数fetchAllVariables使用true时，行为将完全如上所述：获取或设置变量时，将获取和缓存所有其他变量。

但是，当使用false作为值时，将使用特定查询，并且不会提取或缓存其他变量。只有此处使用的变量的值才会被缓存以供后续使用。

### 3.4 Transient variables

瞬态变量是行为类似于常规变量的变量，但不是持久变量。通常，瞬态变量用于高级用例。以下适用于瞬态变量：
- 瞬态变量根本没有存储历史记录。
- 与常规变量一样，瞬态变量在设置时放在最高父级上。这意味着在execution上设置变量时，瞬态变量实际上存储在流程实例的execution中。与常规变量一样，如果在特定执行或任务上设置变量，则存在方法的局部变量。
- 瞬态变量只能在流程定义中的下一个wait state之前访问。在那之后，瞬态变量消失。在这里，wait state表示流程实例中持久存储到数据库的点。请注意，异步活动在此定义中也是wait state！
- 瞬态变量只能通过setTransientVariable(name，value)设置，但在调用getVariable(name)时也会返回瞬态变量（存在getTransientVariable(name)但只检查瞬态变量）。这样做的原因是使用变量时表达式的编写变得容易，并且现有的逻辑可适用于这两种类型。
- 瞬态变量会覆盖（shadows）具有相同名称的持久变量。这意味着当在流程实例上设置持久变量和瞬态变量并且调用getVariable("someVariable")时，将返回瞬态变量值。

您可以在使用常规变量的大多数地方设置和获取瞬态变量：
- 在JavaDelegate实现中的DelegateExecution
- 在ExecutionListener实现中的DelegateExecution和在TaskListener实现上的DelegateTask上
- 通过execution对象在脚本任务中
- 通过运行时服务启动流程实例时
- 完成任务时
- 调用runtimeService.trigger方法时

这些方法遵循常规流程变量的命名约定：

```java
void setTransientVariable(String variableName, Object variableValue);
void setTransientVariableLocal(String variableName, Object variableValue);
void setTransientVariables(Map<String, Object> transientVariables);
void setTransientVariablesLocal(Map<String, Object> transientVariables);

Object getTransientVariable(String variableName);
Object getTransientVariableLocal(String variableName);

Map<String, Object> getTransientVariables();
Map<String, Object> getTransientVariablesLocal();

void removeTransientVariable(String variableName);
void removeTransientVariableLocal(String variableName);
```

### 3.5 Expressions

Flowable使用UEL进行表达式解析。 UEL代表Unified Expression Language，是EE6规范的一部分（有关详细信息，请参阅EE6规范）。

表达式可用于，例如Java Sercice tasks，Execution Listeners，Task Listeners和Conditional sequence flows。尽管有两种类型的表达式，值表达式和方法表达式，但Flowable对此进行了抽象，因此它们都可以在需要expression的地方使用。
- 值表达式：解析为值。默认情况下，可以使用所有流程变量。此外，所有spring-beans（如果使用Spring）都可以在表达式中使用。一些例子：`${myVar}``${myBean.myProperty} `。
- 方法表达式：调用带或不带参数的方法。在调用不带参数的方法时，请务必在方法名称后面添加括号以将方法表达式与值表达式区分开来。传递的参数可以是文字值或自己解析的表达式。如，`${printer.print()}``${myBean.addNewOrder('ORDERNAME')}``${myBean.doSomething(myVar，execution)}`。

请注意，这些表达式支持解析primitive（包括比较它们），bean，lists，arrays和maps。

除了所有流程变量之外，还有一些默认可用于表达式的对象：
- `execution`：DelegateExecution包含有关正在进行execution的其他信息。
- `task`：DelegateTask包含有关当前任务的其他信息。注意：仅适用于在任务监听器中计算的表达式。
- `authenticatedUserId`：当前已通过身份验证的用户的ID。如果没有用户通过身份验证，则该变量不可用。
