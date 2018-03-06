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

**Log Archive File Attribute View Policy: Custom file attribute on Rollover**

Log4j-2.9引入了PosixViewAttribute操作，使用户可以更好地控制文件属性权限，所有者和组。PosixViewAttribute操作允许用户配置一个或多个条件来选择相对于基目录的合格文件。

PosixViewAttriute参数如下表所示：

- `basePath，String`：**必需**。开始扫描文件以应用属性的基本路径。
- `maxDepth，int`：要访问的目录的最大层级数。值为0意味着只有启动文件（基本路径本身）被访问，除非被安全管理器拒绝。Integer.MAX_VALUE的值表示应该访问所有级别。默认值为1，仅表示指定基本目录中的文件。
- `followLinks，boolean`：是否跟踪符号链接。默认为false。
- `pathConditions，PathCondition[]`：请参见DeletePathCondition
- `filePermissions，String`：当执行操作时以POSIX格式的文件属性权限。底层文件系统应支持POSIX文件属性视图。例如：`rw-------`或`rw-rw-rw-`等。
- `fileOwner，String`：定义何时执行操作的文件所有者。出于安全原因，更改文件的所有者可能受到限制并且操作不允许抛出IOException。如果_POSIX_CHOWN_RESTRICTED对路径有效，则只有有效用户ID等于文件用户ID或具有适当权限的进程才可以更改文件的所有权。底层文件系统应支持文件owner属性视图。
- `fileGroup，String`：定义何时执行的文件组。底层文件系统应支持POSIX文件属性视图。

以下是使用RollingFileAppender的配置示例，并为当前日志文件和滚动日志文件定义了不同的POSIX文件属性视图。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          		 filePattern="${baseDir}/$${date:yyyy-MM}/app-%d{yyyyMMdd}.log.gz"
                 filePermissions="rw-------">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy stopCustomActionsOnError="true">
        <PosixViewAttribute basePath="${baseDir}/$${date:yyyy-MM}" filePermissions="r--r--r--">
        	<IfFileName glob="*.gz" />
        </PosixViewAttribute>
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

RollingRandomAccessFileAppender类似于标准的RollingFileAppender，只是它始终被缓冲（这不能被关闭），并且在内部它使用ByteBuffer+RandomAccessFile而不是BufferedOutputStream。与设置bufferedIO = true的RollingFileAppender相比，测试结果显示性能提高了20-200％。RollingRandomAccessFileAppender将写入fileName参数中指定的File，并根据TriggeringPolicy和RolloverPolicy将文件rollover。类似于RollingFileAppender，RollingRandomAccessFileAppender使用RollingRandomAccessFileManager来实际执行文件I/O并执行rollover。虽然来自不同配置的RollingRandomAccessFileAppender无法共享，但如果Manager可用，RollingRandomAccessFileManagers可以被共享。例如，如果Log4j处于一个公共的ClassLoader中，那么servlet容器中的两个Web应用程序可以拥有自己的配置并安全地写入同一文件。

RollingRandomAccessFileAppender需要一个TriggeringPolicy和一个RolloverStrategy。triggering policy确定在RolloverStrategy中所定义的rollover方法是否应执行。如果没有配置RolloverStrategy，RollingRandomAccessFileAppender将使用DefaultRolloverStrategy。自log4j-2.5以来，可以在DefaultRolloverStrategy中配置自定义删除操作，以便在rollover时运行。

RollingRandomAccessFileAppender不支持文件锁定。

RollingRandomAccessFileAppender的参数如下所示：

