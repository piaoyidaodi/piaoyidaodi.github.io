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

**SizeBased Triggering Policy**

一旦文件达到指定的大小，SizeBasedTriggeringPolicy将导致rollover。大小可以用字节指定，后缀为KB，MB或GB，例如20MB。

**TimeBased Triggering Policy**

一旦date/time模式不再适用于活动文件，TimeBasedTriggeringPolicy会导致rollover。该policy接受一个interval属性，该属性基于时间模式和modulate布尔属性，确定应该发生rollover的频率。TimeBasedTriggeringPolicy参数如下所示：

- interval，integer：根据日期模式中最特定的时间单位进行rollover的频率。例如，使用小时数作为最特定项目的日期模式，并且每4小时发生4次rollover增量。默认值是1。
- modulate，boolean：指示是否应该调整时间间隔以在间隔边界上发生下一次翻转。例如，如果项目是小时，当前时间是凌晨3点，间隔是4，那么第一次rollover将在凌晨4点发生，然后下面的会在上午8点，中午，下午4点等发生。
- maxRandomDelay，integer：指示随机延迟rollover的最大秒数。 默认情况下是0表示没有延迟。此设置在多个应用程序配置为同时rollover日志文件的服务器上非常有用，并且可以实现随时分散负载。

#### Rollover Strategies

**Default Rollover Strategy**

默认的rollover策略同时接受日期/时间模式和来自RollingFileAppender本身指定的filePattern属性的整数。如果日期/时间模式存在，它将被替换为当前的日期和时间值。如果模式包含一个整数，它将在每次rollover时递增。如果模式同时包含日期/时间和模式中的整数，则整数将递增直到日期/时间模式的结果更改。如果文件格式以.gz，.zip，.bz2，.deflate，.pack200或.xz结尾，则将使用对应的压缩方案进行压缩。bzip2，Deflate，Pack200和XZ格式需要Apache Commons Compress。另外，XZ需要XZ for Java。该模式还可能包含可在运行时解析的lookup引用，如以下示例中所示。

默认rollover策略支持三种增加计数器的方法。首先是“固定窗口”策略，为了说明它是如何工作的，假设min属性设置为1，max属性设置为3，文件名为`foo.log`，文件名称模式为`foo-%i.log`。输出模式如下所示：

- `0，foo.log，-`：所有日志输出至初始化文件。
- `1，foo.log，foo-1.log`：第一次rollover时，foo.log重命名为foo-1.log；并新建一个foo.log文件写入。
- `2，foo.log，foo-1.log、foo-2.log`：第二次rollover时，foo-1.log重命名为foo-2.log；foo.log重命名为foo-1.log；并新建一个foo.log文件写入。
- `3，foo.log，foo-1.log、foo-2.log、foo-3.log`：第三次rollover时，foo-2.log重命名为foo-3.log；foo-1.log重命名为foo-2.log；foo.log重命名为foo-1.log；并新建一个foo.log文件写入。
- `4，foo.log，foo-1.log、foo-2.log、foo-3.log`：在第四次rollover和后续的rollover中，foo-3.log被删除；foo-2.log重命名为foo-3.log；foo-1.log重命名为foo-2.log；foo.log重命名为foo-1.log，并新建一个foo.log文件写入。

相反，当fileIndex属性设置为`max`，但所有其他设置相同时，将执行以下操作。

- `0，foo.log，-`：所有日志输出至初始化文件。
- `1，foo.log，foo-1.log`：第一次rollover时，foo.log重命名为foo-1.log；并新建一个foo.log文件写入。
- `2，foo.log，foo-1.log、foo-2.log`：第二次rollover时，foo.log重命名为foo-2.log；并新建一个foo.log文件写入。
- `3，foo.log，foo-1.log、foo-2.log、foo-3.log`：第三次rollover时，foo.log重命名为foo-3.log；并新建一个foo.log文件写入。
- `4，foo.log，foo-1.log、foo-2.log、foo-3.log`：在第四次rollover和后续的rollover中，foo-1.log被删除；foo-2.log重命名为foo-1.log；foo-3.log重命名为foo-2.log；foo.log重命名为foo-3.log，并新建一个foo.log文件写入。

最后，从版本2.8开始，如果fileIndex属性设置为`nomax`，那么最小值和最大值将被忽略，文件编号将递增1，并且每次rollover将递增，且无最大数量的文件。

DefaultRolloverStrategy参数如下所示：

- `fileIndex，String`：如果设置为max（默认值），索引较高的文件将比索引较小的文件更新。如果设置为“min”，则文件重命名和计数器将遵循上述固定窗口策略。
- `min，integer`：计数器的最小值。默认值是1。
- `max，integer`：计数器的最大值。一旦达到此值，旧的存档将在随后的转存中被删除。 默认值是7。
- `compressionLevel，integer`：设置0-9压缩级别，其中0 = 无，1 = 最佳速度，9 = 最佳压缩。仅适用于ZIP文件。
- `tempCompressedFilePattern，String`：压缩期间归档日志文件的文件名称的模式。

**DirectWrite Rollover Strategy**

