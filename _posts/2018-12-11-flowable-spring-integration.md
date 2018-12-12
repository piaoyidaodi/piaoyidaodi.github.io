---
layout: post
title: "Flowable文档学习笔记--Spring Integration"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 4. Spring integration

### 4.1 ProcessEngineFactoryBean

ProcessEngine可以配置为常规Spring bean。集成的起点是org.flowable.spring.ProcessEngineFactoryBean类。该bean使用process engine configuration并创建流程引擎。这意味着Spring的属性创建和配置与普通配置相同。对于Spring集成，配置和引擎bean将如下所示，注意现在processEngineConfiguration使用org.flowable.spring.SpringProcessEngineConfiguration：

```xml
<bean id="processEngineConfiguration" class="org.flowable.spring.SpringProcessEngineConfiguration">
    ...
</bean>

<bean id="processEngine" class="org.flowable.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```

### 4.2 Transactions

我们将逐步解释Spring示例中的SpringTransactionIntegrationTest。下面是我们在此示例中使用的Spring配置文件（您可以在SpringTransactionIntegrationTest-context.xml中找到它）。下面显示的部分包含dataSource，transactionManager，processEngine和Flowable engine service。

当将DataSource传递给SpringProcessEngineConfiguration（使用属性“dataSource”）时，Flowable在内部使用org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy，它将包装传递的DataSource。这样做是为了确保从DataSource和Spring事务中检索到的SQL连接能够很好地协同工作。这意味着不再需要在Spring配置中自己代理dataSource，尽管仍然可以将TransactionAwareDataSourceProxy传递给SpringProcessEngineConfiguration。在这种情况下，不会发生额外的包装。

确保在Spring配置中自己声明TransactionAwareDataSourceProxy时，不要将它用于已经知道Spring事务的资源（例如，DataSourceTransactionManager和JPATransactionManager需要未代理的dataSource）。

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                             http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context
                             http://www.springframework.org/schema/context/spring-context-2.5.xsd
                           http://www.springframework.org/schema/tx
                             http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">

  <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
    <property name="driverClass" value="org.h2.Driver" />
    <property name="url" value="jdbc:h2:mem:flowable;DB_CLOSE_DELAY=1000" />
    <property name="username" value="sa" />
    <property name="password" value="" />
  </bean>

  <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
  </bean>

  <bean id="processEngineConfiguration" class="org.flowable.spring.SpringProcessEngineConfiguration">
    <property name="dataSource" ref="dataSource" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="databaseSchemaUpdate" value="true" />
    <property name="asyncExecutorActivate" value="false" />
  </bean>

  <bean id="processEngine" class="org.flowable.spring.ProcessEngineFactoryBean">
    <property name="processEngineConfiguration" ref="processEngineConfiguration" />
  </bean>

  <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService" />
  <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService" />
  <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService" />
  <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService" />
  <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService" />

...
```

该Spring配置文件的其余部分包含我们将在此特定示例中使用的bean和配置：

```xml
<beans>
  ...
  <tx:annotation-driven transaction-manager="transactionManager"/>

  <bean id="userBean" class="org.flowable.spring.test.UserBean">
    <property name="runtimeService" ref="runtimeService" />
  </bean>

  <bean id="printer" class="org.flowable.spring.test.Printer" />

</beans>
```

首先，使用Spring支持的任何方式创建应用程序上下文。 在此示例中，您可以使用类路径XML资源来配置Spring应用程序上下文：`ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("org/flowable/examples/spring/SpringTransactionIntegrationTest-context.xml");`；或作为一个测试`@ContextConfiguration("classpath:org/flowable/spring/test/transaction/SpringTransactionIntegrationTest-context.xml")`。

然后我们可以获取服务bean并在它们上调用方法。 ProcessEngineFactoryBean将向服务添加一个额外的拦截器，该服务在Flowable服务方法上应用Propagation.REQUIRED事务语义。因此，例如，我们可以使用repositoryService来部署这样的进程：

```java
RepositoryService repositoryService =
  (RepositoryService) applicationContext.getBean("repositoryService");
String deploymentId = repositoryService
  .createDeployment()
  .addClasspathResource("org/flowable/spring/test/hello.bpmn20.xml")
  .deploy()
  .getId();
```

在这种情况下，Spring事务将围绕userBean.hello()方法，Flowable服务方法调用将加入同一事务。

```java
UserBean userBean =(UserBean)applicationContext.getBean("userBean");
userBean.hello();
```

UserBean看起来像这样。请记住，在Spring bean配置中，我们将repositoryService注入userBean。

```java
public class UserBean {

  /** injected by Spring */
  private RuntimeService runtimeService;

  @Transactional
  public void hello() {
    // here you can do transactional stuff in your domain model
    // and it will be combined in the same transaction as
    // the startProcessInstanceByKey to the Flowable RuntimeService
    runtimeService.startProcessInstanceByKey("helloProcess");
  }

  public void setRuntimeService(RuntimeService runtimeService) {
    this.runtimeService = runtimeService;
  }
}
```

### 4.3 Expressions

使用ProcessEngineFactoryBean时，BPMN进程中的所有表达式默认情况下也会看到所有Spring bean。您可以通过配置映射来限制要向表达式中暴露的bean（甚至可以没有）。下面的示例暴露了一个bean（printer），可以在“printer”键下使用。要完全不暴露任何bean，只需向SpringProcessEngineConfiguration的beans属性传递**空列表**。如果未设置beans属性，则上下文中的所有Spring bean都可用。

```xml
<bean id="processEngineConfiguration" class="org.flowable.spring.SpringProcessEngineConfiguration">
  ...
  <property name="beans">
    <map>
      <entry key="printer" value-ref="printer" />
    </map>
  </property>
