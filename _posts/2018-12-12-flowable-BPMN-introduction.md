---
layout: post
title: "Flowable文档学习笔记--BPMN介绍"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 6. BPMN2.0介绍

### 6.1 Defining a process

BPMN2.0模式的根元素是definitions元素。在这个元素中，可以给出多个流程定义（尽管我们的建议是每个文件中只有一个流程定义，因为这样可以在开发过程的后期简化维护）。空流程定义如下所示。请注意，最简单的definitions元素只需要xmlns和targetNamespace声明。targetNamespace可以是任何内容，主要用于流程定义分类。

```xml
<definitions
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:flowable="http://flowable.org/bpmn"
  targetNamespace="Examples">

  <process id="myProcess" name="My First Process">
    ..
  </process>

</definitions>
```

process元素包含两个属性：
- `id`：此属性是**必需**的，并映射到Flowable中ProcessDefinition对象的key属性。然后，可以使用此id通过RuntimeService上的startProcessInstanceByKey方法启动流程定义的新流程实例。此方法将始终采用流程定义的最新部署版本。`ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("myProcess");`
- 需要注意的是，这与调用startProcessInstanceById方法不同，后者需要Flowable引擎在部署时生成的String ID（可以通过调用processDefinition.getId()方法来检索ID）。生成的ID的格式为**key:version**，长度约束为**64个字符**。如果您收到FlowableException，指出生成的ID太长，请限制进程的键字段中的文本。
- `name`：此属性是可选的，并映射到ProcessDefinition的name属性。引擎本身不使用此属性，因此它可用于在用户界面中显示更人性化的名称。

### 6.2 简单快速教程

#### 6.2.1 用例介绍

用例很简单：我们有一家公司，我们称之为BPMCorp。在BPMCorp，每个月都需要为公司股东撰写财务报告，这是会计部门的责任。报告完成后，高层管理人员中的一位成员需要在将文件发送给所有股东之前批准该文件。

流程图如下所示，包含一个None Start Event，接着两个User Tasks，接触为none end event。其中第一个任务分配给accountancy团队，第二个任务分配给management团队。示例如下所示：

```xml
<definitions id="definitions"
  targetNamespace="http://flowable.org/bpmn20"
  xmlns:flowable="http://flowable.org/bpmn"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">

    <process id="financialReport" name="Monthly financial report reminder process">

      <startEvent id="theStart" />

      <sequenceFlow id="flow1" sourceRef="theStart" targetRef="writeReportTask" />

      <userTask id="writeReportTask" name="Write monthly financial report" >
        <documentation>
          Write monthly financial report for publication to shareholders.
        </documentation>
        <potentialOwner>
          <resourceAssignmentExpression>
            <formalExpression>accountancy</formalExpression>
          </resourceAssignmentExpression>
        </potentialOwner>
      </userTask>

      <sequenceFlow id="flow2" sourceRef="writeReportTask" targetRef="verifyReportTask" />

      <userTask id="verifyReportTask" name="Verify monthly financial report" >
        <documentation>
          Verify monthly financial report composed by the accountancy department.
          This financial report is going to be sent to all the company shareholders.
        </documentation>
        <potentialOwner>
          <resourceAssignmentExpression>
            <formalExpression>management</formalExpression>
          </resourceAssignmentExpression>
        </potentialOwner>
      </userTask>

      <sequenceFlow id="flow3" sourceRef="verifyReportTask" targetRef="theEnd" />

      <endEvent id="theEnd" />

    </process>

</definitions>
```

#### 6.2.2 创建流程实例

我们现在已经为业务流程创建了流程定义。从这样的流程定义中，我们可以创建流程实例。在此业务情景中，一个流程实例对应于特定月份的单个财务报告的创建和验证。任何月份的所有流程实例共享相同的流程定义。

为了能够从给定的流程定义创建流程实例，我们必须首先部署流程定义。部署流程定义意味着两件事：
- 流程定义将存储在为Flowable引擎配置的持久数据存储中。因此，通过部署我们的业务流程，我们确保引擎在引擎重启后找到流程定义。
- BPMN2.0流程XML将被解析为内存中的对象模型，该模型可以通过FlowableAPI进行操作。

示例如下，请注意，与Flowable引擎的所有交互都通过其服务进行。

```java
Deployment deployment = repositoryService.createDeployment()
  .addClasspathResource("FinancialReportProcess.bpmn20.xml")
  .deploy();
```

现在我们可以使用我们在流程定义中定义的id启动一个新的流程实例。请注意，Flowable术语中的此ID称为key。

```java
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("financialReport");
```

这将创建一个首先执行start事件的流程实例。在启动事件之后，它遵循所有的顺序流（在这种情况下只有一个）并且达到第一个任务（写月度财务报告）。Flowable引擎现在将任务存储在持久数据库中。此时，分配附加到任务中的用户或组将被解析并存储在数据库中。值得注意的是，Flowable引擎将继续执行流程步骤，直到达到等待状态，例如用户任务。在这种等待状态下，流程实例的当前状态存储在数据库中。在用户决定完成任务之前，它仍处于该状态。此时，引擎将继续运行，直到达到新的等待状态或进程结束。如果引擎在此期间重新启动或崩溃，则流程的状态在数据库中是安全可靠的。