DirectWriteRolloverStrategy会将日志事件直接写入由文件模式表示的文件。有了这个策略，文件重命名不会被执行。如果基于size的触发policy导致在指定的时间段内写入多个文件，则它们将从1开始编号并不断递增，直到发生基于time的rollover。

警告：如果文件模式有一个后缀，表明应压缩，当应用程序关闭时，当前文件不会被压缩。此外，如果时间改变使得文件模式不再与当前文件匹配，则它在启动时也不会被压缩。

DirectWriteRolloverStrategy的参数如下所示：

- `maxFiles，String`：在与文件模式匹配的时间段内允许的最大文件数。如果文件数超过，最旧的文件将被删除。如果指定，则该值必须大于1。如果该值小于0或缺省，则文件数量不受限制。
- `compressionLevel，integer`：设置0-9压缩级别，其中0 = 无，1 = 最佳速度，通过9 = 最佳压缩。仅适用于ZIP文件。
- `tempCompressedFilePattern，String`：压缩期间归档日志文件的文件名称的模式。

以下是使用RollingFileAppender并基于time和size的triggering policies的示例配置，将在同一天创建最多7个存档（1-7），存储在基于当前年份和月份的目录中，并且将使用gzip压缩每个存档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

第二个示例展示了一个rollover策略，在删除它们之前会保留多达20个文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="20"/>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

以下是使用RollingFileAppender以及基于time和size的triggering policies的示例配置，将在同一天创建最多7个存档（1-7），存储在基于当前年份和月份的目录中，并且将使用gzip压缩每个压缩文件，并在当前小时数可被6整除时每6小时rollover一次：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置使用RollingFileAppender以及基于cron和size的triggering policies，并直接写入无限数量的归档文件。cron触发器会导致每小时rollover一次，而文件大小限制为250MB：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" filePattern="logs/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <CronTriggeringPolicy schedule="0 0 * * * ?"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

此示例配置与前一个相同，但将每小时保存的文件数限制为10：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" filePattern="logs/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <CronTriggeringPolicy schedule="0 0 * * * ?"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DirectWriteRolloverStrategy maxFiles="10"/>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

**Log Archive Retention Policy: Delete on Rollover**

Log4j-2.5引入了Delete操作，使用户可以更加控制在rollover时删除的文件，而不是DefaultRolloverStrategy的max属性控制的可能会删除的文件。Delete操作允许用户配置一个或多个条件，以选择相对于基本目录的文件删除。

请注意，可以删除任何文件，而不仅仅是rollover日志文件，因此请谨慎使用此操作！使用testMode参数，您可以测试您的配置而不会意外删除错误的文件。

Delete的参数如下所示：

- `basePath，String`：**必需**。从何处开始扫描要删除的文件的基本路径。
- `maxDepth，int`：要访问的目录的最大级别数。除非安全管理器拒绝，值为0意味着只有启动文件（基本路径本身）被访问。Integer.MAX_VALUE的值表示应该访问所有级别。默认值为1，仅表示指定基本目录中的文件。
- `followLinks，boolean`：是否遵循符号链接。默认为false。
- `testMode，boolean`：如果为true，不删除文件而在INFO级别将消息打印到status logger。使用它来执行空运行以测试配置是否按预期工作。默认为false。
- `pathSorter，PathSorter`：实现PathSorter接口的插件，用于在选择要删除的文件之前对文件进行排序。默认是按最近修改事件排序文件。
- `pathConditions，PathCondition[]`：如果未指定ScriptCondition，则为必需。一个或多个PathCondition元素。如果指定了多个条件，则它们在删除之前都需要接受路径。条件可以嵌套，在这种情况下，仅当外部条件接受路径时才评估内部条件。如果条件不嵌套，可以按任何顺序进行评估。条件也可以通过使用IfAll，IfAny和IfNot复合条件与逻辑运算符AND，OR和NOT组合。
- `scriptCondition，ScriptCondition`：如果未指定PathConditions，则为必需。ScriptCondition元素用来指定脚本。ScriptCondition应包含一个Script，ScriptRef或ScriptFile元素，用于指定要执行的逻辑。该脚本传递了许多参数，其中包括在基本路径下找到的路径列表（一直到maxDepth），并且必须返回一个包含要删除的路径的列表。

IfFileName条件参数如下所示：

- `glob，String`：如果未指定正则表达式，则为必需。使用有限模式语言匹配相对路径（相对于基本路径），类似于正则表达式但语法更简单。
- `regex，String`：如果未指定glob，则为必需。使用Pattern类定义的正则表达式匹配相对路径（相对于基本路径）。
- `nestedConditions，PathCondition[]`：一组可选的嵌套PathConditions。 如果存在任何嵌套条件，则它们在删除之前都需要接受该文件。只有外部条件接受文件时才会评估（如果路径名称匹配）。

IfLastModified条件参数如下所示：

- `age，String`：**必需**。指定duration。该条件接受比指定时间长或更早的文件。
- `nestedConditions，PathCondition[]`：一组可选的嵌套PathConditions。 如果存在任何嵌套条件，则它们在删除之前都需要接受该文件。嵌套条件仅在外部条件接受文件时评估（如果文件足够大）。

