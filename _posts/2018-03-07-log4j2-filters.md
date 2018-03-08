---
layout: post
title: "Log4j2 -- Filters"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 8. Filters

filter允许评估日志事件以确定是否或如何发布它们。Filter将在其过滤器方法之一上调用，并将返回一个结果，该结果是具有3个值之一的枚举值，即ACCEPT，DENY或NEUTRAL。

过滤器可以配置在以下四个位置：

- Context-wide的Filter直接在配置中配置。被这些filter拒绝的事件不会被传递给记录器进行进一步处理。一旦一个事件被一个Context-wide过滤器所接受，它将不会被任何其他Context-wide Filters所评估，也不会使用Logger的等级来过滤该事件。然而，事件将由Logger和Appender的Filters进行评估。
- Logger Filter在指定的记录器上进行配置。这些在Logger的Context-wide Filters和Log Level之后进行评估。被这些过滤器拒绝的事件将被丢弃，并且事件不会传递给父记录器，而不考虑可加性设置。
- Appender Filter用于确定特定的Appender是否应该处理事件的格式和发布。
- Appender Reference Filter用于确定Logger是否应将事件路由到appender。

### 8.1 BurstFilter

BurstFilter提供了一种机制，通过在达到最大限制后自动丢弃事件来控制LogEvent的处理速率。

BurstFilter的参数如下所示：

- `level，String`：要过滤的消息级别。如果超过maxBurst，任何等于或低于此级别的内容都将被过滤掉。缺省值是WARN，表示任何高于警告的消息都将被记录，而不管burst的大小如何。
- `rate，float`：每秒允许的平均事件数量。
- `maxBurst，integer`：在事件被过滤的速率超过平均速率之前可能发生的最大事件数量。默认值是倍率的10倍。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

包含BurstFilter的配置如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.2 CompositeFilter

CompositeFilter提供了一种指定多个过滤器的方法。它作为filter元素添加到配置中，并包含要评估的其他filter。filters元素不接受任何参数。

包含CompositeFilter的配置可能如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Filters>
    <MarkerFilter marker="EVENT" onMatch="ACCEPT" onMismatch="NEUTRAL"/>
    <DynamicThresholdFilter key="loginId" defaultThreshold="ERROR"
                            onMatch="ACCEPT" onMismatch="NEUTRAL">
      <KeyValuePair key="User1" value="DEBUG"/>
    </DynamicThresholdFilter>
  </Filters>
  <Appenders>
    <File name="Audit" fileName="logs/audit.log">
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
    </File>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Logger name="EventLogger" level="info">
      <AppenderRef ref="Audit"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.3 DynamicThresholdFilter

DynamicThresholdFilter允许根据特定属性通过日志级别进行筛选。例如，如果在ThreadContext Map中捕获用户的loginId，那么可以仅为该用户启用debug日志记录。如果日志事件不包含指定的ThreadContext项目，则会返回NEUTRAL。

DynamicThresholdFilter的参数如下所示：

- `key，String`：ThreadContext Map中要比较的项目的名称。
- `defaultThreshold，String`：要过滤消息的级别。仅当日志事件包含指定的ThreadContext Map项并且其值与键/值对中的任何键不匹配时，默认threshold才适用。
- `keyValuePair，KeyValuePair[]`：一个或多个KeyValuePair元素，用于定义该键的匹配值以及该键匹配时的评估级别。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

以下为包含CompositeFilter的配置的示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <DynamicThresholdFilter key="loginId" defaultThreshold="ERROR"
                          onMatch="ACCEPT" onMismatch="NEUTRAL">
    <KeyValuePair key="User1" value="DEBUG"/>
  </DynamicThresholdFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.4 MapFilter

MapFilter允许过滤MapMessage中的数据元素。其参数如下所示：

- `keyValuePair，KeyValuePair[]`：一个或多个KeyValuePair元素，用于定义该键的匹配值以及该键匹配时的评估级别。
- `operator，String`：如果运算符是`or`，则任何一个键/值对的匹配都将被视为匹配，否则所有键/值对必须匹配。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

以下为包含MapFilter的配置的示例，将被用于输出特定的事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="eventId" value="Login"/>
    <KeyValuePair key="eventId" value="Logout"/>
  </MapFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置将显示与上例相同的行为，因为配置的唯一记录器是root。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
        <KeyValuePair key="eventId" value="Login"/>
        <KeyValuePair key="eventId" value="Logout"/>
      </MapFilter>
      <AppenderRef ref="RollingFile">
      </AppenderRef>
    </Root>
  </Loggers>
</Configuration>
```

此第三个示例配置将显示与前面的示例相同的行为，因为配置的唯一记录器是root，而且root仅配置有单个appender引用。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile">
        <MapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
          <KeyValuePair key="eventId" value="Login"/>
          <KeyValuePair key="eventId" value="Logout"/>
        </MapFilter>
      </AppenderRef>
    </Root>
  </Loggers>
</Configuration>
```

### 8.5 MarkerFilter

MarkerFilter将配置的Marker值与LogEvent中包含的Marker进行比较。当Marker名称与LogEvent的Marker或其父项之一匹配时，就会发生匹配。

MarkerFilter的参数如下所示：

- `marker，String`：将要比较的Marker的名字。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

示例配置如下，如果Marker匹配，则只允许appender写入事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <MarkerFilter marker="FLOW" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.6 RegexFilter

RegexFilter允许将格式化或未格式化的消息与正则表达式进行比较。RegexFilter参数如下所示：

- `regex，String`：正则表达式。
- `useRawMsg，boolean`：如果为true，则将使用未格式化的消息，否则将使用格式化的消息。默认值是false。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

示例配置如下，appender只允许写入包含单词“test”的事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <RegexFilter regex=".* test .*" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.7 Script

