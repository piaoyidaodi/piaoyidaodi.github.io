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
- `locationInfo，boolean`：如果为true，则文件名和行号将包含在HTML输出中。默认值是false。生成位置信息是一项大开销的操作，可能会影响性能。谨慎使用。
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
- `locationInfo，boolean`：如果为true，则appender会在生成的JSON中包含位置信息。默认为false。生成位置信息是一项大开销的操作，可能会影响性能。谨慎使用。
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

配置pattern string可构成灵活的layout，此类的目标是格式化LogEvent并返回结果。结果的格式取决于conversion pattern。

其conversion pattern与C语言中printf函数的转换模式密切相关。转换模式由称为转换说明符的文本文本和格式控制表达式组成。

请注意，任何文字文本（包括特殊字符）都可能包含在转换模式中。特殊字符包括\t，\n，\r，\f。使用\ \在输出中插入一个反斜杠。

每个转换说明符以百分号（%）开头，后跟可选的format modifier和conversion character。转换字符指定数据的类型，例如类别，优先级，日期，线程名称。格式修饰符控制诸如字段宽度，填充，左右对齐之类的东西。以下是一个简单的例子。

让转换模式为`%-5p [%t]: %m%n`，并假定Log4j环境设置为使用PatternLayout。然后声明如下：

```java
Logger logger = LogManager.getLogger("MyLogger");
logger.debug("Message 1");
logger.warn("Message 2");
```

这将输出：

```
DEBUG [main]: Message 1
WARN  [main]: Message 2
```

请注意，文本和转换说明符之间没有明确的分隔符。pattern解析器在读取转换字符时知道它何时到达转换说明符的末尾。在上面的例子中，转换说明符%-5p意味着记录事件的优先级应该左对齐为五个字符的宽度。

如果模式字符串不包含用于处理正在记录的Throwable的说明符，则对该模式的解析将如同将`%xEx`说明符添加到字符串末尾一样。要完全禁止Throwable的格式化，只需在模式字符串中添加“%ex{0}”作为说明符。

PatternLayout的参数如下所示：

- `charset，String`：将syslog字符串转换为字节数组时使用的字符集。该字符串必须是有效的字符集。如果未指定，则此layout使用平台默认字符集。
- `pattern，String`：下表中的一个或多个转换模式的复合模式字符串。不能用PatternSelector指定。
- `patternSelector，PatternSelector`：用于分析LogEvent中的信息并确定应使用哪种模式来格式化事件。pattern和patternSelector参数是互斥的。
- `replace，RegexReplacement`：允许替换结果字符串的部分。如果配置，替换元素必须指定正则表达式匹配和替换。这将执行类似于RegexReplacement转换器的功能，但适用于整个消息，而转换器仅适用于通过模式生成的字符串。
- `alwaysWriteExceptions，boolean`：如果为true（默认情况下），则即使该模式不包含异常转换，也会始终写入异常。这意味着如果您不在模式中包含输出异常的方式，则将默认异常格式化程序添加到模式的末尾。设置为false会禁用此行为，并允许您从模式输出中排除异常。
- `header，String`：要包含在每个日志文件顶部的可选标题字符串。
- `footer，String`：要包含在每个日志文件底部的可选标题字符串。
- `disableAnsi，boolean`：如果为true（默认值为false），则不输出ANSI转义码。
- `noConsoleNoAnsi，boolean`：如果为true（默认值为false）且System.console()为null，则不输出ANSI转义码。

RegexReplacement参数如下所示：

- `regex，String`：进行结果匹配的java兼容的正则表达式。
- `replacement，String`：替换任何匹配子串的字符串。

#### Patterns

Log4j中所提供的转换pattern如下所示：

**1. `c{precision}`或`logger{precision}`**

用来输出发布logging event的logger的名称。记录器转换说明符可以选择后跟precision specifier（精度说明符），精度说明符由十进制整数或以十进制整数开头的pattern组成。

当精度说明符是一个整数值时，它会减小记录器名称的大小。如果数字是正数，则布局将打印相应数量的最右边的记录器名称组件。如果为负数，布局将删除相应数量的最左边的记录器名称组件。

如果精度包含任何非整数字符，则layout会根据pattern缩写名称。如果精度整数小于1，则布局仍会完整地打印最右侧的标记。默认情况下，布局完全打印记录器名称。

示例如下：

- %c{1}，org.apache.common.Foo，Foo
- %c{2}，org.apache.common.Foo，commons.Foo
- %c{10}，org.apache.common.Foo，org.apache.common.Foo
- %c{-1}，org.apache.common.Foo，apache.common.Foo
- %c{-2}，org.apache.common.Foo，common.Foo
- %c{-10}，org.apache.common.Foo，org.apache.common.Foo
- %c{1.}，org.apache.common.Foo，o.a.c.Foo
- %c{1.1.~.~}，org.apache.common.test.Foo，o.a.~.~.Foo
- %c{.}，org.apache.common.test.Foo，....Foo