- `append，boolean`：如果为true（默认值），记录将被追加到文件的末尾。设置为false时，文件将在写入新记录之前清除。
- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `fileName，String`：要写入的文件的名称。如果该文件或其任何父目录不存在，它们将被创建。
- `filePattern，String`：归档日志文件的文件名模式。模式的格式应该取决于所使用的RolloverPolicy。DefaultRolloverPolicy将接受与SimpleDateFormat兼容的date/time模式和/或代表整数计数器的%i。该模式还支持运行时插值，因此任何lookup（例如DateLookup）都可以包含在模式中。
- `immediateFlush，boolean`：如果为true（默认值），每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。每次写入后刷新仅在与同步记录器一起使用时有用，即使将immediateFlush设置为false，异步记录器和appender也会在一批事件结束时自动刷新。这也保证数据写入磁盘，但效率更高。
- `bufferSize，int`：该参数代表缓冲区大小，默认值为262144字节（256*1024）。
- `layout，Layout`：用于格式化LogEvent的布局。如果未提供布局，则将使用%m%n的默认layout。
- `name，String`：Appender的名称。
- `policy，TriggeringPolicy`：用于确定是否发生rollover的策略。
- `strategy，RolloverStrategy`：用于确定归档文件的名称和位置的策略。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filePermissions，String`：当执行操作时以POSIX格式的文件属性权限。底层文件系统应支持POSIX文件属性视图。例如：`rw-------`或`rw-rw-rw-`等。
- `fileOwner，String`：定义何时执行操作的文件所有者。出于安全原因，更改文件的所有者可能受到限制并且操作不允许抛出IOException。如果_POSIX_CHOWN_RESTRICTED对路径有效，则只有有效用户ID等于文件用户ID或具有适当权限的进程才可以更改文件的所有权。底层文件系统应支持文件owner属性视图。
- `fileGroup，String`：定义何时执行的文件组。底层文件系统应支持POSIX文件属性视图。

下面是使用RollingRandomAccessFileAppender的示例配置，其中包含基于time和size的triggering policies，将在同一天创建最多7个（1-7）存档，存储在基于当前年份和月份的目录中，并且将使用gzip压缩每个存档：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

第二个例子展示了一个rollover策略，在删除它们之前会保留多达20个文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
      <DefaultRolloverStrategy max="20"/>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

下面是使用RollingRandomAccessFileAppender的示例配置，其中包含基于time和size的triggering policies，将在同一天创建最多7个（1-7）存档，存储在基于当前年份和月份的目录中，并且将使用gzip压缩每个压缩文件，并在小时数可被6整除时每6小时rollover一次：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingRandomAccessFile name="RollingRandomAccessFile" fileName="logs/app.log"
                 filePattern="logs/$${date:yyyy-MM}/app-%d{yyyy-MM-dd-HH}-%i.log.gz">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy interval="6" modulate="true"/>
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingRandomAccessFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingRandomAccessFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.19 RoutingAppender

RoutingAppender评估LogEvents，然后将它们路由到下级Appender。 目标Appender可以是之前配置的appender，可以通过其名称来引用，或者也可以是根据需要动态创建的Appender。RoutingAppender应在其引用的任何Appender之后进行配置，以允许它正常关闭。

您还可以使用脚本配置RoutingAppender：您可以在appender启动时以及为日志事件选择路由时运行脚本。

RoutingAppender参数如下所示：

- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `name，String`：Appender的名称。
- `RewritePolicy，RewritePolicy`：操作LogEvent的RewritePolicy。
- `Routes，Routes`：包含一个或多个Route声明以指定选择Appender的标准。
- `Script，Script`：当Log4j启动RoutingAppender时该脚本运行，并返回一个确定默认Route的字符键。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

在此示例中，该脚本会使ServiceWindows路径成为Windows上的默认路由，并在所有其他操作系统上使用ServiceOther。请注意，ListAppender是我们的测试Appender之一，任何附加程序都可以使用，它仅用作简写。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" name="RoutingTest">
  <Appenders>
    <Routing name="Routing">
      <Script name="RoutingInit" language="JavaScript"><![CDATA[
        importPackage(java.lang);
        System.getProperty("os.name").search("Windows") > -1 ? "ServiceWindows" : "ServiceOther";]]>
      </Script>
      <Routes>
        <Route key="ServiceOther">
          <List name="List1" />
        </Route>
        <Route key="ServiceWindows">
          <List name="List2" />
        </Route>
      </Routes>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing" />
    </Root>
  </Loggers>
</Configuration>
```

#### Routes

Routes元素接受一个名为pattern的属性。该模式针对所有注册的lookup进行评估，结果用于选择Route。每个路由可以配置一个密钥。如果该键匹配评估模式的结果，那么将选择该Route。如果在路由上没有指定密钥，那么该路由是默认路由。只有一个路由可以配置为默认路由。

