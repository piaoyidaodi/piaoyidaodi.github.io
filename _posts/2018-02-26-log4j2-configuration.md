---
layout: post
title: "Log4j2 -- Configuration"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 4. 配置

将日志请求插入到应用程序代码中需要进行相当多的计划和努力。观察显示大约4%的代码专用于日志记录。因此，即使大小适中的应用程序也会在其代码中嵌入数千条日志语句。根据他们的编号，无需手动修改的管理这些日志语句变得势在必行。

Log4j 2的配置可以通过以下四种方式完成：

1. 通过以XML，JSON，YAML或properties格式编写的配置文件。
2. 以编程方式，通过创建ConfigurationFactory和Configuration实现。
3. 以编程方式，通过调用Configuration接口中的API将组件添加到默认配置。
4. 以编程方式，通过调用内部Logger类的方法。

本页主要关注通过配置文件配置Log4j。有关以编程方式配置Log4j的信息，请参阅Extending Log4j2和Programmatic Log4j Configuration。

请注意与Log4j 1.x不同的是，Log4j2的公共API无用于添加，修改或删除appender和filter或以任何方式操作配置的方法。

### 4.1 自动配置

Log4j具有在初始化期间自动配置的能力。当Log4j启动时，它将找到所有ConfigurationFactory并按照从高到低的加权顺序排列它们。Log4j包含四个ConfigurationFactory实现：一个用于JSON，一个用于YAML，一个用于properties，一个用于XML。

1. Log4j将检查log4j.configurationFile系统属性，如果已设置，则会尝试使用与文件扩展名匹配的ConfigurationFactory加载配置。
2. 如果没有设置系统属性，则properties ConfigurationFactory将在类路径中查找log4j2-test.properties文件。
3. 如果没有找到这样的文件，则YAML ConfigurationFactory将在类路径中查找log4j2-test.yaml或log4j2-test.yml。
4. 如果没有找到这样的文件，则JSON ConfigurationFactory将在类路径中查找log4j2-test.json或log4j2-test.jsn。
5. 如果没有找到这样的文件，则XML ConfigurationFactory将在类路径中查找log4j2-test.xml。
6. 如果无法找到test文件，则properties ConfigurationFactory将在类路径中查找log4j2.properties。
7. 如果找不到properties文件，则YAML ConfigurationFactory将在类路径中查找log4j2.yaml或log4j2.yml。
8. 如果找不到YAML文件，则JSON ConfigurationFactory将在类路径中查找log4j2.json或log4j2.jsn。
9. 如果找不到JSON文件，则XML ConfigurationFactory将尝试在类路径中找到log4j2.xml。
10. 如果找不到配置文件，将使用DefaultConfiguration。这将导致日志记录输出到控制台。

一个使用log4j的名为MyApp的示例应用程序可以用来说明如何完成。

```java
import com.foo.Bar;

// Import log4j classes.
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;

public class MyApp {

    // Define a static logger variable so that it references the
    // Logger instance named "MyApp".
    private static final Logger logger = LogManager.getLogger(MyApp.class);

    public static void main(final String... args) {

        // Set up a simple configuration that logs on the console.

        logger.trace("Entering application.");
        Bar bar = new Bar();
        if (!bar.doIt()) {
            logger.error("Didn't do it.");
        }
        logger.trace("Exiting application.");
    }
}
```

MyApp首先导入log4j相关的类，然后定义了一个名为MyApp的静态logger变量，该logger名恰好是这个类的完全限定名。

MyApp使用com.foo包中定义的Bar类。

```java
package com.foo;
import org.apache.logging.log4j.Logger;
import org.apache.logging.log4j.LogManager;

public class Bar {
  static final Logger logger = LogManager.getLogger(Bar.class.getName());

  public boolean doIt() {
    logger.entry();
    logger.error("Did it again!");
    return logger.exit(false);
  }
}
```

如果找不到配置文件，Log4j将提供默认配置。DefaultConfiguration类中提供的默认配置将设置为：

- 连接到root logger的ConsoleAppender。
- 将PatternLayout设置为连接到ConsoleAppender，并设置模式为`%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n`。

