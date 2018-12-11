---
layout: post
title: "Flowable文档学习笔记————Configuration"
categories: Flowable
tag: Java-Framework
---
> 基于Flowable 6.4.0的重点文档学习和翻译。

## 2. Configuration

### 2.1 创建ProcessEngine和ProcessEngineConfiguration

Flowable通常通过`flowable.cfg.xml`文件配置（未使用Spring时）。通过`ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine()`获得ProcessEngine时将自动在classpath内寻找`flowable.cfg.xml`文件。其中`flowable.cfg.xml`中必须包含**id**为`processEngineConfiguration`的字段，并指定class。示例配置文件如下所示：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.flowable.engine.impl.cfg.StandaloneProcessEngineConfiguration">

    <property name="jdbcUrl" value="jdbc:h2:mem:flowable;DB_CLOSE_DELAY=1000" />
    <property name="jdbcDriver" value="org.h2.Driver" />
    <property name="jdbcUsername" value="sa" />
    <property name="jdbcPassword" value="" />

    <property name="databaseSchemaUpdate" value="true" />

    <property name="asyncExecutorActivate" value="false" />

    <property name="mailServerHost" value="mail.my-corp.com" />
    <property name="mailServerPort" value="5025" />
  </bean>

</beans>
```

`ProcessEngineConfiguration`对象可通过**配置文件**和**使用内置配置**方式获取，示例如下：

```java
// 配置文件方式
ProcessEngineConfiguration.
  createProcessEngineConfigurationFromResourceDefault();
  createProcessEngineConfigurationFromResource(String resource);
  createProcessEngineConfigurationFromResource(String resource, String beanName);
  createProcessEngineConfigurationFromInputStream(InputStream inputStream);
  createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName);

// 内部支持的配置

//标准配置，支持事务，引擎启动时检查数据库，如果数据库版本不对则抛出异常。
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration();
//多用于单元测试，支持事务，使用H2内存数据库，该数据库在引擎启动关闭时创建和删除。
ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration();
//Spring环境下的配置
ProcessEngineConfiguration.createSpringProcessEngineConfiguration();
//标准模式下的JTA事务
ProcessEngineConfiguration.createJtaProcessEngineConfiguration();
```

### 2.2 数据库配置

重点的JDBC数据库属性，将使用MyBatis的默认连接池设定：
- `jdbcUrl`: JDBC URL of the database.
- `jdbcDriver`: implementation of the driver for the specific database type.
- `jdbcUsername`: username to connect to the database.
- `jdbcPassword`: password to connect to the database.

选择配置的JDBC数据库属性：
- `jdbcMaxActiveConnections`: The number of active connections that the connection pool at maximum at any time can contain. Default is 10.
- `jdbcMaxIdleConnections`: The number of idle connections that the connection pool at maximum at any time can contain.
- `jdbcMaxCheckoutTime`: The amount of time in milliseconds a connection can be checked out from the connection pool before it is forcefully returned. Default is 20000 (20 seconds).
- `jdbcMaxWaitTime`: This is a low level setting that gives the pool a chance to print a log status and re-attempt the acquisition of a connection in the case that it is taking unusually long (to avoid failing silently forever if the pool is misconfigured) Default is 20000 (20 seconds).

也可考虑使用`javax.sql.DataSource`的实现并放入配置文件，如下所示：

```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
  <property name="driverClassName" value="com.mysql.jdbc.Driver" />
  <property name="url" value="jdbc:mysql://localhost:3306/flowable" />
  <property name="username" value="flowable" />
  <property name="password" value="flowable" />
  <property name="defaultAutoCommit" value="false" />
</bean>

<bean id="processEngineConfiguration" class="org.flowable.engine.impl.cfg.StandaloneProcessEngineConfiguration">

  <property name="dataSource" ref="dataSource" />
  ...
