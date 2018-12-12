---
layout: post
title: "Flowable文档学习笔记--Deploy"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 5. Deploy

### 5.1 Business archives

要部署流程，必须将它们包装在业务存档（BAR）中。业务存档是Flowable引擎中的部署单位。业务存档等同于ZIP文件。它可以包含BPMN 2.0流程定义，表单定义，DMN规则和任何其他类型的文件。通常，业务存档包含一组命名资源。

当部署业务存档时，将扫描扩展名为.bpmn20.xml或.bpmn的BPMN文件。其中的每个都将被处理，并可能包含多个流程定义。激活DMN引擎时，还会解析.dmn文件，并在激活form引擎的情况下处理.form文件。

#### 5.1.1 Deploying programmatically

从ZIP文件部署一个业务归档的方法可如下所示：

```java
String barFileName = "path/to/process-one.bar";
ZipInputStream inputStream = new ZipInputStream(new FileInputStream(barFileName));

repositoryService.createDeployment()
    .name("process-one.bar")
    .addZipInputStream(inputStream)
    .deploy();
```

### 5.2 External resources

流程定义存在于Flowable数据库中。当使用Flowable配置文件中的ServiceTasks或execution listeners或Spring bean时，这些流程定义可以引用委托类。这些类和Spring配置文件必须对所有可执行流程定义的流程引擎可用。

#### 5.2.1 Java classes

当一个实例流程启动时，流程中使用的所有自定义类（例如，Service Tasks或事件侦听器中使用的JavaDelegates，TaskListeners等）都应该出现在引擎的类路径中。

但是，在部署业务存档期间，这些类不必存在于类路径中。这意味着，例如在使用Ant部署新的业务存档时，您的委托类不必位于类路径中。

当您使用演示设置并且想要添加自定义类时，应该将包含类的JAR添加到flowable-task或flowable-rest的webapp lib中。不要忘记包含自定义类的依赖项（如果有的话）。或者，您可以将依赖项包含在Tomcat安装的库目录中，${tomcat.home}/lib。

#### 5.2.2 Using Spring beans from a process

当表达式或脚本使用SpringBean，在执行流程定义时，这些bean必须可供引擎使用。如果您正在构建自己的webapp并按照Spring integration部分中的描述在上下文中配置流程引擎，那么这很简单。但请记住，在使用时，您还应该使用该上下文更新Flowable task和rest webapp。

#### 5.2.3 Creating a single app

可以考虑将Flowable REST Web应用程序包含在您自己的Web应用程序中，以确保只有一个ProcessEngine；而不是通过确保所有流程引擎都在其类路径上具有所有委托类并使用正确的Spring配置来实现。

### 5.3 Versioning of process definitions

BPMN没有版本控制的概念。这实际上很好，因为可执行的BPMN流程文件可能会作为开发项目的一部分存在于版本控制系统存储库（例如Subversion，Git或Mercurial）中。但是，其作为部署工作的一部分，会在引擎中创建流程定义的不同版本。在部署期间，Flowable将在ProcessDefinition存储到FlowableDB之前为其分配一个版本。

对于业务存档中的每个流程定义，执行以下步骤以初始化key，version，name，id属性：
- XML文件中的流程定义id属性用作流程定义的key属性。
- XML文件中的流程定义name属性用作流程定义name属性。如果未指定name属性，则将id属性用作name。
- 第一次部署具有特定key的流程时，将分配版本1，对于具有相同key的流程定义的所有后续部署，版本将设置为高于当前部署的最高版本1，key属性用于区分不同的流程定义。
- id属性被设置为{processDefinitionKey}:{processDefinitionVersion}:{generated-id}，其中generated-id是唯一编号，用于保证集群环境流程定义缓存中的流程ID唯一性。

### 5.4 Providing a process diagram

可以将process diagram图添加到部署中。此图将存储在Flowable存储库中，并可通过API访问。此图还用于可视化Flowable应用程序中的流程。

假设我们的类路径上有一个进程，org/flowable/expenseProcess.bpmn20.xml，它有一个process key *expense*。流程图应使用以下命名约定（按此特定顺序）：
- 如果部署中存在以**BPMN2.0 XML文件名+process key+图片后缀**为名称的图资源，则使用此图。在我们的示例中，这将是`org/flowable/expenseProcess.expense.png`（或.jpg / gif）。如果您在一个BPMN 2.0 XML文件中定义了多个图像，则此方法最有用。然后，每个图将在其文件名中包含process key。
- 如果不存在此类图片，则搜索与BPMN2.0 XML文件的名称相匹配的部署中的图片资源。在我们的示例中，这将是`org/flowable/expenseProcess.png`。请注意，这意味着在同一BPMN2.0文件中定义的**每个流程定义**都具有相同的流程图图像。

示例如下：

```java
repositoryService.createDeployment()
  .name("expense-process.bar")
  .addClasspathResource("org/flowable/expenseProcess.bpmn20.xml")
  .addClasspathResource("org/flowable/expenseProcess.png")
  .deploy();
```

```java
ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery()
  .processDefinitionKey("expense")
  .singleResult();

String diagramResourceName = processDefinition.getDiagramResourceName();
InputStream imageStream = repositoryService.getResourceAsStream(
    processDefinition.getDeploymentId(), diagramResourceName);
```

### 5.5 Category

部署和流程定义都具有用户定义的类别。使用BPMN XML中targetNamespace属性的值初始化流程定义类别：`<definitions ... targetNamespace =“yourCategory”...`，部署类别也可以在API中指定，如下所示：

```java
repositoryService
    .createDeployment()
    .category("yourCategory")
    ...
    .deploy();
```