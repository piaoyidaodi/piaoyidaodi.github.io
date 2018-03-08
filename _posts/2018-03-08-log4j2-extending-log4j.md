---
layout: post
title: "Log4j2 -- Extending Log4j"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 9. Extending Log4j

Log4j2提供了多种可以被操纵和扩展的方法。本节概述了Log4j2实现中所直接支持的各种方式。

### 9.1 LoggerContextFactory

LoggerContextFactory将Log4j API绑定到其实现。 Log4j的LogManager通过使用`java.util.ServiceLoader`来定位LoggerContextFactory，以找到`org.apache.logging.log4j.spi.Provider`的所有实例。每个实现必须提供一个扩展`org.apache.logging.log4j.spi.Provider`的类，并且该实现应该有一个无参数构造函数，通过将该构造函数委托给Provider的构造函数，传入与其兼容的Priority和API版本，以及实现`org.apache.logging.log4j.spi.LoggerContextFactory`的类。Log4j将比较当前的API版本，如果兼容，该实现将被添加到提供者列表中。`org.apache.logging.log4j.LogManager`中的API版本仅在有新功能添加，且实现时需要注意时才会更改。如果找到多个有效实现，则Priority的值将用于识别具有最高优先级的工厂。最后，实现`org.apache.logging.log4j.spi.LoggerContextFactory`的类将被实例化并绑定到LogManager。在Log4j2中，这由Log4jContextFactory提供。

应用程序可能通过以下方式更改将使用的LoggerContextFactory：

- 创建一个绑定到日志实现。1) 实现一个新的LoggerContextFactory。2) 
实现一个扩展`org.apache.logging.spi.Provider`的类。使用无参数构造函数来调用具有Priority，API版本，LoggerContextFactory类和可选的ThreadContextMap的实现类的超类构造函数。3) 创建一个`META-INF/services/org.apache.logging.spi.Provider`文件，其中包含实现`org.apache.logging.spi.Provider`的类的名称。
- 将系统属性`log4j2.loggerContextFactory`设置为要使用的LoggerContextFactory类的名称。
- 将名为`log4j2.LogManager.properties`的properties文件中的属性`log4j2.loggerContextFactory`设置为要使用的LoggerContextFactory类的名称。properties文件必须位于类路径中。

### 9.2 ContextSelector

ContextSelectors由Log4j LoggerContext工厂调用。他们执行定位或创建LoggerContext的实际工作，LoggerContext是Loggers及其配置的锚点。ContextSelectors可以自由地实现他们想要管理LoggerContexts的任何机制。默认的Log4jContextFactory检查是否存在名为Log4jContextSelector的系统属性。如果找到，则该属性应包含实现了要使用的ContextSelector类的名称。

Log4j提供了五个ContextSelectors：

- `BasicContextSelector`：使用已存储在ThreadLocal中的LoggerContext或普通的LoggerContext。
- `ClassLoaderContextSelector`：将LoggerContexts与创建getLogger调用的调用者的ClassLoader关联起来。这是默认的ContextSelector。
- `JndiContextSelector`：通过查询JNDI来查找LoggerContext。
- `AsyncLoggerContextSelector`：创建一个LoggerContext，确保所有记录器都是AsyncLogger。
- `BundleContextSelector`：将LoggerContexts与创建getLogger调用的调用者的一堆ClassLoader关联起来。这在OSGi环境中默认启用。

### 9.3 ConfigurationFactory

修改日志记录的配置方式通常是最感兴趣的领域之一。主要的方法是通过实现或扩展ConfigurationFactory。Log4j提供了两种添加新ConfigurationFactory的方法。第一种方法是将名为`log4j.configurationFactory`的系统属性定义为应首先搜索配置的类的名称。第二种方法是将ConfigurationFactory定义为Plugin。

然后按顺序处理所有ConfigurationFactories。每个工厂都在其getSupportedTypes方法上调用以确定它支持的文件扩展名。如果配置文件位于指定的文件扩展名之一中，则控制权将传递给该ConfigurationFactory以加载配置并创建配置对象。