Routes元素可能包含一个Script子元素。如果指定，则每个日志事件运行该脚本并返回要使用的Route字符键。

您必须指定pattern属性或Script元素，但不能同时指定两者。

每条路线必须引用一个Appender。如果Route包含一个ref属性，那么Route将引用在配置中定义的Appender。如果Route包含Appender定义，则将在RoutingAppender的上下文中创建一个Appender，并且在每次通过Route引用匹配的Appender名称时将重用该Appender。

该脚本传递了以下变量：

- configuration，Configuration：活动的Configuration。
- staticVariables，Map：在此appender实例的所有脚本调用之间共享的一个Map。这是传递给Routes Script的相同映射。

在以下示例中，该脚本针对每个日志事件运行，并根据名为“AUDIT”的标记的存在route。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" name="RoutingTest">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT" />
    <Flume name="AuditLogger" compress="true">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Routing name="Routing">
      <Routes>
        <Script name="RoutingInit" language="JavaScript"><![CDATA[
          if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("AUDIT")) {
                return "AUDIT";
            } else if (logEvent.getContextMap().containsKey("UserId")) {
                return logEvent.getContextMap().get("UserId");
            }
            return "STDOUT";]]>
        </Script>
        <Route>
          <RollingFile
              name="Rolling-${mdc:UserId}"
              fileName="${mdc:UserId}.log"
              filePattern="${mdc:UserId}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="AuditLogger" key="AUDIT"/>
        <Route ref="STDOUT" key="STDOUT"/>
      </Routes>
      <IdlePurgePolicy timeToLive="15" timeUnit="minutes"/>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing" />
    </Root>
  </Loggers>
