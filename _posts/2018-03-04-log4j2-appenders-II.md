---
layout: post
title: "Log4j2 -- Appenders-II"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 7. Appenders

### 7.14 OutputStreamAppender

OutputStreamAppender为许多其他Appender提供基础，例如将事件写入Output Stream的FileAppender和SocketAppender。OutputStreamAppender不能直接配置，它提供对immediateFlush和buffering的支持。OutputStreamAppender使用OutputStreamManager来处理实际的I/O，在多个配置中允许Appender共享该流。

### 7.15 RandomAccessFileAppender

RandomAccessFileAppender与标准FileAppender相似，除了它始终被缓冲（不能关闭）并在内部使用ByteBuffer和RandomAccessFile而不是BufferedOutputStream。在我们的测试中，与设置`bufferedIO = true`的FileAppender相比，性能提升了20-200％的。类似于FileAppender，RandomAccessFileAppender使用RandomAccessFileManager来实际执行文件I/O。虽然来自不同配置的RandomAccessFileAppender不能共享，但如果Manager可用，则可以使用RandomAccessFileManagers。例如，如果Log4j处于一个公用的ClassLoader中，那么servlet容器中的两个Web应用程序可以拥有自己的配置并安全地写入同一文件。

RandomAccessFileAppender的参数如下所示：

- `append，boolean`：如果为true（默认值），记录将被追加到文件的末尾。设置为false时，文件将在写入新记录之前清除。
- `fileName，String`：要写入的文件的名称。如果该文件或其任何父目录不存在，它们将被创建。
- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `immediateFlush，boolean`：如果为true（默认值），每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。每次写入后刷新仅在与同步记录器一起使用时有用，即使将immediateFlush设置为false，异步记录器和appender也会在一批事件结束时自动刷新。这也保证数据写入磁盘，但效率更高。
- `bufferSize，int`：该参数代表缓冲区大小，默认值为262144字节（256*1024）。
- `layout，Layout`：用于格式化LogEvent的布局。如果未提供布局，则将使用%m%n的默认layout。
- `name，String`：Appender的名称。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

RandomAccessFile的配置示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RandomAccessFile name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </RandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.16 RewriteAppender

RewriteAppender允许LogEvent在被另一个Appender处理之前进行操作。 这可以用来屏蔽密码等敏感信息或将信息注入每个事件。RewriteAppender必须配置一个RewritePolicy。RewriteAppender应该在它引用的任何Appender之后进行配置，以允许其正确关闭。

RewriteAppender参数如下所示：

- `AppenderRef，String`：在操作LogEvent后要调用的Appender的名称。可以配置多个AppenderRef元素。
- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `name，String`：Appender的名称。
- `rewritePolicy，RewritePolicy`：将会操作LogEvent的RewritePolicy。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

#### RewritePolicy
RewritePolicy是一个接口，允许实现在传递给Appender之前检查并可能修改LogEvent。RewritePolicy声明了一个必须实现的名为rewrite的方法。该方法传递给LogEvent并可以返回相同的事件或创建一个新的。

**MapRewitePolicy**

MapRewritePolicy将评估包含MapMessage的LogEvents，并将添加或更新Map的元素。

MapRewritePolicy的参数如下所示：

- `mode，String`：`Add`或`Update`。
- `keyValuePair，KeyValuePair[]`：键和对应值的数组。

下面示例展示了RewriteAppender配置添加一个产品键和值到MapMessage：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <MapRewritePolicy mode="Add">
        <KeyValuePair key="product" value="TestProduct"/>
      </MapRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

**PropertiesRewritePolicy**

PropertiesRewritePolicy会将在policy上配置的属性添加到正在记录的ThreadContext映射中。这些属性不会被添加到实际的ThreadContext映射中。属性值可能包含变量，这些变量将在处理配置和事件记录时进行评估。

PropertiesRewritePolicy参数如下所示：

- properties，Property[]：用于增加到ThreadContext映射中的，定义键和值的一个或多个属性。

下面示例展示了RewriteAppender配置添加一个产品键和值到MapMessage：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <PropertiesRewritePolicy>
        <Property name="user">${sys:user.name}</Property>
        <Property name="env">${sys:environment}</Property>
      </PropertiesRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

**LoggerNameLevelRewritePolicy**

您可以使用此policy通过更改事件level，使第三方代码中的日志记录器不那么健谈。LoggerNameLevelRewritePolicy将重写给定的记录器名称前缀的日志事件level。您可以使用记录器名前缀和一对level来配置LoggerNameLevelRewritePolicy，其中一对level分别定义了source level和target level。

LoggerNameLevelRewritePolicy的参数如下所示：

- logger，String：作为前缀使用的logger名，用来测试每个事件的logger名。
- LevelPair，KeyValuePair[]：键值对数组，每个键定义了source level，每个值定义了target level。

以下显示了一个RewriteAppender配置，将所有以com.foo.bar开头的记录器的级别从level INFO映射到DEBUG，从WARN级别映射到INFO。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Rewrite name="rewrite">
      <AppenderRef ref="STDOUT"/>
      <LoggerNameLevelRewritePolicy logger="com.foo.bar">
        <KeyValuePair key="INFO" value="DEBUG"/>
        <KeyValuePair key="WARN" value="INFO"/>
      </LoggerNameLevelRewritePolicy>
    </Rewrite>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Rewrite"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.17 RollingFileAppender