ScriptFilter执行一个返回true或false的脚本。

ScriptFilter参数如下所示：

- `script，Script、ScriptFile、ScriptRef`：指定执行逻辑的Script元素。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

Script参数如下所示：

- `configuration，Configuration`：拥有此ScriptFilter的Configuration。
- `level，Level`：与事件关联的日志记录级别。只有在配置为全局filter时才存在。
- `loggerName，String`：logger的名称。只有在配置为全局filter时才存在。
- `logEvent，LogEvent`：正在处理的LogEvent。配置为全局filter时不存在。
- `marker，Marker`：在日志记录调用时传递的Marker（如果有的话）。只有在配置为全局filter时才存在。
- `message，Message`：与日志记录调用关联的Message。只有在配置为全局filter时才存在。
- `parameters，Object[]`：传递给日志记录调用的参数。只有在配置为全局filter时才存在。某些Message将这些参数作为Message的一部分。
- `throwable，Throwable`：传递给日志记录调用的Throwable（如果有的话）。只有在配置为全局filter时才存在。一些Message将Throwable作为消息的一部分。
- `substitutor，StrSubstitutor`：StrSubstitutor用于替换lookup变量。

下面的示例显示了如何声明脚本字段，然后在特定组件中引用它们。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="ERROR">
  <Scripts>
    <ScriptFile name="filter.js" language="JavaScript" path="src/test/resources/scripts/filter.js" charset="UTF-8" />
    <ScriptFile name="filter.groovy" language="groovy" path="src/test/resources/scripts/filter.groovy" charset="UTF-8" />
  </Scripts>
  <Appenders>
    <List name="List">
      <PatternLayout pattern="[%-5level] %c{1.} %msg%n"/>
    </List>
  </Appenders>
  <Loggers>
    <Logger name="TestJavaScriptFilter" level="trace" additivity="false">
      <AppenderRef ref="List">
        <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
          <ScriptRef ref="filter.js" />
        </ScriptFilter>
      </AppenderRef>
    </Logger>
    <Logger name="TestGroovyFilter" level="trace" additivity="false">
      <AppenderRef ref="List">
        <ScriptFilter onMatch="ACCEPT" onMisMatch="DENY">
          <ScriptRef ref="filter.groovy" />
        </ScriptFilter>
      </AppenderRef>
    </Logger>
    <Root level="trace">
      <AppenderRef ref="List" />
    </Root>
  </Loggers>
</Configuration>
```

### 8.8 StructuredDataFilter

StructuredDataFilter是一个MapFilter，它也允许过滤事件ID，类型和消息。

- `keyValuePair，KeyValuePair[]`：一个或多个KeyValuePair元素，用于定义该键的匹配值以及该键匹配时的评估级别。
- `operator，String`：如果运算符是`or`，则任何一个键/值对的匹配都将被视为匹配，否则所有键/值对必须匹配。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

示例配置如下所示，StructuredDataFilter可以用来记录特定的事件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <StructuredDataFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="id" value="Login"/>
    <KeyValuePair key="id" value="Logout"/>
  </StructuredDataFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.9 ThreadContextMapFilter/ContextMapFilter

ThreadContextMapFilter或ContextMapFilter允许针对当前上下文中的数据元素进行过滤。 默认情况为ThreadContext Map。ContextMapFilter参数如下所示：

- `keyValuePair，KeyValuePair[]`：一个或多个KeyValuePair元素，用于定义该键的匹配值以及该键匹配时的评估级别。
- `operator，String`：如果运算符是`or`，则任何一个键/值对的匹配都将被视为匹配，否则所有键/值对必须匹配。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

ContextMapFilter配置如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <ContextMapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
    <KeyValuePair key="User1" value="DEBUG"/>
    <KeyValuePair key="User2" value="WARN"/>
  </ContextMapFilter>
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

ContextMapFilter也可以应用于记录器过滤：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <BurstFilter level="INFO" rate="16" maxBurst="100"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
      <ContextMapFilter onMatch="ACCEPT" onMismatch="NEUTRAL" operator="or">
        <KeyValuePair key="foo" value="bar"/>
        <KeyValuePair key="User2" value="WARN"/>
      </ContextMapFilter>
    </Root>
  </Loggers>
</Configuration>
```

### 8.10 ThresholdFilter

如果LogEvent中的级别与配置的级别相同或更具体（specific），则此过滤器将返回onMatch结果，否则返回onMismatch值。例如，如果ThresholdFilter配置了ERROR级别，并且LogEvent包含级别DEBUG，则会返回onMismatch值，因为ERROR事件比DEBUG更具体。

ThresholdFilter的参数如下所示：

- `level，String`：一个有效的可匹配的Level名。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

如果级别匹配，则只允许appender写入事件的示例配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <ThresholdFilter level="TRACE" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 8.11 TimeFilter

时间过滤器可用于限制过滤器仅限一天中的某一部分。其参数如下所示：

- `start，String`：HH:mm:ss格式的事件
- `end，String`：HH:mm:ss格式的事件。如果将结束时间设置的小于开始时间将导致无log实体被写出。
- `timezone，String`：与事件时间戳比较时所使用的时区。
- `onMatch，String`：filter匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是NEUTRAL。
- `onMismatch，String`：filter不匹配时采取的操作。可能是ACCEPT，DENY或者NEUTRAL。 缺省值是DENY。

仅允许appender在每天5:00至5:30使用默认时区写入事件的示例配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{MM-dd-yyyy}.log.gz">
      <TimeFilter start="05:00:00" end="05:30:00" onMatch="ACCEPT" onMismatch="DENY"/>
      <PatternLayout>
        <pattern>%d %p %c{1.} [%t] %m%n</pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```