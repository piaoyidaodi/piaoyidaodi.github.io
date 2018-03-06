---
layout: post
title: "Log4j2 -- Layouts"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 8. Layouts

Appender使用Layout将LogEvent格式化为满足使用的日志事件需求格式。在Log4j1.x和Logback中，Layouts将一个事件转换为一个String。在Log4j2中Layout返回一个字节数组。这允许Layout的结果在更多类型的Appender中有用。 但是，这意味着您需要使用Charset配置大多数Layout，以确保字节数组包含正确的值。

使用Charset的root layout类是org.apache.logging.log4j.core.layout.AbstractStringLayout，其中默认值是UTF-8。扩展AbstractStringLayout的每个layout都可以提供自己的默认值。

在Log4j2.4.1中增加了ISO-8859-1和US-ASCII字符集并添加了一个自定义字符编码器，以将Java8内置的一些性能改进带入Log4j，以便在Java7上使用。对于仅记录ISO-8859-1字符集日志的应用程序，指定此字符集将显着提高性能。

### 8.1 CSV Layouts

CSV layout会创建Comma Separated Value（CSV）记录，并且需要Apache Commons CSV 1.4。

可以通过两种方式使用CSV layout：首先，使用CsvParameterLayout记录事件参数以创建自定义数据库，通常是为了此目的而唯一配置Appender和file appender。其次，使用CsvLogEventLayout来记录事件以创建数据库，以作为使用支持CSV格式的DBMS或JDBC驱动程序的替代品。

CsvParameterLayout将事件的参数转换为CSV记录且忽略消息。要记录CSV记录，可以使用通常的Logger方法info()，debug()等：

```java
logger.info("Ignored",value1,value2,value3);
```

将创建CSV记录`value1,value2,value3`。

或者，您可以使用只携带参数的ObjectArrayMessage：

```java
logger.info(new ObjectArrayMessage(value1, value2, value3));
```

使用以下参数配置CsvParameterLayout和CsvLogEventLayout：

- `format，String`：预定义格式之一：Default，Excel，MySQL，RFC4180，TDF。
- `delimiter，Character`：设置格式分隔符为指定的字符。
- `escape，Character`：设置格式转义字符为指定的字符。
- `quote，Character`：将格式quoteChar设置为指定的字符。
- `quoteMode，String`：将格式的output quote policy设置为指定的值。下列其中之一：ALL，MINIMAL，NON_NUMERIC，NONE。
- `nullString，String`：在写入记录时，将null写为给定的nullString。
- `recordSeparator，String`：将格式记录分隔符设置为指定的String。
- `charset，Charset`：输出字符集。
- `header`：设置打开流时包含的header。
- `footer`：设置关闭流时包含的footer。

按照CSV事件Logging如下所示：

```java
logger.debug("one={}, two={}, three={}", 1, 2, 3);
```

使用以下字段产生一个CSV记录：

Time Nanos, Time Millis, Level, Thread ID, Thread Name, Thread Priority, Formatted Message, Logger FQCN, Logger Name, Marker
Thrown Proxy, Source, Context Map, Context Stack

```
0,1441617184044,DEBUG,main,"one=1, two=2, three=3",org.apache.logging.log4j.spi.AbstractLogger,,,,org.apache.logging.log4j.core.layout.CsvLogEventLayoutTest.testLayout(CsvLogEventLayoutTest.java:98),{},[]
```

### 8.2 GELF Layout

在Graylog Extended Log Format（GELF）1.1中layout事件。

如果日志事件数据大于1024字节（compressionThreshold），则此布局将JSON压缩为GZIP或ZLIB（compressionType）。该layout不实现分块。

如下配置将使用UDP发送到Graylog 2.x服务器：

```xml
 <Appenders>
    <Socket name="Graylog" protocol="udp" host="graylog.domain.com" port="12201">
        <GelfLayout host="someserver" compressionType="ZLIB" compressionThreshold="1024"/>
    </Socket>
</Appenders>
```

如下配置将使用TCP发送到Graylog 2.x服务器：

```xml
<Appenders>
    <Socket name="Graylog" protocol="tcp" host="graylog.domain.com" port="12201">
        <GelfLayout host="someserver" compressionType="OFF" includeNullDelimiter="true"/>
    </Socket>
</Appenders>
```

GelfLayout参数如下所示：

- `host，String`：host属性的值（可选，默认为本地主机名）。
- `compressionType，GZIP、ZLIB、OFF`：压缩（可选，默认为GZIP）。
- `compressionThreshold，int`：如果大于此字节数则压缩数据（可选，默认为1024）。
- `includeStacktrace，boolean`：是否包含记录的Throwables的完整堆栈跟踪（可选，默认为true）。如果设置为false，则只包含Throwable的类名和消息。
- `includeThreadContext，boolean`：是否包含线程上下文作为附加字段（可选，默认为true）。
- `includeNullDelimiter，boolean`：是否在每个事件之后包含NULL字节作为分隔符（可选，默认为false）。对Graylog GELF TCP输入有用。不能与压缩一起使用。

