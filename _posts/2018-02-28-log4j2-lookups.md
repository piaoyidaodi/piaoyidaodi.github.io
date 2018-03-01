---
layout: post
title: "Log4j2 -- Lookups"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 6. Lookups

Lookups提供了一种在任意位置为Log4j配置添加值的方法。它们是实现StrLookup接口的特定类型的Plugin。有关如何在配置文件中使用Lookups的信息可以在Property Substitution部分找到。

### 6.1 Context Map Lookup

ContextMapLookup允许应用程序将数据存储在Log4j ThreadContext Map中，然后在Log4j配置中检索该值。在下面的例子中，应用程序会使用键loginId将当前用户的登录ID存储在ThreadContext Map中。在初始化配置处理期间，第一个$将被删除。PatternLayout支持通过Lookup插值，然后解析每个事件的变量。请注意，模式%X{loginId}将获得相同的结果。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${ctx:loginId} %m%n</pattern>
  </PatternLayout>
</File>
```

### 6.2 Data Lookup

DateLookup与其他Lookup有些不同寻常，因为它不使用键来定位项目。相反，该键可用于指定一个对SimpleDateFormat有效的日期格式字符串。当前日期或与当前日志事件关联的日期将按指定格式化。

```xml
<RollingFile name="Rolling-${map:type}" fileName="${filename}" filePattern="target/rolling1/test1-$${date:MM-dd-yyyy}.%i.log.gz">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] %m%n</pattern>
  </PatternLayout>
  <SizeBasedTriggeringPolicy size="500" />
</RollingFile>
```

### 6.3 Environment Lookup

EnvironmentLookup允许系统在全局文件（如/etc/profile）或应用程序的启动脚本中配置环境变量，然后从日志配置中检索这些变量。下面的示例包含应用程序日志中当前登录的用户的名称。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${env:USER} %m%n</pattern>
  </PatternLayout>
</File>
```

### 6.4 Java Lookup

JavaLookup允许使用以`java:`为前缀方式的预格式化字符串，方便的检索Java环境信息。

- version：Java版本简要信息，如：Java version 1.7.0_67。
- runtime：Java运行时版本，如：Java(TM) SE Runtime Environment (build 1.7.0_67-b01) from Oracle Corporation。
- vm：Java VM版本，如：Java HotSpot(TM) 64-Bit Server VM (build 24.65-b04, mixed mode)。
- os：OS版本，如：Windows 7 6.1 Service Pack 1, architecture: amd64-64。
- locale：硬件信息，如：default locale: en_US, platform encoding: Cp1252。
- hw：硬件信息，如：processors: 4, architecture: amd64-64, instruction sets: amd64。

例如：

```xml
<File name="Application" fileName="application.log">
  <PatternLayout header="${java:runtime} - ${java:vm} - ${java:os}">
    <Pattern>%d %m%n</Pattern>
  </PatternLayout>
</File>
```

### 6.5 JNDI Lookup

JndiLookup允许通过JNDI检索变量，默认情况下，键将以`java:comp/env/`为前缀，但如果键中包含`:`则不会添加前缀。在Android中不能使用Java的JNDI模块。

```xml
<File name="Application" fileName="application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] $${jndi:logging/context-name} %m%n</pattern>
  </PatternLayout>
</File>
```

### 6.6 Log4j Configuration Location Lookup

Log4j配置属性。表达式`${log4j:configLocation}`和`${log4j:configParentLocation}`分别为log4j配置文件及其父文件夹提供绝对路径。

以下示例，使用此Lookup将日志文件放置在一个相对于log4j配置文件的目录中。

```xml
<File name="Application" fileName="${log4j:configParentLocation}/logs/application.log">
  <PatternLayout>
    <pattern>%d %p %c{1.} [%t] %m%n</pattern>
  </PatternLayout>
</File>
```

### 6.7 Main Arguments Lookup(Application)

此Lookup要求您手动向Log4j提供应用程序的主要参数：

```java
import org.apache.logging.log4j.core.lookup.MainMapLookup;

public static void main(String args[]) {
  MainMapLookup.setMainArguments(args);
  ...
}
```

如果主参数已设置，则此Lookup允许应用程序从日志配置中检索这些主参数值。以`main:`为前缀的键的后面部分可以是从参数列表中的以0开始的索引，也可以是一个字符串，其中`${main:myString}`的myString由main参数列表中的值替换。

例如，假设static void main String[]是：`--file foo.txt --verbose -x bar`，则

- ${main:0} : --file
- ${main:1} : foo.txt
- ${main:2}	: --verbose
- ${main:3}	: -x
- ${main:4}	: bar
- ${main:--file} : foo.txt
- ${main:-x} : bar
- ${main:bar} : null

使用示例：

```xml
<File name="Application" fileName="application.log">
  <PatternLayout header="File: ${main:--file}">
    <Pattern>%d %m%n</Pattern>
  </PatternLayout>
</File>
```

### 6.8 Map Lookup

MapLookup有几个用途：

1. 为配置文件中声明的Properties提供基础。
2. 从在LogEvents中的MapMessages中检索值。
3. 获取使用`MapLookup.setMainArguments(String[])`设置的值。

