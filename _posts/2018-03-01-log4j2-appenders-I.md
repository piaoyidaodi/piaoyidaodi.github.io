---
layout: post
title: "Log4j2 -- Appenders-I"
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

- `name，String`：**必需**。数据库列的名称。
- `pattern，String`：使用此属性可以使用PatternLayout模式在此列中的日志事件中插入一个或多个值。只需在该属性中指定任何合法模式即可。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- `literal，String`：使用此属性可在此列中插入文字值。该值不用任何引用而将被直接包含在insert SQL中（这意味着如果你希望它是一个字符串，你的值应该包含如下单引号，如literal ="'Literal String'"）。这对于不支持列标识的数据库特别有用。例如，如果您使用的是Oracle，则可以通过指定literal = "NAME_OF_YOUR_SEQUENCE.NEXTVAL"在ID列中插入唯一ID。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- `isEventTimestamp，boolean`：使用此属性在此列中插入事件时间戳，该列应该是SQL datetime。该值应作为java.sql.Types.TIMESTAMP插入。pattern，literal或isEventTimestamp = true之一必须指定，但不能超过一个。
- `isUnicode，boolean`：除非指定pattern，否则将忽略此属性。如果为true（默认），则该值将作为unicode插入（setNString或setNClob）。否则，该值将以非Unicode插入（setString或setClob）。
- `isClob，boolean`：除非指定pattern，否则将忽略此属性。使用此属性以指示该列存储Character Large Objects（CLOBs）。如果为true，则该值将作为CLOB（setClob或setNClob）插入。如果为false或省略（默认），则该值将作为VARCHAR或NVARCHAR（setString或setNString）插入。

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

JMS Appender将格式化的日志事件发送到JMS目标。

请注意，在Log4j 2.0中，此appender被分为JMSQueueAppender和JMSTopicAppender。从Log4j 2.1开始，这些appender被合并到JMS Appender中，它不区分queue和topic。但是，使用`<JMSQueue/>`或`<JMSTopic/>`元素编写的2.0配置将继续与新的`<JMS/>`配置元素配合使用。JMSAppender的参数如下所示：

- `factoryBindingName，String`：**默认必需**，在Context中查找的ConnectionFactory的名称。这也可以是ConnectionFactory的任何子接口。
- `factoryName，String`：**默认必需**，在INITIAL_CONTEXT_FACTORY中所定义的初始上下文工厂的类全限定名称。如果在没有providerURL的情况下指定了一个factoryName，则会记录一条警告消息，因为这可能会导致问题。
- `filter，Filter`：**默认为null**，用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `layout，Layout`：**默认必需**，用于格式化LogEvent的layout。自2.9以来的新版本，在以前的版本中，SerializedLayout是默认的。
- `name，String`：**默认必需**，Appender的名称。
- `password，String`：**默认为null**，用于创建JMS连接的密码。
- `providerURL，String`：**默认必需**，在PROVIDER_URL定义的要使用的提供程序的URL。
- `destinationBindingName，String`：**默认必需**，用于查找Destination的名称。这可以是Queue或Topic，因此，属性名称为queueBindingName和topicBindingName是别名以保持与Log4j2.0 JMS appender的兼容性。
- `securityPrincipalName，String`：**默认为null**，在SECURITY_PRINCIPAL中所指定的Principal的标识的名称。如果在没有securityCredentials的情况下指定securityPrincipalName，则会输出一条警告消息，因为这可能会导致问题。
- `securityCredentials，String`：**默认为null**，在SECURITY_CREDENTIALS所指定的principal的安全凭证。
- `ignoreExceptions，boolean`：**默认为true**，当为true时，appending事件时所捕获的异常将作为内部日志并被忽略。当为false时，异常将传送给调用者时。当将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `immediateFail，boolean`：**默认为false**，当设置为true时日志事件不会等待尝试重新连接，并且如果JMS资源不可用，将立即失败。为2.9新增功能。
- `reconnectIntervalMillis，long`：**默认为5000**，如果设置为大于0的值，则在发生错误后，JMSManager将在等待指定的毫秒数后尝试重新连接到代理。如果重新连接失败，则会抛出异常（如果ignoreExceptions设置为false，可以被应用程序捕获）。为2.9新增功能。
- `urlPkgPrefixes，String`：**默认为null**，用于创建在URL_PKG_PREFIXES中定义的URL上下文工厂的工厂类类名的包前缀列表，以冒号分隔。
- `userName，String`：**默认为null**，用于创建JMS连接的用户标识。