</Configuration>
```

#### Purge Policy

RoutingAppender可以配置一个PurgePolicy，其目的是停止并删除由RoutingAppender动态创建的休眠Appender。 Log4j目前提供IdlePurgePolicy作为可用于清理Appender的唯一PurgePolicy。IdlePurgePolicy接受2个属性：`timeToLive`，这是在没有发送任何事件发送给Appender的情况下其应该能够存活的timeUnit数量；`timeUnit`是与timeToLive属性一起使用的java.util.concurrent.TimeUnit的字符串表示形式。

以下是使用RoutingAppender将所有Audit事件路由到FlumeAppender的示例配置，所有其他事件都将路由到仅捕获特定事件类型的RollingFileAppender。请注意，AuditAppender是预定义的，而RollingFileAppenders是根据需要创建的。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Flume name="AuditLogger" compress="true">
      <Agent host="192.168.10.101" port="8800"/>
      <Agent host="192.168.10.102" port="8800"/>
      <RFC5424Layout enterpriseNumber="18060" includeMDC="true" appName="MyApp"/>
    </Flume>
    <Routing name="Routing">
      <Routes pattern="$${sd:type}">
        <Route>
          <RollingFile name="Rolling-${sd:type}" fileName="${sd:type}.log"
                       filePattern="${sd:type}.%i.log.gz">
            <PatternLayout>
              <pattern>%d %p %c{1.} [%t] %m%n</pattern>
            </PatternLayout>
            <SizeBasedTriggeringPolicy size="500" />
          </RollingFile>
        </Route>
        <Route ref="AuditLogger" key="Audit"/>
      </Routes>
      <IdlePurgePolicy timeToLive="15" timeUnit="minutes"/>
    </Routing>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Routing"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.20 SMTPAppender

发生特定日志记录事件时（通常为Error或Fatal），发送电子邮件。

此电子邮件中传递的日志记录事件的数量取决于BufferSize选项的值。SMTPAppender只在其循环缓冲区中保留最后一个BufferSize记录事件。这将内存需求保持在合理的水平，同时仍然提供有用的应用程序上下文。电子邮件中包含缓冲区中的所有事件。在触发电子邮件的事件之前，缓冲区将包含TRACE到WARN级别的最新事件。

默认行为是在记录ERROR或更高严重性事件时触发发送电子邮件并将其格式化为HTML。发送电子邮件的时间可以通过在Appender上设置一个或多个filter来控制。和其他Appender一样，可以通过指定Appender的layout来控制格式。

SMTPAppender的参数如下所示：

- `name，String`：Appender的名称。
- `from，String`：邮件发送者的地址。
- `replyTo，String`：用逗号隔开的回复邮件地址列表。
- `to，String`：用逗号隔开的接受者邮件地址列表。
- `cc，String`：用逗号隔开的CC地址列表。
- `bcc，String`：用逗号隔开的BCC地址列表。
- `subject，String`：邮件消息的主题。
- `bufferSize，integer`：将被包含在消息中的日志事件的最大缓冲数，默认为512。
- `layout，Layout`：格式化LogEvent的Layout，如果不设置Layout则默认使用HTML layout。
- `filter，Filter`：Filter用于确定该事件是否应当被此Appender操纵，可使用CompositeFilter配置多个Filter。
- `smtpDebug，boolean`：当设置为true时，在STDOUT上开启session debugging。默认为false。
- `smtpHost，String`：**必需**，将要发送的SMTP主机名。
- `smtpPassword，String`：认证SMTP服务器所需要的密码。
- `smtpPort，String`：发送的SMTP端口。
- `smtpProtocol，String`：SMTP传输协议（如smtps，默认为smtp）。
- `smtpUsername，String`：认证SMTP服务器所需要的用户名。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <SMTP name="Mail" subject="Error Log" to="errors@logging.apache.org" from="test@logging.apache.org"
          smtpHost="localhost" smtpPort="25" bufferSize="50">
    </SMTP>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Mail"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.21 ScriptAppenderSelector

构建配置时，ScriptAppenderSelector appender将调用Script来计算appender名称。Log4j然后使用ScriptAppenderSelector的名称创建一个AppenderSet下列出的appender。配置完成后，Log4j将忽略ScriptAppenderSelector。Log4j只从配置树构建一个选定的appender，并忽略其他AppenderSet子节点。

在以下示例中，该脚本返回名称List2。appender名称记录在ScriptAppenderSelector的名称下，而不是所选appender的名称，在本例中为SelectIt。

```xml
<Configuration status="WARN" name="ScriptAppenderSelectorExample">
  <Appenders>
    <ScriptAppenderSelector name="SelectIt">
      <Script language="JavaScript"><![CDATA[
        importPackage(java.lang);
        System.getProperty("os.name").search("Windows") > -1 ? "MyCustomWindowsAppender" : "MySyslogAppender";]]>
      </Script>
      <AppenderSet>
        <MyCustomWindowsAppender name="MyAppender" ... />
        <SyslogAppender name="MySyslog" ... />
      </AppenderSet>
    </ScriptAppenderSelector>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="SelectIt" />
    </Root>
  </Loggers>
</Configuration>
```

### 7.22 SocketAppender

SocketAppender是一个OutputStreamAppender，它将输出写入到由主机和端口指定的远程目标。数据可以通过TCP或UDP发送，并可以以任何格式发送。您可以选择使用SSL保护通信。

SocketAppender的参数如下所示：

- `name，String`：Appender的名字。
- ``host，String``：**必需**，监听日志事件的系统地址名称。
- `port，integer`：**必需**，监听日志事件的主机端口。
- `protocol，String`：TCP（默认），SSL或UDP。
- `SSL，SSLConfiguration`：包含KeyStore和TrustStore的配置。
- `filter，Filter`：Filter用于确定该事件是否应当被此Appender操纵，可使用CompositeFilter配置多个Filter。
- `immediateFail，boolean`：当设置为true时，日志事件将进行再连接并且如果socket不可用时立即fail。
- `immediateFlush，boolean`：当设置为true时（默认），每次写操作将进行一次flush。这将确保数据写入硬盘，但可能影响性能。
- `bufferedIO，boolean`：当设置为true时（默认），日志将首先被写入缓存，当缓存充满或immediateFlush被设置时，数据将被写入socket。
- `bufferSize，int`：当bufferedIO为true时，在此设置缓冲区大小，默认为8192字节。
- `layout，Layout`：**必需**，且无默认值。格式化LogEvent的Layout。从2.9版本新增，在先前的版本默认为SerializedLayout。
- `reconnectionDelayMilis，integer`：如果设置为大于0的值，则在发生错误后，SocketManager将在等待指定的毫秒数后尝试重新连接到服务器。如果重新连接失败，则会抛出异常（如果ignoreExceptions设置为false，可以被应用程序捕获）。
- `connectTimeoutMilis，integer`：毫秒格式的连接超时时间。默认为0（无限的超时，类似于Socket.connect()方法）。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

以下示例为一个不安全的TCP配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Socket name="socket" host="localhost" port="9500">
      <JsonLayout properties="true"/>
    </Socket>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="socket"/>
    </Root>
  </Loggers>
</Configuration>
```

