---
layout: post
title: "Log4j2 -- Appenders"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 7. Appenders

Appender负责将LogEvents交付到目的地。每个Appender必须实现Appender接口。大多数Appender将扩展AbstractAppender，它增加了Lifecycle和Filterable支持。Lifecycle允许组件在配置完成后完成初始化并在关闭期间执行清理。Filterable允许组件在事件处理过程中评估附加的filter。

Appender通常只负责将事件数据写入目的地。在大多数情况下，他们将格式化事件的责任委托给layout。一些appender会包装其他appender，以便修改LogEvent，或处理一个Appender中的故障，或者根据高级过滤标准将事件路由到下级Appender，或者提供不直接格式化事件以供查看类似的功能。

Appender总是有一个名字，以便它们可以从Logger中引用。

在下表中，Type列对应于预期的Java类型。对于非JDK类，除非另有说明，否则这些类通常应该位于Log4j Core中。

### 7.1 AsyncAppender

AsyncAppender接受对其他Appender的引用，并使LogEvent在单独的Thread上写入这些Appender。请注意，写入这些Appender时的exception将从应用程序中隐藏。AsyncAppender应该在它引用的appender之后进行配置，以允许其正常关闭。

默认情况下，AsyncAppender使用的`java.util.concurrent.ArrayBlockingQueue`不需要任何外部库。请注意，在使用此appender时，多线程应用程序应该小心：阻塞队列容易受到抢占锁的影响，测试表明当有更多线程同时记录时，性能可能会变差。可考虑使用无锁异步记录器以获得最佳性能。AsyncAppender参数如下：

- `AppenderRef，String`：要异步调用的Appender的名称，可配置多个AppenderRef元素。
- `blocking，boolean`：如果为true，则appender将等待队列中有空闲插槽。如果为false，则如果队列已满，则将事件写入错误appender。默认值是true。
- `shutdownTimeout，integer`：Appender在关闭时，应等待队列中的未处理日志事件刷新的时间。默认值为0意味着永远等待。
- `bufferSize，integer`：指定可以排队的最大事件数，默认值为1024。请注意，使用disruptor风格的BlockingQueue时，此缓冲区大小必须是2的幂。当应用程序的log的速度快于底层appender可以跟上填满队列的速度，行为将由AsyncQueueFullPolicy确定。
- `errorRef，String`：如果没有任何appender可以被调用，或者由于appender中的错误，或者队列已满而被调用的Appender的名称。如果未指定，则错误将被忽略。
- `filter，Filter`：用来确定该Appender是否应该处理事件。使用CompositeFilter时可以指定多个Filter。
- `name，String`：Appender的名称。
- `ignoreExceptions，boolean`：缺省值为true，导致在追加事件时遇到异常时，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `includeLocation，boolean`：提取位置是一项昂贵的操作（它可以使日志记录慢5到20倍）。为了提高性能，将日志事件添加到队列时，默认情况下不包括位置。你可以通过设置`includeLocation = true`来改变它。
- `BlockingQueueFactory，BlockingQueueFactory`：该元素重写要使用的BlockingQueue的类型。

即使在底层Appender无法跟上日志生成速率且队列正在填满的情况下，也可以使用一些系统properties来维护应用程序吞吐量。查看系统属性log4j2.AsyncQueueFullPolicy和log4j2.DiscardThreshold的详细信息。