请注意，默认情况下，Log4j将root logger设置为Level.ERROR。

MyApp的输出将类似于：

```
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] ERROR MyApp - Didn't do it.
```

如前所述，Log4j首先尝试通过配置文件进行配置，一个类似于默认配置的配置文件如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

一旦这个文件被放置在类路径并命名为log4j2.xml，你将得到与上面列出的结果相同的结果。将root级别更改为trace将产生类似于以下结果：

```
17:13:01.540 [main] TRACE MyApp - Entering application.
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] TRACE com.foo.Bar - exit with (false)
17:13:01.540 [main] ERROR MyApp - Didn't do it.
17:13:01.540 [main] TRACE MyApp - Exiting application.
```

请注意，使用默认配置时，status记录被禁用。

### 4.2 可加性

也许希望清除除com.foo.Bar之外的所有TRACE输出。简单的更改日志级别**不能**完成任务。相反，需要将新的Logger定义添加到配置中：

```xml
<Logger name="com.foo.Bar" level="TRACE"/>
<Root level="ERROR">
  <AppenderRef ref="STDOUT">
</Root>
```

使用此配置，将记录所有来自com.foo.Bar的所有日志事件，而所有其他组件将仅记录error事件。

在前面的例子中，com.foo.Bar中的所有事件仍然写入控制台。这是因为尽管com.foo.Bar的Logger没有任何appender配置，而它的父类拥有。其实就是下面的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

将产生如下输出：

```
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] TRACE com.foo.Bar - entry
17:13:01.540 [main] ERROR com.foo.Bar - Did it again!
17:13:01.540 [main] TRACE com.foo.Bar - exit (false)
17:13:01.540 [main] TRACE com.foo.Bar - exit (false)
17:13:01.540 [main] ERROR MyApp - Didn't do it.
```

请注意，来自com.foo.Bar的trace消息出现了两次。这是因为首先使用了与logger com.foo.Bar相关联的appender，它将第一个消息实例写入控制台。接下来，引用com.foo.Bar的父类，即本例中的root记录器，然后日志事件被传递给root的appender并写入控制台，产生了第二个消息实例。这被称为可加性。虽然additivity可以是一个非常方便的功能（如前面的示例，无需配置appender引用），但在许多情况下此行为被认为是不可取的。因此可以通过将Logger上的additivity属性设置为false以禁用它：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="com.foo.Bar" level="trace" additivity="false">
      <AppenderRef ref="Console"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="Console"/>
    </Root>
  </Loggers>
</Configuration>
```

一旦日志事件到达一个将其可加性设置为false的Logger，则不管该Logger的父记录器可加性设置为何，该事件都不会传递给它的任何父记录器。

### 4.3 自动再配置

当使用文件配置时，Log4j可以自动检测配置文件的改变并进行自动再配置。如果在配置元素上设置了非零的monitorInterval属性，则将在日志事件下一次被计算和/或被记录以及自上次检查以来已经过去monitorInterval时重新检查配置文件。下面的示例显示了如何配置该属性，以便仅在至少经过30秒后才会检查配置文件的更改。monitorInterval的最小间隔是5秒。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration monitorInterval="30">
...
</Configuration>
```

### 4.4 Chainsaw会自动处理log文件

### 4.5 配置语法

从版本2.9开始，出于安全原因，Log4j不会在XML文件中处理DTD。如果要将配置拆分为多个文件，请使用XInclude或Composite Configuration。

正如前面的示例以及后续示例所示，Log4j允许您轻松重新定义日志记录行为而无需修改应用程序。可以禁用应用程序某些部分的日志记录，只有在满足特定条件时才记录日志，例如正在为特定用户执行的操作，将输出路由到Flume或日志报告系统等。要做到这一点，需要理解配置文件的语法。

XML文件中的配置元素接受几个属性：