IfAccumulatedFileCount条件参数如下所示：

- `exceeds，int`：**必需**。删除文件的阈值。
- `nestedConditions，PathCondition[]`：一组可选的嵌套PathConditions。 如果存在任何嵌套条件，则它们在删除之前都需要接受该文件。只有在外部条件接受文件时才会评估（如果超出阈值计数）。

IfAccumulatedFileSize条件参数如下所示：

- `exceeds，String`：**必需**。删除文件的累积文件阈值大小。可以用字节指定，后缀为KB，MB或GB，例如20MB。
- `nestedConditions，PathCondition[]`：一组可选的嵌套PathConditions。 如果存在任何嵌套条件，则它们在删除之前都需要接受该文件。嵌套条件仅在外部条件接受文件时评估（如果累计文件大小阈值已被超过）。

以下是配置示例，该配置使用RollingFileAppender，并将cron triggering policy配置为每天午夜触发。档案根据当前的年份和月份存储在一个目录中。基本目录下与`*/app-*.log.gz`的glob匹配的所有文件并满足60天或更长时间的将在rollover时删除。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyy-MM-dd}.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy>
        <Delete basePath="${baseDir}" maxDepth="2">
          <IfFileName glob="*/app-*.log.gz" />
          <IfLastModified age="60d" />
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

以下是使用RollingFileAppender以及基于time和size的triggering policy的示例配置，将在同一天根据当前年份和月份存储在目录中创建最多100个（1-100）存档，并且将使用gzip压缩每个压缩文件并每小时rollover一次。在每次rollover过程中，此配置将删除与`*/app-*.log.gz`且大于30天或更早的匹配文件，但保留最近的100GB或最近的10个文件，以先到者为准。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="100">
        <!--
        Nested conditions: the inner condition is only evaluated on files
        for which the outer conditions are true.
        -->
        <Delete basePath="${baseDir}" maxDepth="2">
          <IfFileName glob="*/app-*.log.gz">
            <IfLastModified age="30d">
              <IfAny>
                <IfAccumulatedFileSize exceeds="100 GB" />
                <IfAccumulatedFileCount exceeds="10" />
              </IfAny>
            </IfLastModified>
          </IfFileName>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

ScriptCondition的参数如下所示：

- `script，Script、ScriptFile、ScriptRef`：指定要执行的逻辑Script元素。脚本传递在基路径下找到的路径列表，并且必须返回要删除的路径作为`java.util.List<PathWithAttributes>`。

Script的参数如下所示：

- `basePath，java.nio.file.Path`：Delete操作开始扫描要删除的文件的目录。可用于pathList中的相对路径。
- `pathList，java.util.List<PathWithAttributes>`：在基本路径下找到的达到指定最大深度的路径列表，首先对最近修改过的文件进行排序。脚本可以自由修改并返回此列表。
- `statusLogger，StatusLogger`：在脚本执行期间可用于记录内部事件的StatusLogger。
- `configuration，Configuration`：拥有此ScriptCondition的配置。
- `substitutor，StrSubstitutor`：StrSubstitutor用于替换lookup变量。
- `?，String`：在配置中声明的任何属性。

以下是配置示例，该配置使用RollingFileAppender，并将cron triggering policy配置为每天午夜触发。档案根据当前的年份和月份存储在一个目录中。该脚本返回基路径下为13号星期五的rollover文件列表。Delete操作将删除脚本返回的所有文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyyMMdd}.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy>
        <Delete basePath="${baseDir}" maxDepth="2">
          <ScriptCondition>
            <Script name="superstitious" language="groovy"><![CDATA[
                import java.nio.file.*;

                def result = [];
                def pattern = ~/\d*\/app-(\d*)\.log\.gz/;

                pathList.each { pathWithAttributes ->
                  def relative = basePath.relativize pathWithAttributes.path
                  statusLogger.trace 'SCRIPT: relative path=' + relative + " (base=$basePath)";

                  // remove files dated Friday the 13th

                  def matcher = pattern.matcher(relative.toString());
                  if (matcher.find()) {
                    def dateString = matcher.group(1);
                    def calendar = Date.parse("yyyyMMdd", dateString).toCalendar();
                    def friday13th = calendar.get(Calendar.DAY_OF_MONTH) == 13 \
                                  && calendar.get(Calendar.DAY_OF_WEEK) == Calendar.FRIDAY;
                    if (friday13th) {
                      result.add pathWithAttributes;
                      statusLogger.trace 'SCRIPT: deleting path ' + pathWithAttributes;
                    }
                  }
                }
                statusLogger.trace 'SCRIPT: returning ' + result;
                result;
              ]] >
            </Script>
          </ScriptCondition>
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.18 RollingRandomAccessFileAppender

### 7.19 RoutingAppender

### 7.20 SMTPAppender

### 7.21 ScriptAppenderSelector

### 7.22 SocketAppender

### 7.23 SSL Configuration

### 7.24 SyslogAppender

### 7.25 ZeroMQ/JeroMQ Appender