以下为一个安全的SSL配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Socket name="socket" host="localhost" port="9500">
      <JsonLayout properties="true"/>
      <SSL>
        <KeyStore   location="log4j2-keystore.jks" passwordEnvironmentVariable="KEYSTORE_PASSWORD"/>
        <TrustStore location="truststore.jks"      passwordFile="${sys:user.home}/truststore.pwd"/>
      </SSL>
    </Socket>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="socket"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.23 SSL Configuration

可以配置多个appender为使用普通网络连接或安全套接字层（SSL）连接。本节介绍可用于SSL配置的参数。

- `protocol，String`：缺省为SSL。
- `KeyStore，KeyStore`：包含你的私钥和证书，并确定向远程主机发送哪个认证证书。
- `TrustStore，TrustStore`：包含远程用户的CA证书，确定远程证书是否应当被信任。

**KeyStore**

KeyStore旨在包含您的私钥和证书，并确定将哪些授权证书发送给远程主机。配置参数如下所示：

- `location，String`：KeyStore文件的路径。
- `password，char[]`：访问KeyStore所需的纯文本密码，不能与passwordEnvironmentVariable或passwordFile组合使用。
- `passwordEnvironmentVariable，String`：保存密码的环境变量的名称。不能与password或passwordFile组合使用。
- `passwordFile，String`：保存密码的文件的路径。不能与password或passwordEnvironmentVariable组合使用。
- `type，String`：可选的KeyStore类型，例如 JKS，PKCS12，PKCS11，BKS，Windows-MY/Windows-ROOT，KeychainStore等。默认为JKS。
- `keyManagerFactoryAlgorithm，String`：可选的KeyManagerFactory算法。默认值是SunX509。

**TrustStore**

TrustStore旨在远程方出示其证书时，保存你信任的远程方CA证书。确定是否应该信任远程认证凭证（以及连接）。

在某些情况下，它们可以是同一个store，但使用不同store通常更好（尤其是基于文件的store）。

TrustStore的配置参数如下：

- `location，String`：KeyStore文件的路径。
- `password，char[]`：访问KeyStore所需的纯文本密码，不能与passwordEnvironmentVariable或passwordFile组合使用。
- `passwordEnvironmentVariable，String`：保存密码的环境变量的名称。不能与password或passwordFile组合使用。
- `passwordFile，String`：保存密码的文件的路径。不能与password或passwordEnvironmentVariable组合使用。
- `type，String`：可选的KeyStore类型，例如 JKS，PKCS12，PKCS11，BKS，Windows-MY/Windows-ROOT，KeychainStore等。默认为JKS。
- `keyManagerFactoryAlgorithm，String`：可选的KeyManagerFactory算法。默认值是SunX509。

示例如下：

```xml
<SSL>
    <KeyStore   location="log4j2-keystore.jks" passwordEnvironmentVariable="KEYSTORE_PASSWORD"/>
    <TrustStore location="truststore.jks"      passwordFile="${sys:user.home}/truststore.pwd"/>
</SSL>
```

### 7.24 SyslogAppender

SyslogAppender是一个SocketAppender，它将输出写入由主机和端口指定的远程目标，格式符合BSD Syslog格式或RFC 5424格式。数据可以通过TCP或UDP发送。

SyslogAppender的参数如下所示：