**2. `C{precision}`或`class{precision}`**

输出发出日志记录请求的调用者的全限定类名称。该转换说明符可以选择跟随精度说明符，该说明符遵循与记录器名称转换器相同的规则。

生成调用方的类名（位置信息）是一项大开销的操作，可能会影响性能。谨慎使用。

**3. `d{pattern}`或`date{pattern}`**

输出记录事件的日期。日期转换说明符后面可以跟着一组大括号，其中包含每个SimpleDateFormat的日期和时间模式字符串。

预定义的格式有DEFAULT，ABSOLUTE，COMPACT，DATE，ISO8601和ISO8601_BASIC。

您还可以使用一组，包含每个java.util.TimeZone.getTimeZone的时区id的大括号。如果没有给出日期格式说明符，则使用DEFAULT格式。

示例如下：

- `%d{DEFAULT}`，2012-11-02 14:34:02,781
- `%d{ISO8601}`，2012-11-02T14:34:02,781
- `%d{ISO8601_BASIC}`，20121102T143402,781
- `%d{ABSOLUTE}`，14:34:02,781
- `%d{DATE}`，02 Nov 2012 14:34:02,781
- `%d{COMPACT}`，20121102143402781
- `%d{HH:mm:ss,SSS}`，14:34:02,781
- `%d{dd MMM yyyy HH:mm:ss,SSS}`，02 Nov 2012 14:34:02,781
- `%d{HH:mm:ss}{GMT+0}`，18:34:02
- `%d{UNIX}`，1351866842
- `%d{UNIX_MILLIS}`，1351866842781

`%d{UNIX}`以秒为单位输出UNIX时间。`%d{UNIX_MILLIS}`以毫秒为单位输出UNIX时间。UNIX时间是以1970年1月1日的UTC午夜时间为基准计算当前时间和它之间以毫秒为单位的时间。虽然时间单位是毫秒，但粒度取决于操作系统（Windows）。因为只有从long到String的转换发生并且不涉及日期格式，所以这种输出事件时间的方式很有效。

**4. `enc{pattren}{[HTML|XML|JSON|CRLF]}`或`encode{pattren}{[HTML|XML|JSON|CRLF]}`**

以特定标记语言输出合适的编码和转义字符。默认情况下，如果只指定了一个选项，则这将对HTML进行编码。第二个选项用于指定应使用哪种编码格式。此转换器对编码用户提供的数据非常有用，以便输出数据不会被错误地或不安全地写入。

一种典型的编码消息用法是`%enc{ %m }`，但用户输入也可能来自其他位置，例如MDC`%enc{ %mdc{key}}`。

使用HTML编码格式，下列字符将被替换：

- `'\r', '\n'`：将被分别转换为`'\\r', '\\n'`。
- `&, <, >, ", ', /`：将被转换为对应的HTML实体。

使用XML编码格式，下列字符将被替换：

- `&, <, >, ", '`：将被转换为对应的XML实体。

使用JSON编码格式，下列字符将依据RFC4627条款进行替换：