创建任务后，startProcessInstanceByKey方法将返回，因为用户任务活动是等待状态。在我们的场景中，任务被分配给一个组，这意味着该组的每个成员都是执行该任务的候选者。

我们现在可以将所有这些放在一起并创建一个简单的Java程序。在我们调用Flowable服务之前，我们必须首先构建一个ProcessEngine，它允许我们访问服务。这里我们使用*standalone*配置，它构造一个ProcessEngine，它使用在演示设置中也使用的数据库。

您可以在此处下载流程定义XML。此文件包含上面显示的XML，但还包含必要的BPMN图交换信息，以便可视化Flowable工具中的过程。

```java
public static void main(String[] args) {

  // Create Flowable process engine
  ProcessEngine processEngine = ProcessEngineConfiguration
    .createStandaloneProcessEngineConfiguration()
    .buildProcessEngine();

  // Get Flowable services
  RepositoryService repositoryService = processEngine.getRepositoryService();
  RuntimeService runtimeService = processEngine.getRuntimeService();

  // Deploy the process definition
  repositoryService.createDeployment()
    .addClasspathResource("FinancialReportProcess.bpmn20.xml")
    .deploy();

  // Start a process instance
  runtimeService.startProcessInstanceByKey("financialReport");
}
```

#### 6.2.3 Tasks

**查询任务：**
- 通过用户查询：`List<Task> tasks = taskService.createTaskQuery().taskCandidateUser("kermit").list();`
- 通过组查询：`List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup("accountancy").list();`

**声明任务：**

通过声明任务，该指定用户将成为该任务的责任人，并且该任务将从该组的其他成员的每个任务列表中消失。声明任务是以编程方式完成的，如下所示：`taskService.claim(task.getId(), "fozzie");`现在任务属于指定用户的私人任务了：`List<Task> tasks = taskService.createTaskQuery().taskAssignee("fozzie").list();`。

**完成任务：**

现在可以开始编制财务报告。报告完成后，他可以完成任务，这意味着完成该任务的所有工作：
`taskService.complete(task.getId());`。

对于Flowable引擎，这是一个外部信号，流程实例现在可以继续执行。任务本身将从运行时数据中删除。将execution移动到第二个任务（“报告的验证”）。现在将使用与第一个任务描述的机制相同的机制来分配第二个任务，但差异很小，即任务将分配给管理组。

#### 6.2.4 结束流程
可以以与以前完全相同的方式检索和声明验证任务。完成第二个任务会将流程execution移至结束事件，从而完成流程实例。将从数据存储中删除流程实例和所有相关的运行时执行数据。

您还可以使用的编程方式以historyService验证进程是否已结束。

```java
HistoryService historyService = processEngine.getHistoryService();
HistoricProcessInstance historicProcessInstance =
historyService.createHistoricProcessInstanceQuery().processInstanceId(procId).singleResult();
System.out.println("Process instance end time: " + historicProcessInstance.getEndTime());
```

#### 6.2.5 代码

```java
public class TenMinuteTutorial {

  public static void main(String[] args) {

    // Create Flowable process engine
    ProcessEngine processEngine = ProcessEngineConfiguration
      .createStandaloneProcessEngineConfiguration()
      .buildProcessEngine();

    // Get Flowable services
    RepositoryService repositoryService = processEngine.getRepositoryService();
    RuntimeService runtimeService = processEngine.getRuntimeService();

    // Deploy the process definition
    repositoryService.createDeployment()
      .addClasspathResource("FinancialReportProcess.bpmn20.xml")
      .deploy();

    // Start a process instance
    String procId = runtimeService.startProcessInstanceByKey("financialReport").getId();

    // Get the first task
    TaskService taskService = processEngine.getTaskService();
    List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup("accountancy").list();
    for (Task task : tasks) {
      System.out.println("Following task is available for accountancy group: " + task.getName());

      // claim it
      taskService.claim(task.getId(), "fozzie");
    }

    // Verify Fozzie can now retrieve the task
    tasks = taskService.createTaskQuery().taskAssignee("fozzie").list();
    for (Task task : tasks) {
      System.out.println("Task for fozzie: " + task.getName());

      // Complete the task
      taskService.complete(task.getId());
    }

    System.out.println("Number of tasks for fozzie: "
            + taskService.createTaskQuery().taskAssignee("fozzie").count());

    // Retrieve and claim the second task
    tasks = taskService.createTaskQuery().taskCandidateGroup("management").list();
    for (Task task : tasks) {
      System.out.println("Following task is available for management group: " + task.getName());
      taskService.claim(task.getId(), "kermit");
    }

    // Completing the second task ends the process
    for (Task task : tasks) {
      taskService.complete(task.getId());
    }

    // verify that the process is actually finished
    HistoryService historyService = processEngine.getHistoryService();
    HistoricProcessInstance historicProcessInstance =
      historyService.createHistoricProcessInstanceQuery().processInstanceId(procId).singleResult();
    System.out.println("Process instance end time: " + historicProcessInstance.getEndTime());
  }

}
```