以下是一个JMSAppender的配置示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp">
  <Appenders>
    <JMS name="jmsQueue" destinationBindingName="MyQueue"
         factoryBindingName="MyQueueConnectionFactory">
      <JsonLayout properties="true"/>
    </JMS>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="jmsQueue"/>
    </Root>
  </Loggers>
</Configuration>
```

要将Log4j MapMessages映射到JMS javax.jms.MapMessages，从2.9版本后，请使用`<MessageLayout/>`将appender的布局设置为MessageLayout：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp">
  <Appenders>
    <JMS name="jmsQueue" destinationBindingName="MyQueue"
         factoryBindingName="MyQueueConnectionFactory">
      <MessageLayout />
    </JMS>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="jmsQueue"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.9 JPAAppender

JPAAppender使用Java Persistence API 2.1将日志事件写入关系数据库表。 它要求类路径中具有API和相关的实现。它还需要一个经过配置的装饰实体来持久化所需的表格。该实体应扩展org.apache.logging.log4j.core.appender.db.jpa.BasicLogEventEntity（如果您主要想使用默认映射）并提供至少一个@Id属性或org.apache.logging.log4j.core.appender.db.jpa.AbstractLogEventWrapperEntity（如果您主要想实现自定义映射）。有关这两个类的更多信息，请参阅Javadoc及这两个类的源代码。JPAAppdner的参数如下所示：

- `name，String`：**必需**。Appender的名字。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filter，Filter`：**默认为null**，用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `bufferSize，int`：如果大于0的整数，则会使appender缓冲日志事件，并在缓冲区达到此大小时进行刷新。
- `entityClassName，String`：**必需**。具有将JPA注解映射到数据库表的LogEventWrapperEntity实现的标准名称。
- `persistenceUnitName，String`：**必需**。用于保存日志事件的JPA持久性单元的名称。

以下是JPAAppender的示例配置。 第一个XML示例是Log4j配置文件，第二个是persistence.xml文件。假定在这里使用了EclipseLink，且任何JPA 2.1或更高版本的提供者都会这样做。您应该**始终**创建一个**单独**的持久化单元进行日志记录，且原因有两个。首先，`<shared-cache-mode>`必须设置为`NONE`，这在通常的JPA使用中是不需要的。另外，出于性能方面的考虑，日志实体应该保留在其自己的持久性单元中并远离所有其他实体，并且您应该使用非JTA数据源。请注意，您的持久性单元还**必须**包含所有org.apache.logging.log4j.core.appender.db.jpa.converter类的`<class>`元素。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <JPA name="databaseAppender" persistenceUnitName="loggingPersistenceUnit"
         entityClassName="com.example.logging.JpaLogEntity" />
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
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
                                 http://xmlns.jcp.org/xml/ns/persistence/persistence_2_1.xsd"
             version="2.1">

  <persistence-unit name="loggingPersistenceUnit" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextMapAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextMapJsonAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextStackAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ContextStackJsonAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.MarkerAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.MessageAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.StackTraceElementAttributeConverter</class>
    <class>org.apache.logging.log4j.core.appender.db.jpa.converter.ThrowableAttributeConverter</class>
    <class>com.example.logging.JpaLogEntity</class>
    <non-jta-data-source>jdbc/LoggingDataSource</non-jta-data-source>
    <shared-cache-mode>NONE</shared-cache-mode>
  </persistence-unit>
</persistence>
```

```java
package com.example.logging;
...
@Entity
@Table(name="application_log", schema="dbo")
public class JpaLogEntity extends BasicLogEventEntity {
    private static final long serialVersionUID = 1L;
    private long id = 0L;