```

JDBC和data source两种方式都可使用如下配置：

- `databaseType`: it’s normally not necessary to specify this property, as it is automatically detected from the database connection metadata. Should only be specified when automatic detection fails. Possible values: {h2, mysql, oracle, postgres, mssql, db2}. This setting will determine which create/drop scripts and queries will be used. See the supported databases section for an overview of which types are supported.
- `databaseSchemaUpdate`: sets the strategy to handle the database schema on process engine boot and shutdown.
**false (default)**: Checks the version of the DB schema against the library when the process engine is being created and throws an exception if the versions don’t match.**true**: Upon building the process engine, a check is performed and an update of the schema is performed if it is necessary. If the schema doesn’t exist, it is created.**create-drop**: Creates the schema when the process engine is being created and drops the schema when the process engine is being closed.

### 2.3 JNDI数据源配置

JNDI数据源的配置与不同的servlet容器应用相关，教程主要介绍Tomcat应用服务器。

如果使用Tomcat，JNDI数据源应配置在`$CATALINA_BASE/conf/[enginename]/[hostname]/[warname].xml `中，对于FlowableUI通常在`$CATALINA_BASE/conf/Catalina/localhost/flowable-app.xml`中。当应用第一次部署时，默认的上下文是从Flowable的WAR文件中复制的，所以如果文件存在，应当进行更换。如下所示更换H2数据库为Mysql。

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <Context antiJARLocking="true" path="/flowable-app">
        <Resource auth="Container"
            name="jdbc/flowableDB"
            type="javax.sql.DataSource"
            description="JDBC DataSource"
            url="jdbc:mysql://localhost:3306/flowable"
            driverClassName="com.mysql.jdbc.Driver"
            username="sa"
            password=""
            defaultAutoCommit="false"
            initialSize="5"
            maxWait="5000"
            maxActive="120"
            maxIdle="5"/>
        </Context>
```

### 2.4 创建数据库表

Flowable中的数据库表创建较为简单，配置后执行DbSchemaCreate类的主方法。

Flowable中的SQL DDL（数据库定义语言）可以在下载页、发布文件夹的database子文件夹、JAR文件中的org/flowable/db/create包内找到，其命名规范为`flowable.{db}.{create|drop}.{type}.sql`。其中db为支持的数据库名；type为engine或history。

**MySQL**注意事项：

5.6.0-5.6.3版本的MySql数据库不支持毫秒级别的精度且无对应的DDL语句，建议升级到5.6.4+；5.6.0以下版本可正常使用。执行文件含有*mysql55*字样以代表5.6.4以下的版本。

### 2.5 数据库表名

所有的Flowable数据库均以**ACT_**开头，第二部分是两个字符的表用例标识，这个用例将大致与服务API匹配。
- **`ACT_RE_*`**：RE代表repository，含有该前缀的表包含了静态信息，比如process definitions和process resources(images, rules等)等。
- **`ACT_RU_*`**：RU代表runtime，表示含有运行时信息的运行时表，比如process实例, user tasks, variables, jobs等。Flowable只在process实例执行时保存运行时数据，并在process实例结束时移除记录，以确保运行时数据库表小且快。
- **`ACT_HI_*`**：HI代表history，表示含有历史数据的表，比如旧的process实例, tasks, variables等。
- **`ACT_GE_*`**：general data用于不同的用例。

通常情况下process engine创建时会进行数据库的版本检查，如果有版本差异则抛出异常。因此可设置为数据库升级，在配置文件中的processEngineConfigutation中设置`<property name="databaseSchemaUpdate" value="true" />`实现。

### 2.6 Job Executor

Flowable V6中只有async executor，因为他的性能和数据库友好性更好。

AsyncExecutor是用于管理一个执行计时器和其他同步任务的线程池的组件，其他的实现参见高级章节。默认情况下此组件未被激活，通过设置`<property name="asyncExecutorActivate" value="true" />`开启。

### 2.7 在表达式和脚本中使用配置bean

默认情况下，在flowable.cfg.xml或Spring配置文件中指定的所有bean都可用于表达式和脚本。如果要限制配置文件中bean的可见性，可以在process engine配置中配置名为beans的属性。ProcessEngineConfiguration中的beans属性是一个map。 当指定该属性时，表达式和脚本只能看到该map中指定的bean。公开的bean将使用在map中指定的名称。

### 2.8 缓存配置及日志