大多数配置扩展了BaseConfiguration类。该类预计子类​​将处理配置文件并创建Node对象的层次结构。每个Node都相当简单，它由node名称，与node关联的name/value对，node的PluginType以及其所有child node的列表组成。然后BaseConfiguration将被传递给Node tree并从中实例化配置对象。

如下所示：

```java
@Plugin(name = "XMLConfigurationFactory", category = "ConfigurationFactory")
@Order(5)
public class XMLConfigurationFactory extends ConfigurationFactory {

    /**
     * Valid file extensions for XML files.
     */
    public static final String[] SUFFIXES = new String[] {".xml", "*"};

    /**
     * Returns the Configuration.
     * @param loggerContext The logger context.
     * @param source The InputSource.
     * @return The Configuration.
     */
    @Override
    public Configuration getConfiguration(final LoggerContext loggerContext, final ConfigurationSource source) {
        return new XmlConfiguration(loggerContext, source);
    }

    /**
     * Returns the file suffixes for XML files.
     * @return An array of File extensions.
     */
    public String[] getSupportedTypes() {
        return SUFFIXES;
    }
}
```

### 9.4 LoggerConfig

LoggerConfig对象是将应用程序创建的Logger绑定到配置中的地方。Log4j实现要求所有LoggerConfig都基于LoggerConfig类，因此希望进行更改的应用程序必须通过扩展LoggerConfig类来实现。要声明新的LoggerConfig，将其声明为类型为`Core`的Plugin并提供应用程序应在配置中指定为元素名称的名称。LoggerConfig还应该定义一个PluginFactory，它将创建LoggerConfig的一个实例。

以下示例显示root LoggerConfig如何简单的扩展通用LoggerConfig。

```java
@Plugin(name = "root", category = "Core", printObject = true)
public static class RootLogger extends LoggerConfig {

    @PluginFactory
    public static LoggerConfig createLogger(@PluginAttribute(value = "additivity", defaultBooleanValue = true) boolean additivity,
                                            @PluginAttribute(value = "level", defaultStringValue = "ERROR") Level level,
                                            @PluginElement("AppenderRef") AppenderRef[] refs,
                                            @PluginElement("Filters") Filter filter) {
        List<AppenderRef> appenderRefs = Arrays.asList(refs);
        return new LoggerConfig(LogManager.ROOT_LOGGER_NAME, appenderRefs, filter, level, additivity);
    }
}
```

### 9.5 LogEventFactory

LogEventFactory用于生成LogEvents。应用程序可以通过将系统属性`Log4jLogEventFactory`的值设置为自定义的LogEventFactory类名称来替换标准LogEventFactory。

注意：当log4j配置为使所有记录器异步时，日志事件将预先分配在环形缓冲区中，并且不使用LogEventFactory。

### 9.6 MessageFactory

MessageFactory用于生成Message对象。应用程序可以通过将系统属性`log4j2.messageFactory`的值设置为自定义MessageFactory类的名称来替换标准ParameterizedMessageFactory（或无垃圾模式下的ReusableMessageFactory）。

Logger.entry()和Logger.exit()方法的flow message具有单独的FlowMessageFactory。应用程序可以通过将系统属性`log4j2.flowMessageFactory`的值设置为自定义FlowMessageFactory类的名称来替换DefaultFlowMessageFactory。

### 9.7 Lookups

Lookup是执行参数替换的手段。在Configuration初始化过程中，会创建一个Interpolator用于查找所有Lookup并在变量需要解析时注册它们以供使用。interpolator将变量名称的prefix部分与注册的lookup相匹配，并将控制权交给它来解析变量。

必须使用类型为`Lookup`的Plugin注解来声明Lookup。Plugin注解中指定的名称将用于匹配前缀。与其他Plugin不同，Lookup不使用PluginFactory。相反，他们要求提供一个无参构造函数。下面的例子显示了一个将返回系统属性值的Lookup。