    public TestEntity() {
        super(null);
    }
    public TestEntity(LogEvent wrappedEvent) {
        super(wrappedEvent);
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    public long getId() {
        return this.id;
    }

    public void setId(long id) {
        this.id = id;
    }

    // If you want to override the mapping of any properties mapped in BasicLogEventEntity,
    // just override the getters and re-specify the annotations.
}
```

```java
package com.example.logging;
...
@Entity
@Table(name="application_log", schema="dbo")
public class JpaLogEntity extends AbstractLogEventWrapperEntity {
    private static final long serialVersionUID = 1L;
    private long id = 0L;

    public TestEntity() {
        super(null);
    }
    public TestEntity(LogEvent wrappedEvent) {
        super(wrappedEvent);
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "logEventId")
    public long getId() {
        return this.id;
    }

    public void setId(long id) {
        this.id = id;
    }

    @Override
    @Enumerated(EnumType.STRING)
    @Column(name = "level")
    public Level getLevel() {
        return this.getWrappedEvent().getLevel();
    }

    @Override
    @Column(name = "logger")
    public String getLoggerName() {
        return this.getWrappedEvent().getLoggerName();
    }

    @Override
    @Column(name = "message")
    @Convert(converter = MyMessageConverter.class)
    public Message getMessage() {
        return this.getWrappedEvent().getMessage();
    }
    ...
}
```

### 7.10 HttpAppender

HttpAppender通过HTTP发送日志事件。必须提供layout来格式化LogEvent。

将根据layout设置Content-Type标头。可以使用嵌入的Property元素指定其他header。

将等待来自服务器的响应，并且如果没有收到2xx响应则抛出错误。

实现自HttpURLConnection。

HttpAppender的参数如下所示：

- `name，String`：Appender的名字。
- `filter，Filter`：用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `layout，Layout`：格式化LogEvent的layout。
- `Ssl，SSLConfiguration`：包含https的KeyStore和TrustStore的配置。可选，如果未指定则使用Java运行时的默认值。
- `verifyHostname，boolean`：是否针对证书进行服务器主机名验证。仅对https有效。可选，默认为true。
- `url，String`：要使用的URL。URL协议必须是http或https。
- `method，String`：要使用的HTTP方法。可选，默认为POST。
- `connectTimeoutMillis，integer`：连接超时（以毫秒为单位）。可选，默认值为0（无限超时）。
- `readTimeoutMillis，integer`：套接字读取超时（以毫秒为单位）。可选，默认值为0（无限超时）。
- `headers，Property[]`：要使用的其他HTTP标头。这些值支持lookup。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

HttpAppender配置如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ...
  <Appenders>
    <Http name="Http" url="https://localhost:9200/test/log4j/">
      <Property name="X-Java-Runtime" value="$${java:runtime}" />
      <JsonLayout properties="true"/>
      <SSL>
        <KeyStore   location="log4j2-keystore.jks" passwordEnvironmentVariable="KEYSTORE_PASSWORD"/>
        <TrustStore location="truststore.jks"      passwordFile="${sys:user.home}/truststore.pwd"/>
      </SSL>
    </Http>
  </Appenders>
```

### 7.11 KafkaAppender

KafkaAppender将日志事件传送至Apache Kafka topic。每个日志事件都作为Kafaka记录发送。KafkaAppender参数如下所示：

- `topic，String`：**必需**。将使用的Kafka topic。
- `key，String`：随每个消息将发送给Kafka的密钥。可选值，默认为null。任何lookup都可以包含在内。
- `filter，Filter`：用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `layout，Layout`：格式化LogEvent的layout。
- `name，String`：Appender的名字。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `syncSend，boolean`：缺省值为true，这会导致发送阻塞，直到Kafka服务器确认记录。当设置为false时发送后立即返回，从而实现更低的延迟和更高的吞吐量。自2.8以来新增。请注意，这是一个新增功能，并没有经过广泛的测试。发送到Kafka的任何failure都将作为StatusLogger的错误报告，并且日志事件将被丢弃（ignoreExceptions参数将无效）。日志事件可能不按顺序到达Kafka服务器。
- `properties，Property[]`：可以在Kafka producer properties中设置属性。您需要设置bootstrap.servers属性，其他属性具有适当的默认值。不要设置value.serializer和key.serializer属性。

KafkaAppender配置示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ...
  <Appenders>
    <Kafka name="Kafka" topic="log-test">
      <PatternLayout pattern="%date %message"/>
        <Property name="bootstrap.servers">localhost:9092</Property>
    </Kafka>
  </Appenders>
```
这个appender默认是同步的，并且会阻塞直到Kafka服务器确认记录为止，可以使用timeout.ms属性设置超时时间（默认为30秒）。用AsyncAppender包装和/或将syncSend设置为false以实现异步日志记录。

这个appender需要Kafka客户端库。请注意，您使用的Kafka客户端库版本应与所使用的Kafka服务器相匹配。

注意：确保不要让org.apache.kafka在DEBUG级别将日志输出到Kafka appender，因为这会导致递归日志记录：

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ...
  <Loggers>
    <Root level="DEBUG">
      <AppenderRef ref="Kafka"/>
    </Root>
    <Logger name="org.apache.kafka" level="INFO" /> <!-- avoid recursive logging -->
  </Loggers>
```

### 7.12 MemoryMappedFileAppender

自2.1起新增。请注意，这是一个新增功能，尽管它已在多个平台上进行过测试，但没有其他file appender那么多。

MemoryMappedFileAppender将指定文件的一部分映射到内存中，并将日志事件写入此内存，依赖操作系统的虚拟内存管理器将更改同步到存储设备。使用内存映射文件的主要好处是I/O性能。这个appender可以简单地改变程序的本地内存，而不是将系统调用写入磁盘。而且，在大多数操作系统中，实际映射的内存区域是内核的页面缓存（文件缓存），这意味着不需要在用户空间创建副本。（TODO：比较此Appender与RandomAccessFileAppender和FileAppender的性能的性能测试。）

将文件区域映射到内存有一些开销，特别是区域非常大时（半个GB或更多）。默认区域大小为32MB，这应该在重映射操作的频率和持续时间之间达到合理的平衡。 （TODO：性能测试重新映射各种大小。）

类似于FileAppender和RandomAccessFileAppender，MemoryMappedFileAppender使用MemoryMappedFileManager来实际执行文件I/O。虽然来自不同配置的MemoryMappedFileAppender不能共享，但如果Manager可用，则可以共享MemoryMappedFileManagers。例如，如果Log4j处于一个可以共用的ClassLoader中，那么servlet容器中的两个Web应用程序可以拥有自己的配置并安全地写入同一文件。

MemoryMappedFile配置参数如下所示：

- `append，boolean`：默认为true，记录将被追加到文件的末尾。设置为false时，文件将在写入新记录之前清除。
- `fileName，String`：要写入的文件的名称。如果该文件或其任何父目录不存在，它们将被创建。
- `filter，Filter`：用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `immediateFlush，boolean`：设置为true时，每次写入后都会调用MappedByteBuffer.force()。这将保证数据被写入存储设备。此参数的默认值为false。这意味着即使Java进程崩溃，数据也会写入存储设备，但如果操作系统崩溃，则可能会丢失数据。请注意，在每个日志事件中手动强制同步会失去使用内存映射文件的大部分性能优势。每次写入后刷新仅在与同步logger一起使用此appender时有用。即使将immediateFlush设置为false，异步logger和appender也会在一批事件结束时自动刷新。这也保证数据写入磁盘，但效率更高。
- `regionLength，int`：映射区域的长度，默认为32 MB。该参数必须介于256和1,073,741,824之间（1GB或2^30）;超出此范围的值将被调整为最接近的有效值。Log4j会将指定值四舍五入到最接近的2的幂。
- `layout，Layout`：格式化LogEvent的layout。如果未提供layout，则将使用%m%n的默认layout。
- `name，String`：Appender的名字。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。

MemoryMappedFileAppender配置示例如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <MemoryMappedFile name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
    </MemoryMappedFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="MyFile"/>
    </Root>
  </Loggers>
</Configuration>
```

### 7.13 NoSQLAppender

NoSQLAppender使用内部轻量级提供程序接口将日志事件写入NoSQL数据库。目前为MongoDB和Apache CouchDB提供了实现，并且编写自定义提供程序非常简单。NoSQLAppender参数如下所示：

- `name，String`：Appender的名字。
- `ignoreExceptions，boolean`：缺省值为true，此时若在追加事件时遇到异常，这些事件将在内部记录并被忽略。当设置为false时，异常将被传递给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `filter，Filter`：用来确定该事件是否应当由Appender处理，使用CompositeFilter可以配置多个Filter。
- `bufferSize，int`：如果大于0的整数，则会使appender缓冲日志事件，并在缓冲区达到此大小时进行刷新。
- `NoSqlProvider NoSQLProvider NoSQLProvider<C extends NoSQLConnection<W, T extends NoSQLObject<W>>>`：**必需**。提供与所选NoSQL数据库连接的NoSQL提供程序。

您可以通过在`<NoSql>`元素中指定适当的配置元素来指定要使用哪个NoSQL提供程序。目前支持的类型是`<MongoDb>`和`<CouchDb>`。要创建您自己的自定义provider程序，请阅读NoSQLProvider，NoSQLConnection和NoSQLObject类的JavaDoc以及有关创建Log4j plugin的文档。我们建议您查看MongoDB和CouchDB提供商的源代码，作为创建自己provider的指南。

MongoDBProvider的参数如下所示：

- `collectionName，String`：**必需**。要插入事件的MongoDB集合的名称。
- `writeConcernConstant，Field`：默认情况下，MongoDB provider使用指令com.mongodb.WriteConcern.ACKNOWLEDGED插入日志记录。使用此可选属性指定除ACKNOWLEDGED以外的常量的名称。
- `writeConcernConstantClass，Class`：如果指定writeConcernConstant，你可以使用这个属性来指定不同于com.mongodb.WriteConcern的类以找到（创建自己的定制指令）的常数。
- `factoryClassName，Class`：要提供到MongoDB数据库的连接，可以使用此属性和factoryMethodName以指定获取连接的类和静态方法。该方法必须返回一个com.mongodb.DB或一个com.mongodb.MongoClient。如果DB未通过身份验证，则还必须指定username和password。如果使用工厂方法提供连接，则不得指定databaseName，server或port属性。
- `factoryMethodName，Method`：请参阅属性factoryClassName。
- `databaseName，String`：如果您未指定用于提供MongoDB连接的factoryClassName和factoryMethodName，则必须使用此属性指定MongoDB数据库名称。您还必须指定username和password。您也可以选择指定一个server（默认为localhost）和一个port（默认为默认的MongoDB端口）。
- `server，String`：请参阅databaseName属性的文档。
- `port，int`：请参阅databaseName属性的文档。
- `username，String`：请参阅databaseName和factoryClassName属性的文档。
- `password，String`：请参阅databaseName和factoryClassName属性的文档。
- `capped，boolean`：启用对capped collection的支持。
- `collectionSize，int`：指定启用时使用的capped collection的字节大小。最小为4096字节，最大将增加到接近256的整数倍。

CouchDBProvider的参数如下所示：

factoryClassName类要提供到CouchDB数据库的连接，可以使用此属性和factoryMethodName来指定从中获取连接的类和静态方法。 该方法必须返回org.lightcouch.CouchDbClient或org.lightcouch.CouchDbProperties。 如果使用工厂方法提供连接，则不得指定数据库名称，协议，服务器，端口，用户名或密码属性。
factoryMethodName方法请参阅属性factoryClassName的文档。
databaseName字符串如果您未指定用于提供CouchDB连接的factoryClassName和factoryMethodName，则必须使用此属性指定CouchDB数据库名称。 您还必须指定用户名和密码。 您也可以选择指定协议（默认为http），服务器（默认为localhost）和端口（http默认为80，https默认为443）。
协议字符串必须是“http”或“https”。 请参阅属性databaseName的文档。
- `factoryClassName，Class`：要提供到CouchDB数据库的连接，可以使用此属性和factoryMethodName以指定获取连接的类和静态方法。该方法必须返回一个org.lightcouch.CouchDbClient或org.lightcouch.CouchDbProperties。如果使用工厂方法提供连接，则不得指定databaseName，protocol，server，port，username或password属性。
- `factoryMethodName，Method`：请参阅属性factoryClassName。
- `databaseName，String`：如果您未指定用于提供CouchDB连接的factoryClassName和factoryMethodName，则必须使用此属性指定CouchDB数据库名称。您还必须指定username和password。您也可以选择指定一个protocol（默认为http），server（默认为localhost）和一个port（http默认为80，https默认为443）。
- `protocol，String`：必需为http或https，请参阅databaseName属性的文档。
- `server，String`：请参阅databaseName属性的文档。
- `port，int`：请参阅databaseName属性的文档。
- `username，String`：请参阅databaseName和factoryClassName属性的文档。
- `password，String`：请参阅databaseName和factoryClassName属性的文档。

NoSQLAppender的一些示例如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error">
  <Appenders>
    <NoSql name="databaseAppender">
      <MongoDb databaseName="applicationDb" collectionName="applicationLog" server="mongo.example.org"
               username="loggingUser" password="abc123" />
    </NoSql>
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
    <NoSql name="databaseAppender">
      <MongoDb collectionName="applicationLog" factoryClassName="org.example.db.ConnectionFactory"
               factoryMethodName="getNewMongoClient" />
    </NoSql>
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
    <NoSql name="databaseAppender">
      <CouchDb databaseName="applicationDb" protocol="https" server="couch.example.org"
               username="loggingUser" password="abc123" />
    </NoSql>
  </Appenders>
  <Loggers>
    <Root level="warn">
      <AppenderRef ref="databaseAppender"/>
    </Root>
  </Loggers>
</Configuration>
```

下列示例表示，如果使用JSON格式，日志将如何放入NoSQL数据库中：

```json
{
    "level": "WARN",
    "loggerName": "com.example.application.MyClass",
    "message": "Something happened that you might want to know about.",
    "source": {
        "className": "com.example.application.MyClass",
        "methodName": "exampleMethod",
        "fileName": "MyClass.java",
        "lineNumber": 81
    },
    "marker": {
        "name": "SomeMarker",
        "parent" {
            "name": "SomeParentMarker"
        }
    },
    "threadName": "Thread-1",
    "millis": 1368844166761,
    "date": "2013-05-18T02:29:26.761Z",
    "thrown": {
        "type": "java.sql.SQLException",
        "message": "Could not insert record. Connection lost.",
        "stackTrace": [
                { "className": "org.example.sql.driver.PreparedStatement$1", "methodName": "responder", "fileName": "PreparedStatement.java", "lineNumber": 1049 },
                { "className": "org.example.sql.driver.PreparedStatement", "methodName": "executeUpdate", "fileName": "PreparedStatement.java", "lineNumber": 738 },
                { "className": "com.example.application.MyClass", "methodName": "exampleMethod", "fileName": "MyClass.java", "lineNumber": 81 },
                { "className": "com.example.application.MainClass", "methodName": "main", "fileName": "MainClass.java", "lineNumber": 52 }
        ],
        "cause": {
            "type": "java.io.IOException",
            "message": "Connection lost.",
            "stackTrace": [
                { "className": "java.nio.channels.SocketChannel", "methodName": "write", "fileName": null, "lineNumber": -1 },
                { "className": "org.example.sql.driver.PreparedStatement$1", "methodName": "responder", "fileName": "PreparedStatement.java", "lineNumber": 1032 },
                { "className": "org.example.sql.driver.PreparedStatement", "methodName": "executeUpdate", "fileName": "PreparedStatement.java", "lineNumber": 738 },
                { "className": "com.example.application.MyClass", "methodName": "exampleMethod", "fileName": "MyClass.java", "lineNumber": 81 },
                { "className": "com.example.application.MainClass", "methodName": "main", "fileName": "MainClass.java", "lineNumber": 52 }
            ]
        }
    },
    "contextMap": {
        "ID": "86c3a497-4e67-4eed-9d6a-2e5797324d7b",
        "username": "JohnDoe"
    },
    "contextStack": [
        "topItem",
        "anotherItem",
        "bottomItem"
    ]
}
```