第一项意味着MapLookup用于替换配置文件中定义的properties。这些变量没有指定的前缀，例如`${name}`。第二种用法允许当前MapMessage中的值被替换（如果是当前日志事件的一部分）。在下面的示例中，RoutingAppender将为MapMessage中名为type的键的每个唯一值使用不同的RollingFileAppender。请注意，以这种方式使用时，应在properties声明中声明属性type的值，以便在消息不是MapMessage或MapMessage不包含键的情况下提供默认值。有关如何设置默认值，请参阅Property Substitution部分。

```xml
<Routing name="Routing">
  <Routes pattern="$${map:type}">
    <Route>
      <RollingFile name="Rolling-${map:type}" fileName="${filename}"
                   filePattern="target/rolling1/test1-${map:type}.%i.log.gz">
        <PatternLayout>
          <pattern>%d %p %c{1.} [%t] %m%n</pattern>
        </PatternLayout>
        <SizeBasedTriggeringPolicy size="500" />
      </RollingFile>
    </Route>
  </Routes>
</Routing>
```

### 6.9 Marker Lookup

Marker Lookup允许你像routing appender一样在感兴趣的配置中使用标记。如下的YAML配置和基于Marker记录到不同文件的代码：

```yaml
Configuration:
  status: debug

  Appenders:
    Console:
    RandomAccessFile:
      - name: SQL_APPENDER
        fileName: logs/sql.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"
      - name: PAYLOAD_APPENDER
        fileName: logs/payload.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"
      - name: PERFORMANCE_APPENDER
        fileName: logs/performance.log
        PatternLayout:
          Pattern: "%d{ISO8601_BASIC} %-5level %logger{1} %X %msg%n"

    Routing:
      name: ROUTING_APPENDER
      Routes:
        pattern: "$${marker:}"
        Route:
        - key: PERFORMANCE
          ref: PERFORMANCE_APPENDER
        - key: PAYLOAD
          ref: PAYLOAD_APPENDER
        - key: SQL
          ref: SQL_APPENDER

  Loggers:
    Root:
      level: trace
      AppenderRef:
        - ref: ROUTING_APPENDER
```

```java
public static final Marker SQL = MarkerFactory.getMarker("SQL");
public static final Marker PAYLOAD = MarkerFactory.getMarker("PAYLOAD");
public static final Marker PERFORMANCE = MarkerFactory.getMarker("PERFORMANCE");

final Logger logger = LoggerFactory.getLogger(Logger.ROOT_LOGGER_NAME);

logger.info(SQL, "Message in Sql.log");
logger.info(PAYLOAD, "Message in Payload.log");
logger.info(PERFORMANCE, "Message in Performance.log");
```

请注意，配置的关键部分是`pattern:"$${marker:}"`。这将生成三个日志文件，每个文件都有一个特定marker的日志事件。Log4j会将带有SQL标记的日志事件路由到sql.log，带有PAYLOAD标记的日志事件到payload.log，等。

可以使用符号`${marker:name}`和`$${marker:name}`来检查标记是否存在，其中`name`是标记名称。如果标记存在，则表达式返回名称，否则返回null。

### 6.10 Structured Data Lookup

StructuredDataLookup与MapLookup非常相似，它将从StructuredDataMessages中检索值。除了Map值之外，它还将返回id的名称部分（不包括企业编号）和类型字段。下面的例子和MapMessage的例子之间的主要区别在于type是StructuredDataMessage的一个属性，而type必须是MapMessage中的Map中的一个项目。

```xml
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
  </Routes>
</Routing>
```

### 6.11 System Properties Lookup

由于使用System Properties定义应用程序内部和外部的值是很常见的，所以它们应该可以通过lookup来访问是很自然的。由于System properties通常是在应用程序之外定义的，因此会看到如下所示的常见情况：

```xml
<Appenders>
  <File name="ApplicationLog" fileName="${sys:logPath}/app.log"/>
</Appenders>
```

### 6.12 Web Lookup

WebLookup允许应用程序检索与ServletContext关联的变量。除了能够检索ServletContext中的各个字段外，WebLookup还支持查找作为属性存储或配置为初始化参数的值。下表列出了可以检索的各种键：

- attr.name：返回具有指定名称的ServletContext属性。
- contextPath：Web应用程序的上下文路径。
- effectiveMajorVersion：获取由此ServletContext表示的应用程序所基于的Servlet规范的主要版本。
- effectiveMinorVersion：获取由此ServletContext表示的应用程序所基于的Servlet规范的次要版本。
- initParam.name：返回具有指定名称的ServletContext初始化参数。
- majorVersion：返回此servlet容器所支持的Servlet API的主要版本。
- minorVersion：返回此servlet容器所支持的Servlet API的次要版本。
- rootDir：返回以`/`为参数时调用getRealPath的结果。
- serverInfo：返回运行servlet的servlet容器的名称和版本。
- servletContextName：返回在部署文件的display-name元素中定义的Web应用程序的名称。

首先检查任意指定的其他键名，以查看是否存在具有该名称的ServletContext属性，然后检查该名称的初始化参数是否存在。如果找到该键，则会返回相应的值。

```xml
<Appenders>
  <File name="ApplicationLog" fileName="${web:rootDir}/app.log"/>
</Appenders>
```