- `advertise，boolean`：指示appender是否应advertise。
- `appName，String`：在RFC5424 syslog记录中用作APP-NAME的值。
- `charset，String`：将syslog String转换为字节数组时使用的字符集。该字符串必须是有效的字符集。如果未指定，则将使用默认系统字符集。
- `connectTimeoutMillis，integer`：连接超时（以毫秒为单位）。缺省值为0（无超时限制，如Socket.connect（）方法）。
- `enterpriseNumber，integer`：RFC5424中所述的IANA企业编号。
- `filter，Filter`：确定事件是否应由该Appender处理。使用CompositeFilter可以指定多个Filter。
- `facility，String`：facility用于尝试对消息进行分类。facility选项必须设置为“KERN”，“USER”，“MAIL”，“DAEMON”，“AUTH”，“SYSLOG”，“LPR”，“NEWS”，“UUCP”，“CRON”，“AUTHPRIV”，“FTP”，“NTP”，“AUDIT”，“ALERT”，“CLOCK”，“LOCAL0”，“LOCAL1”，“LOCAL2”，“LOCAL3”，“LOCAL4”，“LOCAL5”，“LOCAL6”，或“LOCAL7”。这些值可能为大写或小写字符。
- `format，String`：如果设置为“RFC5424”，数据将根据RFC5424格式化。否则，它将被格式化为BSD Syslog记录。请注意，虽然BSD Syslog记录要求为1024字节或更短，但SyslogLayout不会截断它们。RFC5424Layout也不会截断记录，因为接收器必须接受长达2048字节的记录，并可能接受更长的记录。
- `host，String`：正在监听日志事件的系统名称或地址。该参数是**必需**的。
- `id，String`：当进行格式化时根据RFC5424所使用的默认结构化数据ID。如果LogEvent包含StructuredDataMessage，则将使用来自消息的ID代替此值。
- `immediateFail，boolean`：当设置为true时，日志事件将不会尝试等待重新连接，如果socket不可用时立即fail。
- `immediateFlush，boolean`：当设置为true时（默认），每次写操作将进行一次flush。这将确保数据写入硬盘，但可能影响性能。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `includeMDC，boolean`：指示来自ThreadContextMap的数据是否将包含在RFC5424 Syslog记录中。默认为true。
- `layout，Layout`：覆盖在format中设置的自定义布局。
- `loggerFields，List of KeyValuePairs`：允许将任意PatternLayout模式作为指定的ThreadContext字段包含在内；未指定默认值。使用时请包含LoggerFields嵌套元素，其中包含一个或多个KeyValuePair元素。每个KeyValuePair 必须具有一个key属性，该属性指定将用于标识MDC Structured Data元素内的字段的键名；以及一个value属性，其指定了要用作值的PatternLayout模式。
- `mdcExcludes，String`：应该从LogEvent中排除的的mdc键列表（使用逗号分隔）。这与mdcIncludes属性是互斥的。该属性仅适用于RFC5424 syslog记录。
- `mdcIncludes，String`：应该包含在FlumeEvent中的mdc键列表（使用逗号分隔）。列表中未找到的任何MDC密钥都将被排除。该选项与mdcExcludes属性互斥。该属性仅适用于RFC5424 syslog记录。
- `mdcRequired，String`：必须存在于MDC中的mdc键列表（使用逗号分隔）。如果某个键不存在，则会抛出LoggingException。该属性仅适用于RFC5424 syslog记录。
- `mdcPrefix，String`：应该为每个MDC密钥添加前缀以便将其与event属性区分开。默认字符串是“mdc：”。该属性仅适用于RFC5424 syslog记录。
- `messageId，String`：在RFC5424 syslog记录的MSGID字段中使用的默认值。
- `name，String`：Appender的名称。
- `newLine，boolean`：如果为true，则将在syslog记录的末尾附加一个换行符。默认值是false。
- `port，integer`：正在侦听日志事件的主机上的端口。该参数必须指定。
- `protocol，String`：“TCP”或“UDP”。该参数是必需的。
- `SSL，SslConfiguration`：包含KeyStore和TrustStore的配置。请参阅SSL。
- `reconnectionDelayMillis，integer`：如果设置为大于0的值，则在发生错误后，SocketManager将在等待指定的毫秒数后尝试重新连接到服务器。如果重新连接失败，则会抛出异常（如果ignoreExceptions设置为false，可以被应用程序捕获）。