- `advertiser`：（可选）将用于配置通告单个FileAppender或SocketAppender的Advertiser插件名称。唯一提供的Advertiser插件是`multicastdns`。
- `dest`：也被成为`err`，将输出发送到stderr或一个文件路径或URL。
- `monitorInterval`：在检查文件配置更改之前必须经过的最短时间（以秒为单位）。
- `name`：配置的名称。
- `packages`：使用逗号分隔的包名来搜索插件。每个类加载器只加载一次插件，所以更改此值可能对重新配置没有任何影响。
- `schema`：标识类加载器的位置以找到用于验证配置的XML架构。只有当严格设置为true时才有效。如果没有设置，则不会发生架构验证。
- `shutdownHook`：指定在JVM关闭时是否应该自动关闭Log4j。shutdown hook默认启用，但可通过将此属性设置为false来禁用。
- `shutdownTimeout`：指定JVM关闭时有多少毫秒时间关闭appender和后台任务。默认值为零表示每个appender使用其默认超时值，并且不等待后台任务。但这只是一个hint，并非所有的appender都会遵守这一点，且并不能绝对保证快速执行关闭程序。将该值设置的太小将会增加日志事件尚未完全写入最终目的地而丢失的风险。详情请参阅`LoggerContext.stop(long, java.util.concurrent.TimeUnit)`。（通过将shutdownHook设置为false而禁用该属性。）
- `status`：应该打印记录到控制台的内部Log4j事件的级别。此属性可被设置为`trace，debug，info，warn，error，fatal`。Log4j会记录有关初始化，rollover和其他内部操作的详细信息到status记录器。如果您需要对log4j排故，可设置`status="trace"`。
（或者，设置系统属性log4j2.debug也会打印内部Log4j2日志记录到控制台，包括在发现配置文件之前发生的内部日志记录。）
- `strict`：允许使用严格的XML格式。不支持JSON配置中。
- `verbose`：在加载插件时启用诊断信息。

**XML配置语法**

Log4j可以使用两种XML风格进行配置：简洁和严谨。简洁的格式使得配置变得非常简单，因为元素名称与它们所代表的组件匹配，但无法使用XML模式进行验证。例如，ConsoleAppender通过在其父appender元素下声明一个名为Console的XML元素进行配置。但是，元素和属性名称不区分大小写。另外，可以将属性指定为XML属性，也可以将其指定为不具有属性且具有文本值的XML元素。因此以下代码的效果是等同的。

```xml
<PatternLayout pattern="%m%n"/>

<PatternLayout>
  <Pattern>%m%n</Pattern>
</PatternLayout>
```

下面的示例是一个XML配置的结构，注意其中的*斜体*元素代表可能出现的简洁元素名称。
```xml
<?xml version="1.0" encoding="UTF-8"?>;
<Configuration>
  <Properties>
    <Property name="name1">value</property>
    <Property name="name2" value="value2"/>
  </Properties>
  <!-- 斜体 -->
  <filter  ... />
  <Appenders>
    <!-- 斜体 -->
    <appender ... >
      <filter  ... />
    </appender>
    ...
  </Appenders>
  <Loggers>
    <Logger name="name1">
      <!-- 斜体 -->
      <filter  ... />
    </Logger>
    ...
    <Root level="level">
      <AppenderRef ref="name"/>
    </Root>
  </Loggers>
</Configuration>
```

**Strict XML**

除了上述简洁版XML格式之外，Log4j允许常规的XML方式进行配置，并可以使用XML模式进行验证。这是通过用它们的对象类型替换上面的简洁元素名称来完成的，具体如下所示。例如，不使用名为Console的元素来配置ConsoleAppender，而是将其配置为包含类型属性Console的appender元素。

```xml
<?xml version="1.0" encoding="UTF-8"?>;
<Configuration>
  <Properties>
    <Property name="name1">value</property>
    <Property name="name2" value="value2"/>
  </Properties>
  <Filter type="type" ... />
  <Appenders>
    <Appender type="type" name="name">
      <Filter type="type" ... />
    </Appender>
    ...
  </Appenders>
  <Loggers>
    <Logger name="name1">
      <Filter type="type" ... />
    </Logger>
    ...
    <Root level="level">
      <AppenderRef ref="name"/>
    </Root>
  </Loggers>
</Configuration>
```