- `U+0000 - U+001F`：`\u0000 - \u001F`
- 其他控制字符：编码为对应的`\uABCD`格式。
- `"`：`\"`
- `\`：`\\`

例如，{"message": "%enc{ %m}{JSON}"}可用于输出包含将日志消息作为String值的有效JSON文档。

使用CRLF编码格式，替换下列字符：

- `'\r', '\n'`：将被分别转换为`'\\r', '\\n'`。

**5. `equals{pattern}{test}{substitution}`或`equalsIgnoreCase{test}{substitution}`**

将使用{substitution}中的字符串，替换{test}中的字符串，从而对字符串进行评估。例如，`%equals{[%marker]}{[]}{}`将替换由无Marker的日志事件所产生的`[]`为空字符串。

该模式可以是任意复杂的，特别是可以包含多个转换关键字。

**6. `ex|exception|throwable{["none"|"full"|depth|"short"|short.className"|"short.fileName"|"short.lineNumber"|"short.methodName"|"short.message"|"short.localizedMessage"]}[,filters(package,package,...)][,separator(separator)]}{suffix(pattern)}`**

输出绑定到日志记录事件的Throwable trace，默认情况下，这将输出完整的trace，通常通过调用Throwable.printStackTrace()实现。

可以以`%throwable{option}`形式的选项来使用下列可抛出的转换词。

- `%throwable{short}`：输出Throwable的第一行。
- `%throwable{short.className}`：输出发生异常的类的名称。
- `%throwable{short.methodName}`：输出发生异常的方法名称。
- `%throwable{short.fileName}`：输出发生异常的类的名称。
- `%throwable{short.lineNumber}`：输出发生异常的行号。
- `%throwable{short.message}`：输出消息。
- `%throwable{short.localizedMessage}`：输出本地化的消息。
- `%throwable{n}`：输出堆栈跟踪的前n行。
- 指定`%throwable{none}`或`%throwable{0}`会禁止输出异常。
- 使用`filter(packages)`，其中package是软件包名称列表，用于从堆栈跟踪中压缩匹配的堆栈帧。
- 使用`separator`字符串来分隔堆栈跟踪的行。例如：`separator(|)`。缺省值是`line.separator`系统属性，该属性取决于操作系统。
- 仅当存在throwable字符时才使用`ex{suffix(pattern)}`将pattern的输出添加到输出中。

**7. `F`或`file`**

输出发出记录请求的文件名。生成文件信息（位置信息）是一项高开销的操作，可能会影响性能。谨慎使用。

**8. `highlight{pattern}{sytle}`**

根据当前事件日志记录级别，将ANSI颜色添加到enclosed pattern的结果中。（请参阅Jansi配置。）
每个级别的默认颜色是：

- FATAL：Bright red
- ERROR：Bright red
- WARN：Yellow
- INFO：Green
- DEBUG：Cyan
- TRACE：Black

使用默认颜色：

`%highlight{ %d [%t] %-5level: %msg%n%throwable}`。

通过{style}选项覆盖默认颜色设置：

`%highlight{ %d [%t] %-5level: %msg%n%throwable}{FATAL=white, ERROR=red, WARN=blue, INFO=black, DEBUG=green, TRACE=blue}`。

可以只高亮日志事件的一部分：

`%d [%t] %highlight{ %-5level: %msg%n%throwable}`

可以风格化消息的一部分并高亮剩下的部分：

`%style{ %d [%t]}{black} %highlight{ %-5level: %msg%n%throwable}`

也可以使用STYLE键：

`%highlight{ %d [%t] %-5level: %msg%n%throwable}{STYLE=Logback}`

**9. `K{key}`或`map{key}`或`MAP{key}`**

输出MapMessage中的条目（如果事件中存在MapMessage）。K转换字符之后的大括号之间放置映射关键字，如`%K{clientNumber}`，其中clientNumber是关键字。Map中对应于该键的值将输出。如果没有指定附加的子选项，那么Map键值对集合的全部内容使用格式`{ {key1,val1},{key2,val2} }`输出。

**10. `l`或`location`**

输出产生记录事件的调用者的位置信息。位置信息取决于JVM的实现，但通常由调用方法的完全限定名称组成，后跟调用者输入文件名和行号（放在括号内）。生成位置信息是一项高开销的操作，可能会影响性能。谨慎使用。

**11. `L`或`line`**

输出发出日志记录请求的行号。生成行号信息（位置信息）是一项高负载的操作，可能会影响性能。谨慎使用。

**12. `m{nolookups}{ansi}`或`msg{nolookups}{ansi}`或`message{nolookups}{ansi}`**

输出应用程序提供的与记录事件相关的消息。添加`{ansi}`以使用ANSI转义代码呈现消息（需要JAnsi）。

嵌入式ANSI代码的默认语法是：`@|code(,code)* text|@`

例如，要以绿色呈现消息Hello，请使用：`@|green Hello|@`

要将消息Hello以粗体和红色显示，请使用：`@|bold,red Warning!|@`

您还可以使用以下语法在配置中自定义样式名称：`%message{ansi}{StyleName=value(,value)*( StyleName=value(,value)*)*}%n`

例如：`%message{ansi}{WarningStyle=red,bold KeyStyle=white ValueStyle=blue}%n`

调用可能如下所示：`logger.info("@|KeyStyle {}|@ = @|ValueStyle {}|@", entry.getKey(), entry.getValue());`

使用`{nolookups}`来记录如`${date:YYYY-MM-dd}`消息，而不使用任何lookup。通常情况下，调用`logger.info("Try ${date:YYYY-MM-dd}")`将用实际日期替换日期模板`${date:YYYY-MM-dd}`。使用nolookups将禁用此功能并原样记录消息字符串。

**13. `M`或`method`**

输出发出日志记录请求的方法名称。生成调用方的方法名称（位置信息）是一项高开销操作，可能会影响性能。谨慎使用。

**14. `marker`**

如果有marker，则输出其的全名，包含其父类。

**15. `markerSimpleName`**

如果有marker，则输出其的简称，不包含其父类。

**16. `maxLen`或`maxLength`**

输出经过评估的pattern并截断结果的结果。如果长度大于20，则输出将包含尾随省略号。如果提供的长度无效，则使用默认值100。

语法示例：`%maxLen{ %p: %c{1} - %m%notEmpty{ =>%ex{short}}}{160}`将被限制为160个字符，并带有尾部省略号。另一个示例：`%maxLen{ %m}{20}`将限制为20个字符，并且不会有结尾省略号。

**17. `n`**

输出平台相关的行分隔符字符。此转换字符与使用non-portable行分隔符字符串（如`\n`或`\r\n`）的性能几乎相同。因此，它是指定行分隔符的首选方式。

**18. `N`或`nano`**

在日志记录事件被创建时输出`System.nanoTime()`的执行结果。

**19. `pid{[defaultValue]}`或`processId{[defaultValue]}`**

如果基础平台支持，则输出进程ID。如果平台不支持进程ID，则可以指定一个可选的缺省值。

**20. `variablesNotEmpty{pattern}`或`varsNotEmpty{pattern}`或`notEmpty{pattern}`**

当且仅当pattern中的所有变量都不为空时，输出pattern的评估结果。比如：`%notEmpty{[%marker]}`。

**21. `p|level{level=label, level=label, ...}`或`p|level{length=n}`或`p|level{lowerCase=true|false}`**

输出记录事件的级别。使用`level = value，level = value`的形式提供级别名称映射，其中level是级别的名称，value是应显示的值而不是Level的名称。

例如：`%level{WARN=Warning, DEBUG=Debug, ERROR=Error, TRACE=Trace, INFO=Info}`，`%level{WARN=W, DEBUG=D, ERROR=E, TRACE=T, INFO=I}`。

更简洁地说，对于与上面相同的结果，您可以定义级别标签的长度：`%level{length=1}`，如果length大于级别名称的长度，则layout使用普通级别名称。

你可以结合这两种选择：`%level{ERROR=Error, length=2}`，这里给定了Error级别名称和名称长度为2的所有其他级别名称。

最后，您可以输出小写的级别名称（默认为大写）：`%level{lowerCase=true}`。

**22. `r`或`relative`**

输出自JVM启动直到创建记录事件以来经过的毫秒数。

**23. `replace{pattern}{regex}{substitution}`**

在pattern的计算结果中，使用在{substitution}中定义的字符串替换{regex}的正则。例如，`%replace{ %msg}{\s}{}`将删除事件消息中包含的所有空格。

该pattern可以是任意复杂的，特别是可以包含多个转换关键字。例如，`%replace { %logger %msg}{\.}{/}`将用正斜杠替换记录器中的所有`.`或事件的消息。

**24. `rEx|rException|rThrowable{["none"|"short"|"full"|depth][,filters(package,package,...)][,separator(separator)]}{ansi(Key=Value,Value,... Key=Value,Value,... ...)}{suffix(pattern)}`**

与`%throwable`转换词相同，但是打印堆栈跟踪时会从第一个抛出的异常开始，然后是每个后续的包装异常。

可抛出的转换字后面跟着一个格式为`%rEx{short}`的选项，该选项将只输出Throwable的第一行或`%rEx{n}`，其中堆栈跟踪的前n行将被打印。

指定`%rEx{none}`或`%rEx{0}`将会禁止打印异常。

使用`filters(packages)`，其中packages是软件包名称列表，用于从堆栈跟踪中压缩匹配的堆栈帧。

使用separator字符串来分隔堆栈跟踪的行。例如：separator(|)。缺省值是line.separator系统属性，该属性取决于操作系统。

仅当存在throwable的打印时，才使用`rEx{suffix(pattern)}`将pattern的输出添加到输出。

**25. `sn`或`sequenceNumber`**

包括在每个事件中都会增加的序列号。该计数器是一个静态变量，因此在共享相同转换器Class对象的应用程序中只会是唯一的。

**26. `style{pattern}{ANSI style}`**

使用ANSI转义序列对封闭模式的结果进行样式设置。 样式可以由下表中由逗号分隔的样式名称列表组成。详表见官方文档。示例如下所示：

`%style{ %d{ISO8601}}{black} %style{[%t]}{blue} %style{ %-5level:}{yellow} %style{ %msg%n%throwable}{green}`。

合并样式：`%d %highlight{ %p} %style{ %logger}{bright,cyan} %C{1.} %msg%n`。

可以使用`%`后跟颜色`%black, %blue, %cyan`，比如：

`%black{ %d{ISO8601}} %blue{[%t]} %yellow{ %-5level:} %green{ %msg%n%throwable}`

**27. `T`或`tid`或`threadId`**：输出产生日志事件的线程ID。

**28. `t`或`tn`或`thread`或`threadName`**：输出产生日志事件的线程名称。

**29. `tp`或`threadPriority`**：输出产生日志事件的线程优先级。

**30. `x`或`NDC`**：输出产生记录事件的线程相关联的线程上下文栈（也称为嵌套诊断上下文或NDC）。

**31. `X{key[,key2...]}`或`mdc{key[,key2...]}`或`MDC{key[,key2...]}`**

输出产生记录事件的线程相关联的线程上下文栈（也称为嵌套诊断上下文或NDC）。在X转换字符后面可以使用大括号中放置一个或多个键的映射表，如`%X{clientNumber}`，其中clientNumber是键。将输出对应于该键在MDC中的值。

如果提供了键列表，例如`%X{name, number}`，那么ThreadContext中的每个键都将使用格式`{name=val1, number=val2}`输出。键/值对将按照它们在列表中出现的顺序进行打印。

如果未指定子选项，则使用格式`{key1=val1, key2=val2}`输出MDC键值对集的全部内容。键/值对将按排序顺序打印。

有关更多详细信息，请参阅ThreadContext类。

**32. `u{"RANDOM"|"TIME"}`或`uuid`**

包含随机或基于时间的UUID。基于时间的UUID是一种Type1 UUID，每毫秒最多可生成10,000个唯一标识（将使用每个主机的MAC地址），并尝试确保同一主机上的多个JVM和/或ClassLoaders具有在0和16,384之间的唯一随机数将与UUID生成器Class的每个实例相关联并且包含在每个生成的基于时间的UUID中。由于基于时间的UUID包含MAC地址和时间戳，因此它们应该小心使用，因为它们可能会导致安全漏洞。

**33. `xEx|xException|xThrowable`**

**34. `%`**：`%%`序列将输出单个百分号。

默认情况下，相关信息按原样输出。但是，借助格式修改器，可以更改最小字段宽度，最大字段宽度和对齐方式。

可选的格式修饰符位于百分号和转换字符之间。

第一个可选的格式修饰符是左对齐标志，它只是减号`(-)`字符。然后是可选的最小字段宽度，这是一个十进制常量，表示要输出的最少字符数。如果数据项需要较少的字符，则将其填充到左侧或右侧，直到达到最小宽度。默认是填充左边（右对齐），但您可以使用左对齐标志指定正确的填充，填充字符是空格。如果数据项大于最小字段宽度，则字段将扩展以适应数据，该值永远不会被截断。

还可以使用最大字段宽度修饰符进行更改，该修饰符由一个句点和一个小数常数指定。如果数据项长于最大字段，则额外的字符将从数据项的开头移除而不是从最后移除。例如，最大字段宽度为8，数据项为10个字符长，则数据项的前两个字符将被丢弃。这种行为偏离了C语言中的printf函数从最后开始截断的特点。

可以通过在该时间段之后追加减号来从结尾截断。在这种情况下，如果最大字段宽度为8，数据项的长度为10个字符，则数据项的最后两个字符将被丢弃。

#### ANSI Styling on Windows

ANSI转义序列在许多平台上本机支持，但在Windows上不默认。要启用ANSI支持，请将Jansi jar添加到您的应用程序，并将属性log4j.skipJansi设置为false。 这允许Log4j在写入控制台时使用Jansi添加ANSI转义码。

注：在Log4j 2.10之前，Jansi默认启用。 Jansi需要本地代码的事实意味着Jansi只能由一个类加载器加载。对于Web应用程序来说，这意味着Jansi jar必须位于Web容器的类路径中。为避免导致Web应用程序出现问题，从Log4j 2.10开始如果显式配置，则Log4j将不再自动尝试加载Jansi。

#### Example Patterns

**Filtered Throwables**

此示例显示如何在堆栈跟踪中过滤掉的非重要包中的类。

```xml
<properties>
  <property name="filters">org.junit,org.apache.maven,sun.reflect,java.lang.reflect</property>
</properties>
...
<PatternLayout pattern="%m%xEx{filters(${filters})}%n"/>
```

**ANSI Styled**

日志级别将根据事件的日志级别高亮显示。所有跟随该级别的内容都将呈现亮绿色。

```xml
<PatternLayout>
  <pattern>%d %highlight{ %p} %style{ %C{1.} [%t] %m}{bold,green}%n</pattern>
</PatternLayout>
```

#### Pattern Selectors

可以使用PatternSelector配置PatternLayout，以允许它根据日志事件的属性或其他因素选择要使用的模式。PatternSelector通常会配置defaultPattern属性，在其他条件不匹配时使用；以及一组PatternMatch元素，用于标识可选的各种模式。

**MarkerPatternSelector**

MarkerPatternSelector根据日志事件中包含的Marker选择模式。如果日志事件中的Marker等于或是PatternMatch键属性上指定的名称的祖先，则将使用该PatternMatch元素上指定的模式。

```xml
<PatternLayout>
  <MarkerPatternSelector defaultPattern="[%-5level] %c{1.} %msg%n">
    <PatternMatch key="FLOW" pattern="[%-5level] %c{1.} ====== %C{1.}.%M:%L %msg ======%n"/>
  </MarkerPatternSelector>
</PatternLayout>
```

**ScriptPatternSelector**

ScriptPatternSelector执行Configuration章节Scripts部分描述的脚本。该脚本将传递配置中的Properties部分中所配置的所有属性，如在substitutor变量中由Configuration使用的StrSubstitutor；在logEvent变量中的日志事件；预期会返回PatternMatch的值；或者如果应该使用默认模式，则为null。

```xml
<PatternLayout>
  <ScriptPatternSelector defaultPattern="[%-5level] %c{1.} %C{1.}.%M.%L %msg%n">
    <Script name="BeanShellSelector" language="bsh"><![CDATA[
      if (logEvent.getLoggerName().equals("NoLocation")) {
        return "NoLocation";
      } else if (logEvent.getMarker() != null && logEvent.getMarker().isInstanceOf("FLOW")) {
        return "Flow";
      } else {
        return null;
      }]]>
    </Script>
    <PatternMatch key="NoLocation" pattern="[%-5level] %c{1.} %msg%n"/>
    <PatternMatch key="Flow" pattern="[%-5level] %c{1.} ====== %C{1.}.%M:%L %msg ======%n"/>
  </ScriptPatternSelector>
</PatternLayout>
```

### 8.6 RFC5424 Layout

顾名思义，Rfc5424Layout根据RFC5424（增强型Syslog规范）格式化LogEvents。尽管规范主要针对通过Syslog发送消息，但此格式对于其他目的非常有用，因为项目作为self-describing键值对在消息中传递。

RFC5452Layout的参数如下所示：

- `appName，String`：在RFC5424 syslog记录中用作APPNAME的值。
- `charset，String`：将syslog字符串转换为字节数组时使用的字符集。该字符串必须是有效的字符集。如果未指定，则将使用系统默认字符集。
- `enterpriseNumber，integer`：RFC5424中所述的IANA企业编号。
- `exceptionPattern，String`：PatternLayout中的一个转换说明符，它定义了用于格式化异常的ThrowablePatternConverter。可以包含对这些说明符有效的任何选项。缺省情况下，在输出中不包括来自事件的Throwable（如果有）。
- `facility，String`：facility用于尝试对消息进行分类。facility选项必须设置为“KERN”，“USER”，“MAIL”，“DAEMON”，“AUTH”，“SYSLOG”，“LPR”，“NEWS”，“UUCP”，“CRON”，“AUTHPRIV”，“FTP”，“NTP”，“AUDIT”，“ALERT”，“CLOCK”，“LOCAL0”，“LOCAL1”，“LOCAL2”，“LOCAL3”，“LOCAL4”，“LOCAL5”，“LOCAL6”，或“LOCAL7”。这些值可能为大写或小写字符。
- `format，String`：如果设置为“RFC5424”，数据将根据RFC5424格式化。否则，它将被格式化为BSD Syslog记录。请注意，虽然BSD Syslog记录要求为1024字节或更短，但SyslogLayout不会截断它们。RFC5424Layout也不会截断记录，因为接收器必须接受长达2048字节的记录，并可能接受更长的记录。
- `id，String`：当进行格式化时根据RFC5424所使用的默认结构化数据ID。如果LogEvent包含StructuredDataMessage，则将使用来自消息的ID代替此值。
- `includeMDC，boolean`：指示来自ThreadContextMap的数据是否将包含在RFC5424 Syslog记录中。默认为true。
- `loggerFields，List of KeyValuePairs`：允许将任意PatternLayout模式作为指定的ThreadContext字段包含在内；未指定默认值。使用时请包含LoggerFields嵌套元素，其中包含一个或多个KeyValuePair元素。每个KeyValuePair 必须具有一个key属性，该属性指定将用于标识MDC Structured Data元素内的字段的键名；以及一个value属性，其指定了要用作值的PatternLayout模式。
- `mdcExcludes，String`：应该从LogEvent中排除的的mdc键列表（使用逗号分隔）。这与mdcIncludes属性是互斥的。该属性仅适用于RFC5424 syslog记录。
- `mdcIncludes，String`：应该包含在FlumeEvent中的mdc键列表（使用逗号分隔）。列表中未找到的任何MDC密钥都将被排除。该选项与mdcExcludes属性互斥。该属性仅适用于RFC5424 syslog记录。
- `mdcRequired，String`：必须存在于MDC中的mdc键列表（使用逗号分隔）。如果某个键不存在，则会抛出LoggingException。该属性仅适用于RFC5424 syslog记录。
- `mdcPrefix，String`：应该为每个MDC密钥添加前缀以便将其与event属性区分开。默认字符串是“mdc：”。该属性仅适用于RFC5424 syslog记录。
- `mdcId，String`：必需的MDC ID。该属性仅适用于RFC 5424 syslog记录。
- `messageId，String`：在RFC5424 syslog记录的MSGID字段中使用的默认值。
- `newLine，boolean`：如果为true，则将在syslog记录的末尾附加一个换行符。默认值是false。
- `newLineEscape`，String：应该用来替换消息文本中的换行符的字符串。

### 8.7 Serialized Layout

SerializedLayout只需使用Java Serialization将LogEvent序列化为一个字节数组。SerializedLayout不接受任何参数。

从版本2.9开始，此layout已被弃用。Java Serialization具有固有的安全弱点，不再推荐使用这种layout。包含相同信息的替代layout是JsonLayout，配置为`properties="true"`。

### 8.8 Syslog Layout

SyslogLayout将LogEvent格式化为与Log4j1.2使用的相同格式匹配的BSD Syslog记录。

Syslog参数如下所示：

- `charset，String`：将syslog字符串转换为字节数组时使用的字符集。该字符串必须是有效的字符集。如果未指定，则将使用系统默认字符集。
- `facility，String`：facility用于尝试对消息进行分类。facility选项必须设置为“KERN”，“USER”，“MAIL”，“DAEMON”，“AUTH”，“SYSLOG”，“LPR”，“NEWS”，“UUCP”，“CRON”，“AUTHPRIV”，“FTP”，“NTP”，“AUDIT”，“ALERT”，“CLOCK”，“LOCAL0”，“LOCAL1”，“LOCAL2”，“LOCAL3”，“LOCAL4”，“LOCAL5”，“LOCAL6”，或“LOCAL7”。这些值可能为大写或小写字符。
- `newLine，boolean`：如果为true，则将在syslog记录的末尾附加一个换行符。默认值是false。
- `newLineEscape`，String：应该用来替换消息文本中的换行符的字符串。

### 8.9 XML Layout

append一系列在log4j.dtd中定义的Event元素。

#### Complete well-formed XML vs fragment XML

如果您配置`complete="true"`，则appender会输出well-formed XML文档，其默认名称空间是Log4j名称空间`"http://logging.apache.org/log4j/2.0/events"`。默认情况下，使用`complete="false"`时，应该将输出作为external entity包含在单独的文件中以形成well-formed XML文档，在这种情况下，appender使用默认为log4j的namespacePrefix。

一个well-formed XML文档遵循如下：

```xml
<Event xmlns="http://logging.apache.org/log4j/2.0/events"
       timeMillis="1493122559666"
       level="INFO"
       loggerName="HelloWorld"
       endOfBatch="false"
       thread="main"
       loggerFqcn="org.apache.logging.log4j.spi.AbstractLogger"
       threadId="1"
       threadPriority="5">
  <Marker name="child">
    <Parents>
      <Marker name="parent">
        <Parents>
          <Marker name="grandparent"/>
        </Parents>
      </Marker>
    </Parents>
  </Marker>
  <Message>Hello, world!</Message>
  <ContextMap>
    <item key="bar" value="BAR"/>
    <item key="foo" value="FOO"/>
  </ContextMap>
  <ContextStack>
    <ContextStackItem>one</ContextStackItem>
    <ContextStackItem>two</ContextStackItem>
  </ContextStack>
  <Source
      class="logtest.Main"
      method="main"
      file="Main.java"
      line="29"/>
  <Thrown commonElementCount="0" message="error message" name="java.lang.RuntimeException">
    <ExtendedStackTrace>
      <ExtendedStackTraceItem
          class="logtest.Main"
          method="main"
          file="Main.java"
          line="29"
          exact="true"
          location="classes/"
          version="?"/>
    </ExtendedStackTrace>
  </Thrown>
</Event>
```

如果`complete="false"`，则appender不会写入XML处理指令和根元素。

#### Marker

Marker由Event元素中的Marker元素表示。Marker元素仅在日志消息中使用marker时出现。marker的父项的名称将在Marker元素的parent属性中提供。

#### Pretty vs compact XML

默认情况下，设置compact="false"使XML布局不紧凑（a.k.a.不"pretty"），这意味着appender使用行尾字符和缩进行来格式化XML。如果compact="true"，则不使用行尾或缩进。当然Message内容可能包含行尾。

JsonLayout参数如下所示：

- `charset，String`：转换为字节数组时使用的字符集。该值必须是有效的字符集。如果未指定，将使用UTF-8。
- `compact，boolean`：如果为true，则appender不使用尾行和缩进。默认为false。
- `complete，boolean`：如果为true，则Appender包含XML的页眉和页脚，以及记录之间的逗号。默认为false。
- `properties，boolean`：如果为true，则appender将在生成的XML中包含线程上下文映射。默认为false。
- `locationInfo，boolean`：如果为true，则appender会在生成的XML中包含位置信息。默认为false。生成位置信息是一项大开销的操作，可能会影响性能。谨慎使用。
- `includeStacktrace，boolean`：如果为true，则包含任何记录的Throwable的完整堆栈跟踪（可选，默认为true）。
- `stacktraceAsString，boolean`：是否将stacktrace格式化为字符串而不是嵌套对象（可选，默认为false）。
- `includeNullDelimiter，boolean`：是否在每个事件之后包含NULL字节作为分隔符（可选，默认为false）。

要在输出中包含任何自定义字段，请使用以下语法：

```xml
<XmlLayout>
  <KeyValuePair key="additionalField1" value="constant value"/>
  <KeyValuePair key="additionalField2" value="$${ctx:key}"/>
</XmlLayout>
```

自定义字段总是最后一个，按其声明的顺序排列。

### 8.10 YAML Layout

将一系列YAML事件append为字节序列化的字符串。

YAML日志事件遵循以下模式：

```yaml
---
timeMillis: 1493122307075
thread: "main"
level: "INFO"
loggerName: "HelloWorld"
marker:
 name: "child"
 parents:
 - name: "parent"
   parents:
   - name: "grandparent"
message: "Hello, world!"
thrown:
 commonElementCount: 0
 message: "error message"
 name: "java.lang.RuntimeException"
 extendedStackTrace:
 - class: "logtest.Main"
   method: "main"
   file: "Main.java"
   line: 29
   exact: true
   location: "classes/"
   version: "?"
contextStack:
- "one"
- "two"
endOfBatch: false
loggerFqcn: "org.apache.logging.log4j.spi.AbstractLogger"
contextMap:
 bar: "BAR"
 foo: "FOO"
threadId: 1
threadPriority: 5
source:
 class: "logtest.Main"
 method: "main"
 file: "Main.java"
 line: 29
```

YamlLayout参数如下所示：

- `charset，String`：转换为字节数组时使用的字符集。该值必须是有效的字符集。如果未指定，将使用UTF-8。
- `properties，boolean`：如果为true，则appender将在生成的YAML中包含线程上下文映射。默认为false。
- `locationInfo，boolean`：如果为true，则appender会在生成的YAML中包含位置信息。默认为false。生成位置信息是一项大开销的操作，可能会影响性能。谨慎使用。
- `includeStacktrace，boolean`：如果为true，则包含任何记录的Throwable的完整堆栈跟踪（可选，默认为true）。
- `stacktraceAsString，boolean`：是否将stacktrace格式化为字符串而不是嵌套对象（可选，默认为false）。
- `includeNullDelimiter，boolean`：是否在每个事件之后包含NULL字节作为分隔符（可选，默认为false）。

要在输出中包含任何自定义字段，请使用以下语法：

```xml
<YamlLayout>
  <KeyValuePair key="additionalField1" value="constant value"/>
  <KeyValuePair key="additionalField2" value="$${ctx:key}"/>
</YamlLayout>
```

自定义字段总是最后一个，按其声明的顺序排列。

### 8.11 Location Information

如果其中一个layout配置了位置相关属性（如HTML locationInfo），或者其中一个模式`%C`或`%class`，`%F`或`%file`，`%l`或`%location`，`%L`或`%line`，`%M`或`%method`，Log4j将对堆栈快照，然后进行栈跟踪以查找位置信息。

这是一项高开销的操作：对同步的logger来说，速度要慢1.3至5倍。同步logger在进行堆栈快照前尽可能长时间地等待。如果不需要位置，则不会拍摄快照。

但是，异步logger需要在将日志消息传递给另一个线程之前作出此决定；该点后位置信息将会丢失。采用堆栈跟踪快照对异步logger的性能影响比同步的更高：使用位置记录比没有位置记录慢30-100倍。出于这个原因，异步记录器和异步appender默认不包含位置信息。

您可以通过指定`includeLocation = "true"`来覆盖logger或异步appender配置中的默认行为。