默认会对所有的配置进行缓存以减少对数据库的访问，且缓存没有限制。可通过`<property name="processDefinitionCacheLimit" value="10" />`进行缓存数设置，具体的数目根据存储和使用的process definition的数量决定。通过配置`<property name="processDefinitionCache"><bean class="org.flowable.MyCache" /></property>`可配置实现了`org.flowable.engine.impl.persistence.deploy.DeploymentCache`接口的缓存实现。

Flowable及所有组件使用slf4j进行日志的分发，需要指定与slf4j绑定的日志程序。

### 2.9 事件操控

Flowable的可通过事件机制进行信息提醒，如通过配置文件、运行时API设置engine-wide的事件监听器、通过BPMN XML增加事件监听器。所有的事件都是org.flowable.engine.common.api.delegate.event.FlowableEvent的子类型，该类提供了type, executionId, processInstanceId和processDefinitionId。

#### 2.9.1 实现事件监听器

所有的事件监听器都是`org.flowable.engine.delegate.event.FlowableEventListener`的子类。Flowable提供了一些实现，如`org.flowable.engine.delegate.event.BaseEntityEventListener`。示例如下所示：

```java
public class MyEventListener implements FlowableEventListener {

  @Override
  public void onEvent(FlowableEvent event) {

    if(event.getType() == FlowableEngineEventType.JOB_EXECUTION_SUCCESS) {
      System.out.println("A job well done!");
    } else if (event.getType() == FlowableEngineEventType.JOB_EXECUTION_FAILURE) {
      System.out.println("A job has failed...");
    } else {
      System.out.println("Event received: " + event.getType());
    }
  }

  @Override
  public boolean isFailOnException() {
    // The logic in the onEvent method of this listener is not critical, exceptions
    // can be ignored if logging fails...
    // 定义了onEvent方法在事件派遣时抛出异常的情况下的行为。
    // 返回false则忽略异常；
    // 如果返回true，异常会被向上抛出并使当前的命令失效，如果此事件使用API（如事务），则事务会回滚。
    // 建议如果事件监听器不是严格用于事务，则返回false
    return false;
  }

  @Override
  public boolean isFireOnTransactionLifecycleEvent() {
    // 确定此事件监听器是在事件发生时立即触发，还是由getOnTransaction方法所确定的事务生命周期事件触发。支持的事务生命周期事件的值包括：COMMITTED，ROLLED_BACK，COMMITTING，ROLLINGBACK。
    return false;
  }

  @Override
  public String getOnTransaction() {
    return null;
  }
}
```

#### 2.9.2 监听器配置

在配置文件中配置的监听器，将从process engine开始到关闭工作。eventListeners属性需要一个`org.flowable.engine.delegate.event.FlowableEventListener`实例的列表，示例如下：

```xml
<bean id="processEngineConfiguration"
    class="org.flowable.engine.impl.cfg.StandaloneProcessEngineConfiguration">
    ...
    <property name="eventListeners">
      <list>
         <bean class="org.flowable.engine.example.MyEventListener" />
      </list>
    </property>
</bean>
```

当需要监听某些类型事件发生时，使用`typedEventListeners`属性，该属性为一个map。该map-entry的key是一个逗号分隔的event-names列，map-entry的值为`org.flowable.engine.delegate.event.FlowableEventListener`的实例，示例如下，在job execution成功或失败时被触发。

```xml
<bean id="processEngineConfiguration"
    class="org.flowable.engine.impl.cfg.StandaloneProcessEngineConfiguration">
    ...
    <property name="typedEventListeners">
      <map>
        <entry key="JOB_EXECUTION_SUCCESS,JOB_EXECUTION_FAILURE" >
          <list>
            <bean class="org.flowable.engine.example.MyJobEventListener" />
          </list>
        </entry>
      </map>
    </property>
</bean>
```

事件派发的顺序根据监听器被增加的顺序决定。首先是正常监听器，即eventListeners属性列按照在list中定义的顺序，其次是typedEventListeners。

#### 2.9.3 运行时增加监听器