以下示例为使用strict模式的xml配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" strict="true" name="XMLConfigTest"
               packages="org.apache.logging.log4j.test">
  <Properties>
    <Property name="filename">target/test.log</Property>
  </Properties>
  <Filter type="ThresholdFilter" level="trace"/>
 
  <Appenders>
    <Appender type="Console" name="STDOUT">
      <Layout type="PatternLayout" pattern="%m MDC%X%n"/>
      <Filters>
        <Filter type="MarkerFilter" marker="FLOW" onMatch="DENY" onMismatch="NEUTRAL"/>
        <Filter type="MarkerFilter" marker="EXCEPTION" onMatch="DENY" onMismatch="ACCEPT"/>
      </Filters>
    </Appender>
    <Appender type="Console" name="FLOW">
      <Layout type="PatternLayout" pattern="%C{1}.%M %m %ex%n"/><!-- class and line number -->
      <Filters>
        <Filter type="MarkerFilter" marker="FLOW" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
        <Filter type="MarkerFilter" marker="EXCEPTION" onMatch="ACCEPT" onMismatch="DENY"/>
      </Filters>
    </Appender>
    <Appender type="File" name="File" fileName="${filename}">
      <Layout type="PatternLayout">
        <Pattern>%d %p %C{1.} [%t] %m%n</Pattern>
      </Layout>
    </Appender>
  </Appenders>
 
  <Loggers>
    <Logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
      <Filter type="ThreadContextMapFilter">
        <KeyValuePair key="test" value="123"/>
      </Filter>
      <AppenderRef ref="STDOUT"/>
    </Logger>
 
    <Logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
      <AppenderRef ref="File"/>
    </Logger>
 
    <Root level="trace">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
 