```java
@Plugin(name = "sys", category = "Lookup")
public class SystemPropertiesLookup implements StrLookup {

    /**
     * Lookup the value for the key.
     * @param key  the key to be looked up, may be null
     * @return The value for the key.
     */
    public String lookup(String key) {
        return System.getProperty(key);
    }

    /**
     * Lookup the value for the key using the data in the LogEvent.
     * @param event The current LogEvent.
     * @param key  the key to be looked up, may be null
     * @return The value associated with the key.
     */
    public String lookup(LogEvent event, String key) {
        return System.getProperty(key);
    }
}
```

### 9.8 Filters

Filter用于在日志通过日志记录系统时拒绝或接受。filter使用`Core`类型的Plugin注解和一个为"filter"的elementType声明。Plugin注解中的name属性用于指定用户用来启用Filter的元素的名称。指定printObject属性的值为true表示当对配置进行处理时，调用toString会将参数格式化后提供给为Filter。Filter还必须指定一个将被调用来创建Filter的PluginFactory方法。

下面的示例显示了一个用于基于LogEvent日志级别来判断拒绝的filter。注意所有filter方法解析为单个filter方法的典型pattern。

```java
@Plugin(name = "ThresholdFilter", category = "Core", elementType = "filter", printObject = true)
public final class ThresholdFilter extends AbstractFilter {

    private final Level level;

    private ThresholdFilter(Level level, Result onMatch, Result onMismatch) {
        super(onMatch, onMismatch);
        this.level = level;
    }

    public Result filter(Logger logger, Level level, Marker marker, String msg, Object[] params) {
        return filter(level);
    }

    public Result filter(Logger logger, Level level, Marker marker, Object msg, Throwable t) {
        return filter(level);
    }

    public Result filter(Logger logger, Level level, Marker marker, Message msg, Throwable t) {
        return filter(level);
    }

    @Override
    public Result filter(LogEvent event) {
        return filter(event.getLevel());
    }

    private Result filter(Level level) {
        return level.isAtLeastAsSpecificAs(this.level) ? onMatch : onMismatch;
    }

    @Override
    public String toString() {
        return level.toString();
    }

    /**
     * Create a ThresholdFilter.
     * @param loggerLevel The log Level.
     * @param match The action to take on a match.
     * @param mismatch The action to take on a mismatch.
     * @return The created ThresholdFilter.
     */
    @PluginFactory
    public static ThresholdFilter createFilter(@PluginAttribute(value = "level", defaultStringValue = "ERROR") Level level,
                                               @PluginAttribute(value = "onMatch", defaultStringValue = "NEUTRAL") Result onMatch,
                                               @PluginAttribute(value = "onMismatch", defaultStringValue = "DENY") Result onMismatch) {
        return new ThresholdFilter(level, onMatch, onMismatch);
    }
}
```

### 9.9 Appenders

Appender传递一个日志事件，（通常）调用Layout来格式化事件，然后以任何期望的方式publish事件。Appender被声明为Plugin，其类型为`Core`，elementType为"appender"。Plugin注解中的name属性指定了用户在其配置中必须提供以使用Appender的元素名称。如果使用toString方法呈现传递给Appender的属性值，则Appender应该将printObject指定为true。

Appender还必须声明一个将创建appender的PluginFactory方法。下面的例子显示了一个可以用作初始模板的名为Stub的Appender。

大多数Appender使用Manager。Manager实际上owns资源，例如OutputStream或socket。当重新配置发生时，将创建一个新的Appender。但是，如果以前的Manager没有什么重大变化，新的Appender将仅引用旧的Manager而不是创建一个新的。这确保日志事件在重新配置过程中不会丢失，而不需要在重新配置发生时发生暂停日志。