RollingFileAppender是一个OutputStreamAppender，它写入fileName参数中指定的File，并根据TriggeringPolicy和RolloverPolicy将文件roll over。RollingFileAppender使用RollingFileManager（它扩展了OutputStreamManager）来实际执行文件I/O并执行rollover。虽然来自不同配置的RolloverFileAppenders不能共享，但如果Manager可用RollingFileManagers可以共享。例如，如果Log4j处于一个公用的ClassLoader中，那么servlet容器中的两个Web应用程序可以拥有自己的配置并安全地写入同一文件。

RollingFileAppender需要一个TriggeringPolicy和一个RolloverStrategy。触发policy决定当在RolloverStrategy中定义的如何完成时rollover时是否应执行rollover。如果没有配置RolloverStrategy，RollingFileAppender将使用DefaultRolloverStrategy。自log4j-2.5以来，可以在DefaultRolloverStrategy中配置自定义删除操作，以便在rollover时运行。从2.8开始，如果没有配置文件名，则将使用DirectWriteRolloverStrategy而不是DefaultRolloverStrategy。从log4j-2.9开始，可以在DefaultRolloverStrategy中配置自定义的POSIX文件属性视图操作，以便在rollover时运行，如果未定义，将继承RollingFileAppender的POSIX文件属性视图。

RollingFileAppender不支持文件锁定。其参数如下所示：

- `append，boolean`：如果为true（默认值），记录将被追加到文件的末尾。设置为false时，文件将在写入新记录之前清除。
- `bufferedIO，boolean`：如果为true（默认值），则记录将写入缓冲区，并且在缓冲区已满或设置了immediateFlush时，将记录数据写入磁盘。文件锁定不能与bufferedIO同时使用。性能测试表明，即使启用了immediateFlush，使用缓冲的I/O也可显着提高性能。
- `bufferSize，int`：当bufferedIO为true时，该参数代表缓冲区大小，默认值为8192字节。
- `createOnDemand，boolean`：appender按需创建文件。appender只在日志事件通过所有filter并且路由到此appender时创建文件。默认为false。
- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `fileName，String`：要写入的文件的名称。如果该文件或其任何父目录不存在，它们将被创建。
- `filePattern，String`：归档日志文件的文件名称模式。模式的格式取决于所使用的RolloverPolicy。DefaultRolloverPolicy将接受与SimpleDateFormat兼容的date/time模式和/或代表整数计数器的%i。该模式还支持在运行时进行插值，因此任何lookup（例如DateLookup）都可以包含在模式中。
- `immediateFlush，boolean`：如果为true（默认值），每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。每次写入后刷新仅在与同步记录器一起使用时有用，即使将immediateFlush设置为false，异步记录器和appender也会在一批事件结束时自动刷新。这也保证数据写入磁盘，但效率更高。
- `layout，Layout`：用于格式化LogEvent的布局。如果未提供布局，则将使用%m%n的默认layout。
- `name，String`：Appender的名称。
- `policy，TriggeringPolicy`：用来确定rollover是否应当执行的policy。
- `strategy，RolloverStrategy`：用来确定归档文件名称和位置的策略。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filePermissions，String`：每次创建文件时都应用POSIX格式的文件属性权限。底层文件系统应支持POSIX文件属性视图。例如：rw-------或rw-rw-rw-等。
- `fileOwner，String`：文件所有者在文件创建时进行定义。出于安全原因，更改文件的所有者可能受到限制，并且操作不允许抛出IOException。如果_POSIX_CHOWN_RESTRICTED对路径有效，则只有有效用户ID等于文件的拥有者标识的进程或具有适当权限的进程才可以更改文件的所有者。底层文件系统应支持文件owner属性视图。
- `fileGroup，String`：文件组在文件创建时进行定义。底层文件系统应支持POSIX文件属性视图。

#### Triggering Policies

**Composite Triggering Policy**

CompositeTriggeringPolicy结合了多个触发policy，如果任何配置的policy返回true，则返回true。CompositeTriggeringPolicy仅通过在Policies元素中包装其他policy来配置。

例如，以下XML片段定义了JVM启动时，日志大小达到20兆字节且当前日期不再与日志的开始日期匹配时rollover日志的policy。

```xml
<Policies>
  <OnStartupTriggeringPolicy />
  <SizeBasedTriggeringPolicy size="20 MB" />
  <TimeBasedTriggeringPolicy />
</Policies>
```

**Cron Triggering Policy**

CronTriggeringPolicy基于cron表达式触发rollover。参数如下所示：

- schedule，String：cron表达式。该表达式与在Quartz调度器中允许的表达式相同。有关表达式的完整描述，请参阅CronExpression。
- evaluateOnStartup，boolean：启动时，将根据文件的上次修改时间戳评估cron表达式。如果cron表达式指示应该在该时间与当前时间之间发生rollover，则该文件将立即rollover。

**OnStartup Triggering Policy**

如果日志文件比当前JVM运行开始的时间早，并且满足或超过最小文件大小，则OnStartupTriggeringPolicy policy会导致rollover。OnStartupTriggeringPolicy参数如下所示：

- minSize，long：文件必须rollover的最小大小。设置为0都会导致无论文件大小如何都将导致rollover。默认值为1，这将防止空文件rollover。

### 7.18 RollingRandomAccessFileAppender

### 7.19 RoutingAppender

### 7.20 SMTPAppender

### 7.21 ScriptAppenderSelector

### 7.22 SocketAppender

### 7.23 SSL Configuration

### 7.24 SyslogAppender

### 7.25 ZeroMQ/JeroMQ Appender