</Configuration>
```

**JSON配置语法**

除了XML之外，可以使用JSON配置Log4j。JSON格式与简洁版XML格式非常相似。每个键表示插件的名称，与其关联的键/值对是其属性。 如果一个关键字包含的不仅仅是一个简单的值，它本身就是一个从属插件。在下面的示例中ThresholdFilter，Console和PatternLayout都是插件，而Console插件将为其name属性分配一个STDOUT值，并且将为ThresholdFilter分配一个debug级别。

```json
{ "configuration": { "status": "error", "name": "RoutingTest",
                     "packages": "org.apache.logging.log4j.test",
      "properties": {
        "property": { "name": "filename",
                      "value" : "target/rolling1/rollingtest-$${sd:type}.log" }
      },
    "ThresholdFilter": { "level": "debug" },
    "appenders": {
      "Console": { "name": "STDOUT",
        "PatternLayout": { "pattern": "%m%n" },
        "ThresholdFilter": { "level": "debug" }
      },
      "Routing": { "name": "Routing",
        "Routes": { "pattern": "$${sd:type}",
          "Route": [
            {
              "RollingFile": {
                "name": "Rolling-${sd:type}", "fileName": "${filename}",
                "filePattern": "target/rolling1/test1-${sd:type}.%i.log.gz",
                "PatternLayout": {"pattern": "%d %p %c{1.} [%t] %m%n"},
                "SizeBasedTriggeringPolicy": { "size": "500" }
              }
            },
            { "AppenderRef": "STDOUT", "key": "Audit"}
          ]
        }
      }
    },
    "loggers": {
      "logger": { "name": "EventLogger", "level": "info", "additivity": "false",
                  "AppenderRef": { "ref": "Routing" }},
      "root": { "level": "error", "AppenderRef": { "ref": "STDOUT" }}
    }
  }
}
```

请注意，在RoutingAppender中，Route元素已被声明为一个数组，这是有效的。因为每个数组元素都将是一个Route组件，这对于诸如appender和filter等元素不起作用，其中每个元素在简明格式中具有不同的名称。如果每个appender或filter声明了一个包含appender类型的名为type的属性，则可以将appender和filter定义为数组元素。以下示例表明了这一点以及如何将多个记录器声明为数组。

```json
{ "configuration": { "status": "debug", "name": "RoutingTest",
                      "packages": "org.apache.logging.log4j.test",
      "properties": {
        "property": { "name": "filename",
                      "value" : "target/rolling1/rollingtest-$${sd:type}.log" }
      },
    "ThresholdFilter": { "level": "debug" },
    "appenders": {
      "appender": [
         { "type": "Console", "name": "STDOUT", "PatternLayout": { "pattern": "%m%n" }, "ThresholdFilter": { "level": "debug" }},
         { "type": "Routing",  "name": "Routing",
          "Routes": { "pattern": "$${sd:type}",
            "Route": [
              {
                "RollingFile": {
                  "name": "Rolling-${sd:type}", "fileName": "${filename}",
                  "filePattern": "target/rolling1/test1-${sd:type}.%i.log.gz",
                  "PatternLayout": {"pattern": "%d %p %c{1.} [%t] %m%n"},
                  "SizeBasedTriggeringPolicy": { "size": "500" }
                }
              },
              { "AppenderRef": "STDOUT", "key": "Audit"}
            ]
          }
        }
      ]
    },
    "loggers": {
      "logger": [
        { "name": "EventLogger", "level": "info", "additivity": "false",
          "AppenderRef": { "ref": "Routing" }},
        { "name": "com.foo.bar", "level": "error", "additivity": "false",
          "AppenderRef": { "ref": "STDOUT" }}
      ],
      "root": { "level": "error", "AppenderRef": { "ref": "STDOUT" }}
    }
  }
}
```

**YAML配置语法**

### 4.6 配置Logger, Appender, Filter, Properties

**配置Logger**

在尝试配置之前，了解Log4j中记录器的工作方式至关重要。如果需要更多信息，请参考Log4j体系结构。在不理解这些概念的情况下尝试配置Log4j会出现问题。

LoggerConfig是使用logger element配置的。logger element必须具有指定的name属性和level属性，并且还可能指定additivty属性。级别可以配置为TRACE，DEBUG，INFO，WARN，ERROR，ALL或OFF。如果没有指定级别，它将默认为ERROR。可以为addability属性赋值true或false，缺省值true。

LoggerConfig（包括root LoggerConfig）可以配置属性，这些属性将添加到从ThreadContextMap中复制的属性中。这些属性可以从Appender，Filter，Layout等引用，就像它们是ThreadContext Map的一部分一样。这些属性可以包含在解析配置时解析的变量，或者在每个日志事件记录时的动态解析变量。

LoggerConfig也可以配置一个或多个AppenderRef元素。每个引用的appender都将与指定的LoggerConfig关联。如果在LoggerConfig上配置了多个appender，则每个appender在处理日志记录事件时都会被调用。

每个配置都必须有root记录器。如果没有配置则使用默认的root LoggerConfig，其级别为ERROR并且连接了控制台appender。根记录器和其他记录器之间的主要区别是：

- root记录器没有name属性。
- root记录器不支持additivity属性，因为它没有父对象。

**配置Appender**

appender的配置使用特定的appender插件名称或appender元素以及包含appender插件名称的type属性。此外，每个appender必须具有一个name属性值作为appender的唯一标示。Logger将使用该name来引用appender，如前一节所述。

大多数appender还支持要配置layout（也可以使用特定的layout插件名或使用"layout"作为元素名称以及使用包含layout插件名称的type属性来指定layout）。各种appender将包含其正确运行所需的其他属性或元素。

**配置Filter**

Log4j允许在4个地方中指定filter：

1. 与appender，logger和property元素处于同一级别。这些filter可以在事件传递给LoggerConfig之前接受或拒绝事件。
2. 在logger元素中。这些filter可以接受或拒绝特定记录器的事件。
3. 在appender元素中。这些filter可以防止或致使appender处理事件。
4. 在appender的引用元素中。这些filter用于确定该logger的事件是否应路由到appender。

虽然只能配置一个filter元素，但该元素可能是表示CompositeFilter的filters元素。filters元素允许在其中配置任意数量的filter元素。以下示例显示了如何在ConsoleAppender上配置多个filter。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="XMLConfigTest" packages="org.apache.logging.log4j.test">
  <Properties>
    <Property name="filename">target/test.log</Property>
  </Properties>
  <ThresholdFilter level="trace"/>

  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m MDC%X%n"/>
    </Console>
    <Console name="FLOW">
      <!-- this pattern outputs class name and line number -->
      <PatternLayout pattern="%C{1}.%M %m %ex%n"/>
      <filters>
        <MarkerFilter marker="FLOW" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
        <MarkerFilter marker="EXCEPTION" onMatch="ACCEPT" onMismatch="DENY"/>
      </filters>
    </Console>
    <File name="File" fileName="${filename}">
      <PatternLayout>
        <pattern>%d %p %C{1.} [%t] %m%n</pattern>
      </PatternLayout>
    </File>
  </Appenders>

  <Loggers>
    <Logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
      <ThreadContextMapFilter>
        <KeyValuePair key="test" value="123"/>
      </ThreadContextMapFilter>
      <AppenderRef ref="STDOUT"/>
    </Logger>

    <Logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
      <Property name="user">${sys:user.name}</Property>
      <AppenderRef ref="File">
        <ThreadContextMapFilter>
          <KeyValuePair key="test" value="123"/>
        </ThreadContextMapFilter>
      </AppenderRef>
      <AppenderRef ref="STDOUT" level="error"/>
    </Logger>

    <Root level="trace">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

**配置Properties**

从版本2.4开始，Log4j现在支持通过properties文件进行配置。请注意，properties语法与Log4j1中使用的语法不同。与XML和JSON配置类似，properties根据插件和插件属性来定义配置。

在版本2.6之前，properties配置要求在properties文件中以逗号为分隔符列出appender，filter和logger的标识符名称。然后，每个组件都将被定义在以`<identifier>`开始的properties中。标识符不必与要定义组件的名称相匹配，但必须能唯一标识组件中所有的属性和子组件。如果标识符列表不存在，标识符不能包含`.`。每个单独的组件必须有一个指定的type属性来标识组件的插件类型。

从版本2.6开始，这个标识符列表不再需要，因为名字是在首次使用时推断出来的，但是如果您希望使用更复杂的标识符，您仍然必须使用列表。如果列表存在，它将被使用。

与基本组件不同，创建子组件时，不能指定包含标识符列表的元素。相反，您必须使用其type定义的包装元素，如下面的RollingFileAppender中的策略定义中所示。在该包装元素下面的定义每个子组件，如TimeBasedTriggeringPolicy和SizeBasedTriggeringPolicy的定义。

properties配置文件支持advertiser，monitorInterval，name，packages，shutdownHook，shutdownTimeout，status，verbose和dest属性。

```properties
status = error
dest = err
name = PropertiesConfig