```java
@Plugin(name = "Stub", category = "Core", elementType = "appender", printObject = true)
public final class StubAppender extends OutputStreamAppender {

    private StubAppender(String name, Layout layout, Filter filter, StubManager manager,
                         boolean ignoreExceptions) {
    }

    @PluginFactory
    public static StubAppender createAppender(@PluginAttribute("name") String name,
                                              @PluginAttribute("ignoreExceptions") boolean ignoreExceptions,
                                              @PluginElement("Layout") Layout layout,
                                              @PluginElement("Filters") Filter filter) {

        if (name == null) {
            LOGGER.error("No name provided for StubAppender");
            return null;
        }

        StubManager manager = StubManager.getStubManager(name);
        if (manager == null) {
            return null;
        }
        if (layout == null) {
            layout = PatternLayout.createDefaultLayout();
        }
        return new StubAppender(name, layout, filter, manager, ignoreExceptions);
    }
}
```

### 9.10 Layouts

Layout将由Appenders写入某个目的地的事件的可打印文本进行格式化。所有的Layout必须实现Layout接口。将日志事件格式化为字符串的Layout应该扩展AbstractStringLayout，它将负责将字符串转换为所需的字节数组。

每个Layout必须使用Plugin注解声明自己为插件。类型必须是`Core`，而elementType必须是"layout"。如果插件的toString方法将提供对象及其参数的表示，则应将printObject设置为true。插件的名称必须与用户在Appender配置中将其指定为元素的值匹配。该插件还必须提供一个注解为PluginFactory的静态方法，并且每个方法参数都根据需要用PluginAttr或PluginElement注解。

```java
@Plugin(name = "SampleLayout", category = "Core", elementType = "layout", printObject = true)
public class SampleLayout extends AbstractStringLayout {

    protected SampleLayout(boolean locationInfo, boolean properties, boolean complete,
                           Charset charset) {
    }

    @PluginFactory
    public static SampleLayout createLayout(@PluginAttribute("locationInfo") boolean locationInfo,
                                            @PluginAttribute("properties") boolean properties,
                                            @PluginAttribute("complete") boolean complete,
                                            @PluginAttribute(value = "charset", defaultStringValue = "UTF-8") Charset charset) {
        return new SampleLayout(locationInfo, properties, complete, charset);
    }
}
```

### 9.11 PatternConverters

PatternLayout使用PatternConverters将日志事件格式化为可打印的字符串。每个Converter都负责一种操作，但Converters可以自由地以复杂的方式格式化事件。例如，有几个转换器操作Throwables并以各种方式对其进行格式化。

一个PatternConverter必须首先使用标准Plugin注解声明自己为插件，但必须在type属性中指定Converter的值。此外，Converter还必须指定ConverterKeys属性来定义可在pattern中指定的token（以%字符开头）以标识Converter。

与大多数其他插件不同，Converter不使用PluginFactory。相反，每个Converter都需要提供一个静态newInstance方法，该方法接受一个字符串数组作为唯一参数。字符串数组在大括号内指定，是可以遵循converter键的的值。

以下显示了Converter插件的框架：

```java
@Plugin(name = "query", category = "Converter")
@ConverterKeys({"q", "query"})
public final class QueryConverter extends LogEventPatternConverter {

    public QueryConverter(String[] options) {
    }

    public static QueryConverter newInstance(final String[] options) {
      return new QueryConverter(options);
    }
}
```

### 9.12 Plugin Buiders

一些插件需要很多可选的配置选项。当插件需要多个选项时，使用构建器类而不是工厂方法会更易于维护。通过注解工厂方法使用注解构建器类还有其他一些优点：

- 如果属性名称与字段名称匹配，则不需要指定属性名称。
- 默认值可以在代码中指定，而不是通过注解（允许运行时计算出的默认值也不允许注解）。
- 添加新的可选参数不需要重构现有的编程配置。
- 更易于使用构建器编写单元测试，而不是使用具有可选参数的工厂方法。
- 默认值是通过代码指定的，而不是依赖反射和注入，所以它们像在配置文件中一样通过编程方式工作。