### 8.3 HTML Layout

HtmlLayout生成一个HTML页面并将每个LogEvent添加到表中的一行。HtmlLayout参数如下所示：

- `charset，String`：将HTML字符串转换为字节数组时使用的字符集。该值必须是有效的字符集。如果未指定，则此布局使用UTF-8。
- `contentType，String`：要分配给Content-Type header的值。默认值是“text/html”。
- `locationInfo，boolean`：如果为true，则文件名和行号将包含在HTML输出中。默认值是false。生成位置信息是一项昂贵的操作，可能会影响性能。谨慎使用。
- `title，String`：将显示为HTML标题的字符串。
- `fontName，String`：要使用的font-family。默认值是“arial，sans-serif”。
- `fontSize，String`：要使用的font-size。默认值是“小”。

### 8.4 JSON Layout

将一系列JSON事件附加为字节序列化的字符串。

#### Complete well-formed JSON vs fragment JSON

如果您配置complete="true"，appender将输出一个well-formed JSON文档。默认情况下，使用complete="false"时，应该将输出作为外部文件包含在单独的文件中以形成well-formed JSON文档。

如果complete="false"，appender不会在文档开头写入JSON开放数组字符`[`，不会在文档末尾写入`]`，也不会在记录之间使用逗号。

日志事件遵循以下模式：

```json
{
  "timeMillis" : 1493121664118,
  "thread" : "main",
  "level" : "INFO",
  "loggerName" : "HelloWorld",
  "marker" : {
    "name" : "child",
    "parents" : [ {
      "name" : "parent",
      "parents" : [ {
        "name" : "grandparent"
      } ]
    } ]
  },
  "message" : "Hello, world!",
  "thrown" : {
    "commonElementCount" : 0,
    "message" : "error message",
    "name" : "java.lang.RuntimeException",
    "extendedStackTrace" : [ {
      "class" : "logtest.Main",
      "method" : "main",
      "file" : "Main.java",
      "line" : 29,
      "exact" : true,
      "location" : "classes/",
      "version" : "?"
    } ]
  },
  "contextStack" : [ "one", "two" ],
  "endOfBatch" : false,
  "loggerFqcn" : "org.apache.logging.log4j.spi.AbstractLogger",
  "contextMap" : {
    "bar" : "BAR",
    "foo" : "FOO"
  },
  "threadId" : 1,
  "threadPriority" : 5,
  "source" : {
    "class" : "logtest.Main",
    "method" : "main",
    "file" : "Main.java",
    "line" : 29
  }
}
```

#### Pretty vs compact JSON

默认情况下，设置compact="false"使JSON布局不紧凑（a.k.a.不"pretty"），这意味着appender使用行尾字符和缩进行来格式化文本。如果compact="true"，则不使用行尾或缩进。当然Message内容可能包含escaped行尾。

JsonLayout参数如下所示：

- `charset，String`：转换为字节数组时使用的字符集。该值必须是有效的字符集。如果未指定，将使用UTF-8。
- `compact，boolean`：如果为true，则appender不使用尾行和缩进。默认为false。
- `eventEol，boolean`：如果为true，则appender会在每条记录后附加一行结尾。默认为false。使用eventEol = true和compact = true可以获得每行一条记录。
- `complete，boolean`：如果为true，则Appender包含JSON的页眉和页脚，以及记录之间的逗号。默认为false。
- `properties，boolean`：如果为true，则appender将在生成的JSON中包含线程上下文映射。默认为false。
- `propertiesAsList，boolean`：如果为true，则线程上下文映射将作为映射条目对象列表包含，其中每个条目都有key属性（其值为可以）和value属性（其值为value）。默认为false，在这种情况下，线程上下文映射被包含为键值对的简单映射。
- `locationInfo，boolean`：如果为true，则appender会在生成的JSON中包含位置信息。默认为false。生成位置信息是一项昂贵的操作，可能会影响性能。谨慎使用。
- `includeStacktrace，boolean`：如果为true，则包含任何记录的Throwable的完整堆栈跟踪（可选，默认为true）。
- `stacktraceAsString，boolean`：是否将stacktrace格式化为字符串而不是嵌套对象（可选，默认为false）。
- `includeNullDelimiter，boolean`：是否在每个事件之后包含NULL字节作为分隔符（可选，默认为false）。

要在输出中包含任何自定义字段，请使用以下语法：

```xml
<JsonLayout>
    <KeyValuePair key="additionalField1" value="constant value"/>
    <KeyValuePair key="additionalField2" value="$${ctx:key}"/>
</JsonLayout>
```

自定义字段总是最后一个，按其声明的顺序排列。

### 8.5 Pattern Layout

### 8.6 RFC5424 Layout

### 8.7 Serialized Layout

### 8.8 Syslog Layout

### 8.9 XML Layout

### 8.10 YAML Layout

### 8.11 Location Information