property.filename = target/rolling/rollingtest.log

filter.threshold.type = ThresholdFilter
filter.threshold.level = debug

appender.console.type = Console
appender.console.name = STDOUT
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %m%n
appender.console.filter.threshold.type = ThresholdFilter
appender.console.filter.threshold.level = error

appender.rolling.type = RollingFile
appender.rolling.name = RollingFile
appender.rolling.fileName = ${filename}
appender.rolling.filePattern = target/rolling2/test1-%d{MM-dd-yy-HH-mm-ss}-%i.log.gz
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = %d %p %C{1.} [%t] %m%n
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy
appender.rolling.policies.time.interval = 2
appender.rolling.policies.time.modulate = true
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy
appender.rolling.policies.size.size=100MB
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 5

logger.rolling.name = com.example.my.app
logger.rolling.level = debug
logger.rolling.additivity = false
logger.rolling.appenderRef.rolling.ref = RollingFile

rootLogger.level = info
rootLogger.appenderRef.stdout.ref = STDOUT
```

### 4.7属性替换

Log4j2支持在配置中指定token，这些token可以作为引用其他地方定义的属性。其中一些属性将在解释配置文件时解析，而其他属性可能会传递到将在运行时计算它们的组件。为此，Log4j使用Apache Commons Lang的StrSubstitutor和StrLookup类的变体。以类似于Ant或Maven的方式，这允许声明为${name}的变量在配置本身中被解析使用。例如，以下示例显示了RollingFileAppender的filename即被声明为一个变量。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="RoutingTest" packages="org.apache.logging.log4j.test">
  <Properties>
    <Property name="filename">target/rolling1/rollingtest-$${sd:type}.log</Property>
  </Properties>
  <ThresholdFilter level="debug"/>

  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m%n"/>
      <ThresholdFilter level="debug"/>
    </Console>
    <Routing name="Routing">
      <Routes pattern="$${sd:type}">
        <Route>
          <RollingFile name="Rolling-${sd:type}" fileName="${filename}"
                       filePattern="target/rolling1/test1-${sd:type}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="STDOUT" key="Audit"/>
      </Routes>
    </Routing>
  </Appenders>

  <Loggers>
    <Logger name="EventLogger" level="info" additivity="false">
      <AppenderRef ref="Routing"/>
    </Logger>

    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

虽然这很有用，但properties可能来自更多地方。为了适应这种情况，Log4j还支持语法`${prefix:name}`，前缀标识告诉Log4j变量名应该在特定的上下文中进行处理。Logj4内置的上下文是：

- bundle：Resource bundle。格式为${bundle:BundleName:BundleKey}，bundle命名遵循package命名规则，如${budnle:com.domain.Messages:MyKey}。
- ctx：Thread Context Map(MDC)。
- date：使用特定格式的插入日期和/或时间。
- env：系统环境变量。
- jndi：在JNDI上下文中设置的一个值。
- jvmrunargs：通过JMX获得的JVM输入参数。
- log4j：Log4j配置属性，${log4j:configLocation}和${log4j:configParentLocation}分别提供了指向log4j配置文件和包含其文件夹的绝对路径。
- main：使用`MapLookup.setMainArguments(String[])`设置的值。
- map：通过MapMessage设置的值。
- sd：通过StructuredDataMessage设置的值。键id获取无企业编号的StructuredDataId；键type将返回消息类型；其他键将从Map中检索单独的元素。
- sys：System properties。

默认的属性映射可以在配置文件中声明。如果不能在指定位置中查找到目标值，则将使用默认属性映射中的值。默认映射将预设hostName的值为当前系统的主机名或IP地址，预设contextName是当前日志记录上下文的值。

### 4.8 使用多个$前缀的Lookup变量

StrLookup处理的一个有趣特征是：每次变量被解析时，拥有多个'$'前缀的变量引用，前缀'$'将简单地删除。在前面的例子中，Routes元素能够在运行时解析变量。为了允许此前缀值被指定为具有两个前导'$'字符的变量，首次处理配置文件时，第一个$将被简单地删除。因此，当在运行时计算Routes元素时变量为`${sd:type}`，致使从StructuredDataMessage得到变量，并使用其中的type属性的值作为路由键。并非所有元素都支持在运行时解析变量，支持的组件会在他们的文档中特别提到。

如果在与前缀关联的Lookup中找不到键的值，则将使用在配置文件中声明的properties中的键相关联的值。如果未找到任何值，则变量声明将作为值返回。通过执行下列操作可以在配置中声明默认值：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Properties>
    <Property name="type">Audit</property>
  </Properties>
  ...
</Configuration>
```