```java
/**
 * Adds an event-listener which will be notified of ALL events by the dispatcher.
 * @param listenerToAdd the listener to add
 */
void addEventListener(FlowableEventListener listenerToAdd);

/**
 * Adds an event-listener which will only be notified when an event occurs,
 * which type is in the given types.
 * @param listenerToAdd the listener to add
 * @param types types of events the listener should be notified for
 */
void addEventListener(FlowableEventListener listenerToAdd, FlowableEventType... types);

/**
 * Removes the given listener from this dispatcher. The listener will no longer be notified,
 * regardless of the type(s) it was registered for in the first place.
 * @param listenerToRemove listener to remove
 */
 void removeEventListener(FlowableEventListener listenerToRemove);
```

#### 2.9.4 process definition监听器

可以将侦听器添加到特定的流程定义中。只会调用与流程定义相关的事件监听器以及与使用该特定流程定义启动的流程实例相关的所有事件的监听器。可以使用完全限定的类名来定义监听器的实现，该表达式解析为实现侦听器接口的bean，或者可以配置为抛出消息/信号/错误BPMN事件。

**Listeners executing user-defined logic**

监听事件：

```xml
<process id="testEventListeners">
<extensionElements>
  <flowable:eventListener class="org.flowable.engine.test.MyEventListener" />
  <flowable:eventListener delegateExpression="${testEventListener}" events="JOB_EXECUTION_SUCCESS,JOB_EXECUTION_FAILURE" />
</extensionElements>

...

</process>
```

监听实体：

```xml
<process id="testEventListeners">
  <extensionElements>
    <flowable:eventListener class="org.flowable.engine.test.MyEventListener" entityType="task" />
    <flowable:eventListener delegateExpression="${testEventListener}" events="ENTITY_CREATED" entityType="task" />
  </extensionElements>

  ...

</process>
```

**Listeners throwing BPMN events**

处理正在调度的事件的另一种方法是抛出BPMN事件。注意，只有使用某些Flowable事件类型抛出BPMN事件才有意义。例如，删除流程实例时抛出BPMN事件将导致错误。下面的示例显示了如何在process实例中抛出信号，向外部process（全局）抛出信号，在process实例中抛出消息事件并在process实例中抛出错误事件。不使用class或delegateExpression，而是使用属性throwEvent以及特定于要抛出的事件类型的附加属性。

```xml
<process id="testEventListeners">
  <extensionElements>
    <flowable:eventListener throwEvent="signal" signalName="My signal" events="TASK_ASSIGNED" />
  </extensionElements>
</process>
```

```xml
<process id="testEventListeners">
  <extensionElements>
    <flowable:eventListener throwEvent="globalSignal" signalName="My signal" events="TASK_ASSIGNED" />
  </extensionElements>
</process>
```

```xml
<process id="testEventListeners">
  <extensionElements>
    <flowable:eventListener throwEvent="message" messageName="My message" events="TASK_ASSIGNED" />
  </extensionElements>
</process>
```

```xml
<process id="testEventListeners">
  <extensionElements>
    <flowable:eventListener throwEvent="error" errorCode="123" events="TASK_ASSIGNED" />
  </extensionElements>
</process>
```

#### 2.9.5 通过API调度事件

建议只调度CUSTOM类型的FlowableEvents，使用RuntimeService进行调度。

```java
/**
 * Dispatches the given event to any listeners that are registered.
 * @param event event to dispatch.
 *
 * @throws FlowableException if an exception occurs when dispatching the event or
 * when the {@link FlowableEventDispatcher} is disabled.
 * @throws FlowableIllegalArgumentException when the given event is not suitable for dispatching.
 */
 void dispatchEvent(FlowableEvent event);
```

#### 2.9.6 支持的事件类型