下面一个syslogAppender配置示例，配置了两个SyslogAppenders，一个使用BSD格式，一个使用RFC5424。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Syslog name="bsd" host="localhost" port="514" protocol="TCP"/>
    <Syslog name="RFC5424" format="RFC5424" host="localhost" port="8514"
            protocol="TCP" appName="MyApp" includeMDC="true"
            facility="LOCAL0" enterpriseNumber="18060" newLine="true"
            messageId="Audit" id="App"/>
  </Appenders>
  <Loggers>
    <Logger name="com.mycorp" level="error">
      <AppenderRef ref="RFC5424"/>
    </Logger>
    <Root level="error">
      <AppenderRef ref="bsd"/>
    </Root>
  </Loggers>
</Configuration>
```

对于SSL，该appender将其输出写入由主机和端口通过SSL指定的远程目标，格式符合BSD Syslog格式或RFC5424格式。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <TLSSyslog name="bsd" host="localhost" port="6514">
      <SSL>
        <KeyStore   location="log4j2-keystore.jks" passwordEnvironmentVariable="KEYSTORE_PASSWORD"/>
        <TrustStore location="truststore.jks"      passwordFile="${sys:user.home}/truststore.pwd"/>
      </SSL>
    </TLSSyslog>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="bsd"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.25 ZeroMQ/JeroMQ Appender

ZeroMQ appender使用JeroMQ库将日志事件发送到一个或多个ZeroMQ端点。

下面是一个简单的JeroMQ配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration name="JeroMQAppenderTest" status="TRACE">
  <Appenders>
    <JeroMQ name="JeroMQAppender">
      <Property name="endpoint">tcp://*:5556</Property>
      <Property name="endpoint">ipc://info-topic</Property>
    </JeroMQ>
  </Appenders>
  <Loggers>
    <Root level="info">
      <AppenderRef ref="JeroMQAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

下表列出了JeroMQ和ZeroMQ的参数列表：

- `name，String`：**必需**，Appender的名称。
- `Layout，Layout`：用于格式化LogEvent的布局。如果未提供布局，则将使用默认的%m%n pattern layout。
- `Filters，Filter`：过滤Appender的过滤器。
- `Properties，Property[]`：一个或多个属性元素，名为endpoint。
- `ignoreExceptions，boolean`：如果为true，则将记录和禁止异常。如果错误将被记录随后将传递给应用程序。
- `affinity，long`：ZMQ_AFFINITY选项。默认为0。
- `backlog，long`：ZMQ_BACKLOG选项。默认为100。
- `delayAttachOnConnect，boolean`：ZMQ_DELAY_ATTACH_ON_CONNECT选项。默认为false。
- `identity，byte[]`：ZMQ_IDENTITY选项。默认为none。
- `ipv4Only，boolean`：ZMQ_IPV4ONLY选项。默认为true。
- `linger，long`：ZMQ_LINGER选项。默认为-1。
- `maxMsgSize，long`：ZMQ_MAXMSGSIZE选项。默认为-1。
- `rcvHwm，long`：ZMQ_RCVHWM选项。默认为1000。
- `receiveBufferSize，long`：ZMQ_RCVBUF选项。默认为0。
- `receiveTimeOut，int`：ZMQ_RCVTIMEO选项。默认为-1。
- `reconnectIVL，long`：ZMQ_RECONNECT_IVL选项。默认为100。
- `reconnectIVLMax，long`：ZMQ_RECONNECT_IVL_MAX选项。默认为0。
- `sendBufferSize，long`：ZMQ_SNDBUF选项。默认为0。
- `sendTimeOut，int`：ZMQ_SNDTIMEO选项。默认为-1。
- `sndHwm，long`：ZMQ_SNDHWM选项。默认为1000。
- `tcpKeepAlive，int`：ZMQ_TCP_KEEPALIVE选项。默认为-1。
- `tcpKeepAliveCount，long`：ZMQ_TCP_KEEPALIVE_CNT选项。默认为-1。
- `tcpKeepAliveIdle，long`：ZMQ_TCP_KEEPALIVE_IDLE选项。默认为-1。
- `tcpKeepAliveInterval，long`：ZMQ_TCP_KEEPALIVE_INTVL选项。默认为-1。
- `xpubVerbose，boolean`：ZMQ_XPUB_VERBOSE选项。默认为false。