</bean>

<bean id="printer" class="org.flowable.examples.spring.Printer" />
```

现在可以在表达式中使用暴露的bean：例如，SpringTransactionIntegrationTest的hello.bpmn20.xml显示了如何使用UEL方法表达式调用Spring bean上的方法：

```xml
<definitions id="definitions">

  <process id="helloProcess">

    <startEvent id="start" />
    <sequenceFlow id="flow1" sourceRef="start" targetRef="print" />

    <serviceTask id="print" flowable:expression="#{printer.printMessage()}" />
    <sequenceFlow id="flow2" sourceRef="print" targetRef="end" />

    <endEvent id="end" />

  </process>

</definitions>
```

Printer类如下所示：

```java
public class Printer {

  public void printMessage() {
    System.out.println("hello world");
  }
}
```

Spring bean配置如下所示：

```xml
<beans>
  ...

  <bean id="printer" class="org.flowable.examples.spring.Printer" />

</beans>
```

### 4.4 Automatic resource deployment

Spring集成还具有部署资源的特殊功能。在流程引擎配置中，您可以指定一组资源。创建流程引擎后，将扫描和部署所有这些资源，筛选器可以防止重复部署。仅当资源实际更改时，才会将新deploy部署到FlowableDB。这在Spring容器经常重启的许多用例中是有意义的（例如，测试），比如：

```xml
<bean id="processEngineConfiguration" class="org.flowable.spring.SpringProcessEngineConfiguration">
  ...
  <property name="deploymentResources"
    value="classpath*:/org/flowable/spring/test/autodeployment/autodeploy.*.bpmn20.xml" />
</bean>

<bean id="processEngine" class="org.flowable.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```

默认情况下，上面的配置会将与筛选器匹配的所有资源分组到Flowable引擎的单个部署中，重复筛选器防止重新部署未更改资源。在某些情况下，这可能不是您想要的。例如，如果以这种方式部署一组流程资源，并且这些资源中只有一个流程定义发生了变化，那么整个部署将被视为新部署，并且将重新部署该部署中的所有流程定义。

为了能够自定义部署的确定方式，您可以在SpringProcessEngineConfiguration，deploymentMode中指定其他属性。此属性定义确定了与筛选器匹配的资源集的部署方式。默认情况下，此属性支持3个值：
- `default`：将所有资源分组到单个部署中，并对该部署应用重复过滤器，这是默认值。

- `signle-resource`：为每个单独的资源创建单独的部署，并对该部署应用重复过滤器。这是用于单独部署每个流程定义的值，如果更改后则仅创建一个新版本的流程定义。

- `resource-parent-folder`：为共享同一父文件夹的资源创建单独的部署，并对该部署应用重复过滤器。此值可用于为大多数资源创建单独的部署，但仍可以通过将它们放在共享文件夹中来对其进行分组。以下是如何为deploymentMode指定`signle-resource`配置的示例：

```xml
<bean id="processEngineConfiguration"
    class="org.flowable.spring.SpringProcessEngineConfiguration">
  ...
  <property name="deploymentResources" value="classpath*:/flowable/*.bpmn" />
  <property name="deploymentMode" value="single-resource" />
</bean>
```

除了使用上面列出的deploymentMode值之外，您还可能需要自定义行为来确定部署。如果是这样，您可以创建SpringProcessEngineConfiguration的子类并覆盖getAutoDeploymentStrategy(String deploymentMode)方法。此方法确定将哪个部署策略用于deploymentMode配置的特定值。

### 4.5 Spring Boot

Flowable支持Spring Boot 2.0和1.5，具有相同的启动器。基本支持适用于Spring Boot 2.0，这意味着仅在2.0上支持actuator endpoints。Flowable启动器逐步转移到spring boot启动器，这意味着用户必须在自己的构建文件中定义1.5版本的spring boot启动器。

#### 4.5.1 起始

可在maven中添加flowable-spring-boot-starter或flowable-spring-boot-starter-rest依赖，以及数据库：

```xml
<dependency>
    <groupId>org.flowable</groupId>
    <artifactId>flowable-spring-boot-starter</artifactId>
    <version>${flowable.version}</version>
</dependency>
<!-- h2,mysql -->
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <version>1.3.176</version>
</dependency>
```

SpringBoot应用示例如下：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

以上过程后台进行了如下操作：
- 自动创建内存数据源（因为H2驱动程序在classpath上）并传递给Flowable process engine configuration。
- 创建并暴露Flowable ProcessEngine，CmmnEngine，DmnEngine，FormEngine，ContentEngine和IdmEngine的bean。
- 所有Flowable服务都以Spring bean的形式暴露公开。
- Spring Job Executor被创建
- 将自动部署processes文件夹中的任何BPMN2.0流程定义文件。创建processes文件夹并将虚拟流程定义（名为one-task-process.bpmn20.xml）添加到此文件夹。
- 将自动部署cases文件夹中的任何CMMN1.1 case定义。
- 将自动部署dmn文件夹中的任何DMN1.1 dmn定义。
- 将自动部署forms文件夹中的任何Form定义。