下面是ListAppender的一个插件工厂的例子：

```java
@PluginFactory
public static ListAppender createAppender(
        @PluginAttribute("name") @Required(message = "No name provided for ListAppender") final String name,
        @PluginAttribute("entryPerNewLine") final boolean newLine,
        @PluginAttribute("raw") final boolean raw,
        @PluginElement("Layout") final Layout<? extends Serializable> layout,
        @PluginElement("Filter") final Filter filter) {
    return new ListAppender(name, filter, layout, newLine, raw);
}
```

下面是使用生成器模式的同一个插件工厂：

```java
@PluginBuilderFactory
public static Builder newBuilder() {
    return new Builder();
}

public static class Builder implements org.apache.logging.log4j.core.util.Builder<ListAppender> {

    @PluginBuilderAttribute
    @Required(message = "No name provided for ListAppender")
    private String name;

    @PluginBuilderAttribute
    private boolean entryPerNewLine;

    @PluginBuilderAttribute
    private boolean raw;

    @PluginElement("Layout")
    private Layout<? extends Serializable> layout;

    @PluginElement("Filter")
    private Filter filter;

    public Builder setName(final String name) {
        this.name = name;
        return this;
    }

    public Builder setEntryPerNewLine(final boolean entryPerNewLine) {
        this.entryPerNewLine = entryPerNewLine;
        return this;
    }

    public Builder setRaw(final boolean raw) {
        this.raw = raw;
        return this;
    }

    public Builder setLayout(final Layout<? extends Serializable> layout) {
        this.layout = layout;
        return this;
    }

    public Builder setFilter(final Filter filter) {
        this.filter = filter;
        return this;
    }

    @Override
    public ListAppender build() {
        return new ListAppender(name, filter, layout, entryPerNewLine, raw);
    }
}
```

注解中的唯一区别是使用`@PluginBuilderAttribute`而不是`@PluginAttribute`，因此可以使用默认值和反射，而不是在注解中指定它们。任何一种注解都可以在构建器中使用，但前者更适合于字段注入，而后者更适合参数注入。否则，相同的注解（@PluginConfiguration，@PluginElement，@PluginNode和@PluginValue）均在字段上受支持。请注意，提供构建器工厂方法仍然需要提供构建器，并且此工厂方法应使用`@PluginBuilderFactory`进行注解。

在解析配置后构建插件时，将使用插件构建器（如果可用），否则插件工厂方法将用作后备。如果一个插件既不包含工厂，则不能从配置文件中使用（当然，它仍然可以用编程方式使用）。

以下是以编程方式使用插件工厂与插件生成器的示例：

```java
ListAppender list1 = ListAppender.createAppender("List1", true, false, null, null);
ListAppender list2 = ListAppender.newBuilder().setName("List1").setEntryPerNewLine(true).build();
```

### 9.13 Custom ContextDataInjector

ContextDataInjector（在Log4j2.7中引入）负责使用键值对填充LogEvent的context data或完全替换它。默认实现是ThreadContextDataInjector，它从ThreadContext获取上下文属性。

应用程序可以通过将系统属性`log4j2.contextDataInjector`的值设置为自定义ContextDataInjector类的名称来替换默认的ContextDataInjector。

实现者应该意识到与线程安全相关的一些细微之处，并且以无垃圾的方式实现context data injector。

### 9.14 Custom ThreadContextMap implementations

通过将系统属性log4j2.garbagefreeThreadContextMap设置为true，可以安装无垃圾的基于StringMap的上下文映射。（必须启用Log4j才能使用ThreadLocals。）

通过将系统属性`log4j2.threadContextMap`设置为实现ThreadContextMap接口的类的完全限定类名，可以安装任何自定义ThreadContextMap实现。还实现ReadOnlyThreadContextMap接口，您的自定义ThreadContextMap实现将可通过`ThreadContext::getThreadContextMap`方法访问应用程序。

### 9.15 Custom Plugins

参见Plugins章节。