典型的AsyncAppender配置可能如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </File>
    <Async name="Async">
      <AppenderRef ref="MyFile"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```

从Log4j 2.7开始，可以使用BlockingQueueFactory插件自定义实现BlockingQueue或TransferQueue。要覆盖默认的BlockingQueueFactory，请在`<Async/>`元素内指定插件，如下所示：

```xml
<Configuration name="LinkedTransferQueueExample">
  <Appenders>
    <List name="List"/>
    <Async name="Async" bufferSize="262144">
      <AppenderRef ref="List"/>
      <LinkedTransferQueue/>
    </Async>
  </Appenders>
  <Loggers>
    <Root>
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```

Log4j有以下BlockingQueueFactory实现：

- ArrayBlockingQueue：默认实现。
- DisruptorBlockingQueue：使用BlockingQueue的Conversant Disruptor实现。这个插件采用一个对应的可选的属性spinPolicy。
- JCToolsBlockingQueue：使用JCTools指定无锁队列绑定的MPSC。
- LinkedTransferQueue：使用Java7新实现的LinkedTransferQueue，注意这个队列不使用来自AsyncAppender的bufferSize属性，因为它不支持大容量。

### 7.2 CassandraAppender

CassandraAppender将其输出写入Apache Cassandra数据库。必须提前配置keyspace和表，并将该表的列映射到配置文件中。每列可以指定一个StringLayout（例如一个PatternLayout）以及一个可选的conversion类型，或者只指定org.apache.logging.log4j.spi.ThreadContextMap或org.apache.logging.log4j.spi.ThreadContextStack的conversion类型，将MDC或NDC分别存储在Map或list列中。与java.util.Date兼容的conversion类型将使用转换为该类型的日志事件时间戳（例如，使用java.util.Date填充Cassandra中的时间戳列类型）。
详见[CassandraAppender官方文档](http://logging.apache.org/log4j/2.x/manual/appenders.html#CassandraAppender)

### 7.3 ConsoleAppender

正如人们所预料的那样，ConsoleAppender将其输出写入System.out或System.err，而System.out是默认目，格式化LogEvent必须提供Layout。ConsoleAppender参数如下所示：

- `filter，filter`：用来确定特定事件是否应由该Appender处理，使用CompositeFilter可以使用多个Filter。
- `layout，layout`：用于格式化LogEvent的Layout。如果未提供Layout，则将使用默认为%m%n的Layout。
- `follow，boolean`：标识appender在完成配置后能否遵循System.setOut或System.setErr重新定义System.out或System.err。请注意，follow属性不能用于Windows上的Jansi，且不能与direct同时使用。
- `direct，boolean`：直接写入java.io.FileDescriptor并绕过java.lang.System.out/.err。当输出重定向到文件或其他进程时，可以提供高达10倍的性能提升。该属性不能用于Windows上的Jansi，且不能与follow同时使用。输出不会遵循java.lang.System.setOut()/.setErr()，并且可能会在多线程应用程序中可能与其他输出交织在java.lang.System.out/.err中。该功能自2.6.2以后新增，迄今为止仅在Linux和Windows上的Oracle JVM上进行了测试。
- `name，String`：Appender的名称。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `target，String`：可以是SYSTEM_OUT或SYSTEM_ERR，缺省值是SYSTEM_OUT。

Console配置示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <Console name="STDOUT" target="SYSTEM_OUT">
      <PatternLayout pattern="%m%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="STDOUT"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.4 FailoverAppender

FailoverAppender包装了一组appender。如果主Appender失效，则次级appender将按顺序尝试，直到成功或者无次级别Appender可尝试。FailoverAppender参数如下所示：

- `filter，filter`：用来确定特定事件是否应由该Appender处理，使用CompositeFilter可以使用多个Filter。
- `layout，layout`：用于格式化LogEvent的Layout。如果未提供Layout，则将使用默认为%m%n的Layout。
- `primary，String`：主appender的名字。
- `failovers，String[]`：次级appender的名字。
- `name，String`：Appender的名称。
- `retryIntervalSeconds，integer`：重试主Appender的冷却时间（秒），默认为60。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `target，String`：可以是SYSTEM_OUT或SYSTEM_ERR，缺省值是SYSTEM_OUT。

Failover配置示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log" filePattern="logs/app-%d{MM-dd-yyyy}.log.gz"
                 ignoreExceptions="false">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <TimeBasedTriggeringPolicy />
    </RollingFile>
    <Console name="STDOUT" target="SYSTEM_OUT" ignoreExceptions="false">
      <PatternLayout pattern="%m%n"/>
    </Console>
    <Failover name="Failover" primary="RollingFile">
      <Failovers>
        <AppenderRef ref="Console"/>
      </Failovers>
    </Failover>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Failover"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.5 FileAppender

FileAppender是一个OutputStreamAppender，它将日志写入fileName参数中指定的文件。 FileAppender使用FileManager（它扩展了OutputStreamManager）来实际执行文件的I/O。虽然来自不同Configuration的FileAppender不能共享，但如果Manager可以访问则可以共享FileManagers。例如，如果Log4j处于一个公共的ClassLoader中，那么servlet容器中的两个Web应用程序可以在拥有独自配置的情况下向同一个文件安全地写入。FileAppender的参数如下所示：

- `append，boolean`：如果为true（默认值），记录将被追加到文件的末尾。设置为false时，文件将在写入新记录之前清除。
- `bufferedIO，boolean`：如果为true（默认值），则记录将写入缓冲区，并且在缓冲区已满或设置了immediateFlush时，将记录数据写入磁盘。文件锁定不能与bufferedIO同时使用。性能测试表明，即使启用了immediateFlush，使用缓冲的I/O也可显着提高性能。
- `bufferSize，int`：当bufferedIO为true时，该参数代表缓冲区大小，默认值为8192字节。
- `createOnDemand，boolean`：appender按需创建文件。appender只在日志事件通过所有filter并且路由到此appender时创建文件。默认为false。
- `filter，Filter`：Filter用来确定该Appender是否应该处理该事件。使用CompositeFilter可以配置多个Filter。
- `fileName，String`：要写入的文件的名称。如果该文件或其任何父目录不存在，它们将被创建。
- `immediateFlush，boolean`：如果为true（默认值），每次写入后都会进行刷新。这将保证数据写入磁盘，但可能会影响性能。每次写入后刷新仅在与同步记录器一起使用时有用，即使将immediateFlush设置为false，异步记录器和appender也会在一批事件结束时自动刷新。这也保证数据写入磁盘，但效率更高。
- `layout，Layout`：用于格式化LogEvent的布局。如果未提供布局，则将使用%m%n的默认layout。
- `locking，boolean`：当设置为true时，I/O操作仅在保持文件锁定的情况下发生，这将允许在多个JVM和在多个潜在主机的FileAppender同时写入同一文件。这将对性能有显著影响，因此应谨慎使用。此外，在许多系统上，文件锁定是“被建议的”，这意味着其他应用程序可以在不获取锁的情况下而对文件执行操作。默认值是false。
- `name，String`：Appender的名称。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filePermissions，String`：每次创建文件时都应用POSIX格式的文件属性权限。底层文件系统应支持POSIX文件属性视图。例如：rw-------或rw-rw-rw-等。
- `fileOwner，String`：文件所有者在文件创建时进行定义。出于安全原因，更改文件的所有者可能受到限制，并且操作不允许抛出IOException。如果_POSIX_CHOWN_RESTRICTED对路径有效，则只有有效用户ID等于文件的拥有者标识的进程或具有适当权限的进程才可以更改文件的所有者。底层文件系统应支持文件owner属性视图。
- `fileGroup，String`：文件组在文件创建时进行定义。底层文件系统应支持POSIX文件属性视图。