每一个类型对应于`org.flowable.engine.common.api.delegate.event.FlowableEventType`下的一个枚举值。
- `ENGINE_CREATED`：该监听器所附加的process engine已被创建且准备好接受API调用；org.flowable…​FlowableEvent。
- `ENGINE_CLOSED`：该监听器所附加的process engine已被关闭且API调用将不可用；org.flowable…​FlowableEvent。
- `ENTITY_CREATED`：一个新entity被创建，这个新entity包含在事件中；org.flowable…​FlowableEntityEvent。
- `ENTITY_INITIALIZED`：一个新entity被创建并被完全初始化，如果该entity有子entity创建则该事件直到所有子entity完成创建后才会发出；org.flowable…​FlowableEntityEvent。
- `ENTITY_UPDATED`：一个已存在的entity被升级，被升级的entity包含在事件中；org.flowable…​FlowableEntityEvent。
- `ENTITY_DELETED`：一个已存在的entity被删除，被删除的entity包含在事件中；org.flowable…​FlowableEntityEvent。
- `ENTITY_SUSPENDED`：一个已存在的entity被暂停，被暂停的entity包含在事件中，将被ProcessDefinitions、ProcessInstances、Tasks分派；org.flowable…​FlowableEntityEvent。
- `ENTITY_ACTIVATED`：一个已存在的entity被激活，被激活的entity包含在事件中，将被ProcessDefinitions、ProcessInstances、Tasks分派；org.flowable…​FlowableEntityEvent。
- `JOB_EXECUTION_SUCCESS`：一项作业已执行成功，该事件包含被执行的作业；org.flowable…​FlowableEntityEvent。
- `JOB_EXECUTION_FAILURE`：一项作业已执行失败，该事件包含被执行的作业及异常；org.flowable…​FlowableEntityEvent和org.flowable…​FlowableExceptionEvent。
- `JOB_RETRIES_DECREMENTED`：由于作业失败，作业重试次数已减少，该事件包含已更新的作业；org.flowable…​FlowableEntityEvent。
- `TIMER_SCHEDULED`：一项记时作业被创建并计划在未来某个时间点执行；org.flowable…​FlowableEntityEvent。
- `TIMER_FIRED`：计时器被触发，该事件包含已被执行的作业；org.flowable…​FlowableEntityEvent。
- `JOB_CANCELED`：作业被取消，该事件包含已被取消的作业，在新的流程定义部署中，可以通过API调用取消作业，完成任务并取消关联的边界计时器；org.flowable…​FlowableEntityEvent。
- `ACTIVITY_STARTED`：一项活动开始执行；org.flowable…​FlowableActivityEvent。
- `ACTIVITY_COMPLETED`：一项活动执行成功；org.flowable…​FlowableActivityEvent。
- `ACTIVITY_CANCELLED`：一项活动将被取消，可能有三个原因（MessageEventSubscriptionEntity, SignalEventSubscriptionEntity, TimerEntity）；org.flowable…​FlowableActivityCancelledEvent。
- `ACTIVITY_SIGNALED`：一项活动收到信号；org.flowable…​FlowableSignalEvent。
- `ACTIVITY_MESSAGE_RECEIVED`：一项活动收到了消息；在活动收到消息之前进行调度，收到后，将根据类型（边界事件或事件子进程启动事件）为此活动调度ACTIVITY_SIGNAL或ACTIVITY_STARTED；org.flowable…​FlowableMessageEvent。
- `ACTIVITY_MESSAGE_WAITING`：一项活动创建了一个消息事件订阅且正在等待接收；org.flowable…​FlowableMessageEvent。
- `ACTIVITY_MESSAGE_CANCELLED`：已经为其创建了消息事件订阅的活动被取消，因此接收该消息将不再触发该特定消息；org.flowable…​FlowableMessageEvent
- `ACTIVITY_ERROR_RECEIVED`：一项活动收到错误事件；在活动处理实际错误之前进行调度；事件的activityId包含对错误处理活动的引用。如果错误成功传递，则此事件将跟随ACTIVITY_SIGNALLED或ACTIVITY_COMPLETE用于所涉及的活动；org.flowable…​FlowableErrorEvent
- `UNCAUGHT_BPMN_ERROR`：抛出未处理的BPMN错误，该process没有针对该特定错误的任何处理程序，此事件的activityId将为空；org.flowable…​FlowableErrorEvent。
- `ACTIVITY_COMPENSATE`：一项活动即将得到补偿，该事件包含将要执行以进行补偿的活动的ID；org.flowable…​FlowableActivityEvent。
- `MULTI_INSTANCE_ACTIVITY_STARTED`：一个多实例活动将被执行；org.flowable…​FlowableMultiInstanceActivityEvent。
- `MULTI_INSTANCE_ACTIVITY_COMPLETED`：一个多实例活动已成功执行完毕；org.flowable…​FlowableMultiInstanceActivityEvent。
- `MULTI_INSTANCE_ACTIVITY_CANCELLED`：一个多实例活动将被取消，可能有三个原因（MessageEventSubscriptionEntity, SignalEventSubscriptionEntity, TimerEntity）；org.flowable…​FlowableMultiInstanceActivityCancelledEvent。
- `VARIABLE_CREATED`：创建变量，该事件包含变量名称，以及相关执行和任务；org.flowable…​FlowableVariableEvent。
- `VARIABLE_UPDATED`：现有变量已更新，该事件包含变量名称，更新值以及相关执行和任务（如果有）；org.flowable…​FlowableVariableEvent。
- `VARIABLE_DELETED`：现有变量已删除，该事件包含变量名称，最后的值以及相关执行和任务（如果有）；org.flowable…​FlowableVariableEvent。
- `TASK_ASSIGNED`：一个任务被分配给用户，该事件包含任务；org.flowable…​FlowableEntityEvent。
- `TASK_CREATED`：一个任务被创建，这是在ENTITY_CREATE事件之后调度的，如果任务是process的一部分，则在执行任务监听器之前将触发此事件；org.flowable…​FlowableEntityEvent。
- `TASK_COMPLETED`：一个任务已完成，这是在ENTITY_DELETE事件之前调度的。如果该任务是process的一部分，则在process继续之前将触发此事件，之后将跟随触发ACTIVITY_COMPLETE事件（该事件指向表示已完成任务的活动）；org.flowable…​FlowableEntityEvent。
- `PROCESS_CREATED`：一个流程实例已经被创建，所有的基础属性被设置，但是还没有变量；org.flowable…​FlowableEntityEvent。
- `PROCESS_STARTED`：一个流程实例已启动，在启动先前创建的流程实例时调度。在关联事件ENTITY_INITIALIZED之后和设置变量之后调度事件PROCESS_STARTED；org.flowable…​FlowableEntityEvent。
- `PROCESS_COMPLETED`：一个流程已完成，表示流程实例已停止所有的执行，在最后一个活动ACTIVITY_COMPLETED事件之后调度，当流程达到流程实例没有任何转换的状态时，流程就完成了；org.flowable…​FlowableEntityEvent。
- `PROCESS_COMPLETED_WITH_TERMINATE_END_EVENT`：一个流程已经结束并达到最后的事件；org.flowable…​FlowableProcessTerminatedEvent。
- `PROCESS_CANCELLED`：一个流程已被取消。在从运行时删除流程实例之前调度。例如，可以通过API调用RuntimeService.deleteProcessInstance，通过调用活动上的中断边界事件等方法来取消流程实例；org.flowable…​FlowableCancelledEvent。
- `MEMBERSHIP_CREATED`：用户已添加到组中，该事件包含所涉及的用户和组的ID；org.flowable…​FlowableMembershipEvent。
- `MEMBERSHIP_DELETED`：用户已从组中删除，该事件包含所涉及的用户和组的ID；org.flowable…​FlowableMembershipEvent。
- `MEMBERSHIPS_DELETED`：所有成员都将从一个组中删除，在删除成员之前抛出该事件，因此仍可访问它们。出于性能原因，如果立即删除所有成员，则不会抛出任何单独的MEMBERSHIP_DELETED事件；org.flowable…​FlowableMembershipEvent。

所有`ENTITY_`开头的事件都与引擎内entity相关，下面的列表显示了为哪些实体分派了哪些实体事件的概述：
- `ENTITY_CREATED, ENTITY_INITIALIZED, ENTITY_DELETED`: Attachment, Comment, Deployment, Execution, Group, IdentityLink, Job, Model, ProcessDefinition, ProcessInstance, Task, User.
- `ENTITY_UPDATED`: Attachment, Deployment, Execution, Group, IdentityLink, Job, Model, ProcessDefinition, ProcessInstance, Task, User.
- `ENTITY_SUSPENDED, ENTITY_ACTIVATED`: ProcessDefinition, ProcessInstance/Execution, Task.