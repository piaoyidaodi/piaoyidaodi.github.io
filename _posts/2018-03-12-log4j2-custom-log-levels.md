---
layout: post
title: "Log4j2 -- Custom Log Levels"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 12. Custom Log Levels

### 12.1 Defining Custom Log Levels in Code

Log4j2支持自定义日志级别。自定义日志级别可以在代码或配置中定义。要在代码中定义自定义日志级别，请使用`Level.forName()`方法。此方法为指定的名称创建一个新的级别。定义日志级别后，可以通过调用`Logger.log()`方法并传递自定义日志级别来记录此级别的消息：

```java
// This creates the "VERBOSE" level if it does not exist yet.
final Level VERBOSE = Level.forName("VERBOSE", 550);

final Logger logger = LogManager.getLogger();
logger.log(VERBOSE, "a verbose message"); // use the custom VERBOSE level

// Create and use a new custom level "DIAG".
logger.log(Level.forName("DIAG", 350), "a diagnostic message");

// Use (don't create) the "DIAG" custom level.
// Only do this *after* the custom level is created!
logger.log(Level.getLevel("DIAG"), "another diagnostic message");

// Using an undefined level results in an error: Level.getLevel() returns null,
// and logger.log(null, "message") throws an exception.
logger.log(Level.getLevel("FORGOT_TO_DEFINE"), "some message"); // throws exception!
```

定义自定义日志级别时，intLevel参数（上例中的550和350）确定自定义级别与Log4j2内置的标准级别相关的位置。作为参考，下表显示了内建的intLevel在日志级别。

- OFF：0
- FATAL：100
- ERROR：200
- WARN：300
- INFO：400
- DEBUG：500
- TRACE：600
- ALL：Interger.MAX_VALUE

### 12.2 Defining Custom Log Levels in Configuration

自定义日志级别也可以在配置中定义。这对于在logger filter或appender filter中使用自定义级别很方便。与在代码中定义日志级别类似，必须首先定义自定义级别，然后才能使用它。如果logger或appender配置了未定义的级别，则该logger或appender将无效，并且不会处理任何日志事件。

CustomLevel配置元素创建一个自定义级别。它在内部调用上面讨论的相同的`Level.forName()`方法。CustomLevel参数如下所示：

- name，String：自定义级别的名字，该名字大小写敏感，全使用大写字母更方便。
- intLevel，integer：确定自定义级别相对于Log4j2内置的标准级别存在的位置。

以下示例显示定义一些自定义日志级别并使用自定义日志级别来过滤发送到控制台的日志事件的配置。

```java
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
  <!-- Define custom levels before using them for filtering below. -->
  <CustomLevels>
    <CustomLevel name="DIAG" intLevel="350" />
    <CustomLevel name="NOTICE" intLevel="450" />
    <CustomLevel name="VERBOSE" intLevel="550" />
  </CustomLevels>

  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <PatternLayout pattern="%d %-7level %logger{36} - %msg%n"/>
    </Console>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout pattern="%d %-7level %logger{36} - %msg%n"/>
    </File>
  </Appenders>
  <Loggers>
    <Root level="trace">
      <!-- Only events at DIAG level or more specific are sent to the console. -->
      <AppenderRef ref="Console" level="diag" />
      <AppenderRef ref="MyFile" level="trace" />
    </Root>
  </Loggers>
</Configuration>
```

### 12.3 Convenience Methods for the Built-in Log Levels

内置的日志级别在Logger接口上有一组便利的方法，使它们更易于使用。例如，Logger接口有24个`debug()`方法支持DEBUG级别：

```java
// convenience methods for the built-in DEBUG level
debug(Marker, Message)
debug(Marker, Message, Throwable)
debug(Marker, Object)
debug(Marker, Object, Throwable)
debug(Marker, String)
debug(Marker, String, Object...)
debug(Marker, String, Throwable)
debug(Message)
debug(Message, Throwable)
debug(Object)
debug(Object, Throwable)
debug(String)
debug(String, Object...)
debug(String, Throwable)
// lambda support methods added in 2.4
debug(Marker, MessageSupplier)
debug(Marker, MessageSupplier, Throwable)
debug(Marker, String, Supplier<?>...)
debug(Marker, Supplier<?>)
debug(Marker, Supplier<?>, Throwable)
debug(MessageSupplier)
debug(MessageSupplier, Throwable)
debug(String, Supplier<?>...)
debug(Supplier<?>)
debug(Supplier<?>, Throwable)
```

其他内置级别也有类似的方法。相比之下，自定义级别需要将日志级别作为额外参数传递。

```java
// need to pass the custom level as a parameter
logger.log(VERBOSE, "a verbose message");
logger.log(Level.forName("DIAG", 350), "another message");
```

如果在自定义级别上使用相同的易用性将会很好，因此在声明自定义VERBOSE/DIAG级别之后，我们可以使用如下代码：

```java
// nice to have: descriptive methods and no need to pass the level as a parameter
logger.verbose("a verbose message");
logger.diag("another message");
logger.diag("java 8 lambda expression: {}", () -> someMethod());
```

标准的Logger接口不能提供自定义级别的方便方法，但接下来的几节介绍了一种代码生成工具来创建用于使自定义级别与内置级别一样易于使用的记录器。

后续详见[官方文档](http://logging.apache.org/log4j/2.x/manual/customloglevels.html#AddingOrReplacingLevels)。