最后，值得指出的是在处理配置时RollingFileAppender声明中的变量也不会被计算。这只是因为整个RollingFile元素的处理被推迟到匹配发生时。

### 4.9 Scripts

### 4.10 Xinclude

XML配置文件可以通过XInclude包含其他文件，下面是一个log4j2.xml包含其他两个文件的示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration xmlns:xi="http://www.w3.org/2001/XInclude"
               status="warn" name="XIncludeDemo">
  <properties>
    <property name="filename">xinclude-demo.log</property>
  </properties>
  <ThresholdFilter level="debug"/>
  <xi:include href="log4j-xinclude-appenders.xml" />
  <xi:include href="log4j-xinclude-loggers.xml" />
</configuration>
```

log4j-xinclude-appenders.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<appenders>
  <Console name="STDOUT">
    <PatternLayout pattern="%m%n" />
  </Console>
  <File name="File" fileName="${filename}" bufferedIO="true" immediateFlush="true">
    <PatternLayout>
      <pattern>%d %p %C{1.} [%t] %m%n</pattern>
    </PatternLayout>
  </File>
</appenders>
```

log4j-xinclude-loggers.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<loggers>
  <logger name="org.apache.logging.log4j.test1" level="debug" additivity="false">
    <ThreadContextMapFilter>
      <KeyValuePair key="test" value="123" />
    </ThreadContextMapFilter>
    <AppenderRef ref="STDOUT" />
  </logger>
 
  <logger name="org.apache.logging.log4j.test2" level="debug" additivity="false">
    <AppenderRef ref="File" />
  </logger>
 
  <root level="error">
    <AppenderRef ref="STDOUT" />
  </root>