File配置示例如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <File name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </File>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.6 FlumeAppender

关于Apache Flume详见[Flume官方文档](http://logging.apache.org/log4j/2.x/manual/appenders.html#FlumeAppender)。

### 7.7 JDBCAppender

JDBCAppender使用标准JDBC将日志事件写入关系数据库表。它可以配置为使用JNDI数据源或自定义工厂方法获取JDBC连接。无论采用哪种方式，它都**必须**由连接池支持。否则，记录性能将受到很大影响。如果配置的JDBC驱动程序支持批处理语句，并且将bufferSize配置为正数，则会对日志事件进行批处理。请注意，从Log4j2.8开始，有两种方法可以将日志事件配置为列映射：只允许字符串和时间戳的原始ColumnConfig样式，和使用Log4j的内置类型转换来允许更多的数据类型的ColumnMapping插件（这和Cassandra Appender中的插件是一样的）。JDBCAppender的参数如下所示：

- `name，String`：Appender的名字。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filter，Filter`：用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `bufferSize，int`：如果大于0的整数，则会使appender缓冲日志事件，并在缓冲区达到此大小时进行刷新。
- `connectionSource，ConnectionSource`：**必需**。检索数据库连接检索源。
- `tableName，String`：**必需**。要将日志事件插入到的数据库表的名称。
- `columnConfigs，ColumnConfig[]`：**必需**。关于记录事件数据应该插入的列的信息以及如何插入该数据。这由多个`<Column>`元素表示。
- `columnMappings，ColumnMapping[]`：**必需**。列映射配置的列表。每列必须指定一个列名称。每列可以有一个由其全限定类名指定的conversion类型。默认情况下，conversion类型是String。如果配置的类型与ReadOnlyStringMap/ThreadContextMap或ThreadContextStack的assignment相兼容，则该列将分别填充MDC或NDC（如何操作插入Map或List的值是由所使用的数据库确定的）。如果配置类型与java.util.Date分配兼容，则日志时间戳将转换为配置的日期类型。如果配置的类型与java.sql.Clob或java.sql.NClob的assignment相兼容，则格式化事件将分别设置为Clob或NClob（与传统的ColumnConfig插件类似）。如果给定了一个literal属性，那么它的值将在INSERT查询中按原样使用，而不进行任何转义。否则，指定的layout或pattern将转换为配置类型并存储在该列中。

在配置JDBCAppender时，必须指定Appender获取JDBC连接的ConnectionSource实现，必须使用`<DataSource>`或`<ConnectionFactory>`嵌套元素中的一个。DataSource的参数如下所示：

- `jndiName，String`：**必需**。javax.sql.DataSource绑定的完整的前缀JNDI名称，如java:/comp/env/jdbc/LoggingDatabase。DataSource必须支持连接池支持，否则，日志会很慢。

ConnectionFactory的参数如下所示：

- `class，Class`：**必需**。一个包含静态工厂方法以获取JDBC连接的类的完全限定名。
- `method，Method`：**必需**。用于获取JDBC连接的静态工厂方法的方法名。此方法必须没有参数，其返回类型必须是java.sql.Connection或DataSource。如果该方法返回Connection，则它必须从连接池中获取它们（并且在Log4j完成后它们将返回到连接池中）。否则，日志会很慢。如果该方法返回一个DataSource，那么DataSource将只被检索一次，并且必须有连接池支持。

在配置JDBCAppender时，使用嵌套的`<Column>`元素来指定表中应该写入哪些列以及如何写入。JDBCAppender使用这些信息来制定一个PreparedStatement以插入没有SQL注入漏洞的记录。Column的参数如下所示：

- name，String：**必需**。数据库列的名称。
- pattern，String：使用此属性可以使用PatternLayout模式在此列中的日志事件中插入一个或多个值。只需在该属性中指定任何合法模式即可。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- literal，String：使用此属性可在此列中插入文字值。该值不用任何引用而将被直接包含在insert SQL中（这意味着如果你希望它是一个字符串，你的值应该包含如下单引号，如literal ="'Literal String'"）。这对于不支持列标识的数据库特别有用。例如，如果您使用的是Oracle，则可以通过指定literal = "NAME_OF_YOUR_SEQUENCE.NEXTVAL"在ID列中插入唯一ID。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- isEventTimestamp，boolean：使用此属性在此列中插入事件时间戳，该列应该是SQL datetime。该值应作为java.sql.Types.TIMESTAMP插入。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- isUnicode，boolean：除非指定pattern，否则将忽略此属性。如果为true（默认），则该值将作为unicode插入（setNString或setNClob）。否则，该值将以非Unicode插入（setString或setClob）。
- isClob，boolean：除非指定pattern，否则将忽略此属性。使用此属性以指示该列存储Character Large Objects（CLOBs）。如果为true，则该值将作为CLOB（setClob或setNClob）插入。如果为false或省略（默认），则该值将作为VARCHAR或NVARCHAR（setString或setNString）插入。

以下是JDBCAppender的两个示例配置，以及一个使用Commons Pooling和Commons DBCP来共享数据库连接的示例工厂实现：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JDBC name="databaseAppender" tableName="dbo.application_log">
      <DataSource jndiName="java:/comp/env/jdbc/LoggingDataSource" />
      <Column name="eventDate" isEventTimestamp="true" />
      <Column name="level" pattern="%level" />
      <Column name="logger" pattern="%logger" />
      <Column name="message" pattern="%message" />
      <Column name="exception" pattern="%ex{full}" />
    </JDBC>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JDBC name="databaseAppender" tableName="LOGGING.APPLICATION_LOG">
      <ConnectionFactory class="net.example.db.ConnectionFactory" method="getDatabaseConnection" />
      <Column name="EVENT_ID" literal="LOGGING.APPLICATION_LOG_SEQUENCE.NEXTVAL" />
      <Column name="EVENT_DATE" isEventTimestamp="true" />
      <Column name="LEVEL" pattern="%level" />
      <Column name="LOGGER" pattern="%logger" />
      <Column name="MESSAGE" pattern="%message" />
      <Column name="THROWABLE" pattern="%ex{full}" />
    </JDBC>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

```java
package net.example.db;
 
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Properties;
 
import javax.sql.DataSource;
 
import org.apache.commons.dbcp.DriverManagerConnectionFactory;
import org.apache.commons.dbcp.PoolableConnection;
import org.apache.commons.dbcp.PoolableConnectionFactory;
import org.apache.commons.dbcp.PoolingDataSource;
import org.apache.commons.pool.impl.GenericObjectPool;
 
public class ConnectionFactory {
    private static interface Singleton {
        final ConnectionFactory INSTANCE = new ConnectionFactory();
    }
 
    private final DataSource dataSource;
 
    private ConnectionFactory() {
        Properties properties = new Properties();
        properties.setProperty("user", "logging");
        properties.setProperty("password", "abc123"); // or get properties from some configuration file
 
        GenericObjectPool<PoolableConnection> pool = new GenericObjectPool<PoolableConnection>();
        DriverManagerConnectionFactory connectionFactory = new DriverManagerConnectionFactory(
                "jdbc:mysql://example.org:3306/exampleDb", properties
        );
        new PoolableConnectionFactory(
                connectionFactory, pool, null, "SELECT 1", 3, false, false, Connection.TRANSACTION_READ_COMMITTED
        );
 
        this.dataSource = new PoolingDataSource(pool);
    }
 
    public static Connection getDatabaseConnection() throws SQLException {
        return Singleton.INSTANCE.dataSource.getConnection();
    }
}
```

### 7.8 JMSAppender

### 7.9 JPAAppender

### 7.10 HttpAppender

### 7.11 KafkaAppender

### 7.12 MemoryMappedFileAppender

### 7.13 NoSQLAppender

### 7.14 OutputStreamAppender

### 7.15 RandomAccessFileAppender

### 7.16 RewriteAppender

### 7.17 RollingFileAppender

### 7.18 RollingRandomAccessFileAppender

### 7.19 RoutingAppender

### 7.20 SMTPAppender

### 7.21 ScriptAppenderSelector

### 7.22 SocketAppender

### 7.23 SSL Configuration

### 7.24 SyslogAppender

### 7.25 ZeroMQ/JeroMQ Appender