</loggers>
```

### 4.11 组合配置

Log4j允许使用多个配置文件，方法是将它们的文件路径用逗号分隔开并指定在log4j.configurationFile。可以通过在log4j.mergeStrategy属性上指定一个实现MergeStrategy接口的类来控制合并逻辑。默认合并策略将使用以下规则合并文件：

1. 全局配置属性将与以后配置中的配置属性进行聚合，先前的配置属性将被替换，但最高的status级别和大于0的最低monitorInterval将被使用。
2. 所有配置的properties将被聚合。重复的属性取代以前的配置。
3. 如果定义了多个filter，filter将在CompositeFilter中聚合，这是由于filter不能命名重复。
4. Script和ScriptFile引用被聚合。重复的定义取代了以前配置中的定义。
5. Appender被聚合。具有相同名称的Appender被后面的Appender配置取代，包括所有Appender的子组件。
6. Logger被聚合。Logger属性将被单独合并，重复项将被后面配置中的重复项替换。Logger上的Appender重复引用会被聚合，并被后面配置中的重复项替换。如果定义了多个filter，Logger上的filter将聚合在CompositeFilter中，由于filter不能重复命名。 Appender引用下的filter被包含或弃用，取决于它们的父Appender引用是保留还是弃用。

### 4.12 Status消息

正如希望能够诊断应用程序中的问题一样，经常需要能够诊断日志记录配置或配置组件中的问题。由于日志尚未配置，因此在初始化期间无法使用normal日志记录。此外，appender中的正常日志记录可能会导致无限递归，Log4j将检测并将递归事件被忽略。为了适应这种需求，Log4j2 API包含一个StatusLogger。组件声明一个StatusLogger的实例，类似于：

```java
protected final static Logger logger = StatusLogger.getLogger();
```

由于StatusLogger实现Log4j2 API的Logger接口，因此可以使用所有常规的Logger方法。

配置Log4j时，有时需要查看生成的status事件。这可以通过将status属性添加到配置元素来完成，或者可以通过设置“Log4jDefaultStatusLevel”系统属性来提供默认值。 状态属性的有效值是trace，debug，info，warn，error和fatal。 以下配置将状态属性设置为debug。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="RoutingTest">
  <Properties>
    <Property name="filename">target/rolling1/rollingtest-$${sd:type}.log</Property>
  </Properties>
  <ThresholdFilter level="debug"/>
 
  <Appenders>
    <Console name="STDOUT">
      <PatternLayout pattern="%m%n"/>
      <ThresholdFilter level="debug"/>
    </Console>
    <Routing name="Routing">
      <Routes pattern="$${sd:type}">
        <Route>
          <RollingFile name="Rolling-${sd:type}" fileName="${filename}"
                       filePattern="target/rolling1/test1-${sd:type}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="STDOUT" key="Audit"/>
      </Routes>
    </Routing>
  </Appenders>
 
  <Loggers>
    <Logger name="EventLogger" level="info" additivity="false">
      <AppenderRef ref="Routing"/>
    </Logger>
 
    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
 
</Configuration>
```

### 4.13 系统properties

详见官方文档[System Properties](http://logging.apache.org/log4j/2.x/manual/configuration.html#SystemProperties)。
