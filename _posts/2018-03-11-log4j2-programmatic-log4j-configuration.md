---
layout: post
title: "Log4j2 -- Programmatic Log4j Configuration"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 11. Programmatic Log4j Configuration

Log4j2为应用程序创建自己的编程配置提供了几种方法：

- 指定一个自定义ConfigurationFactory，以编程配置启动Log4j。
- 在Log4j启动后使用Configurator替换配置。
- 使用配置文件和编程配置的组合来初始化Log4j。
- 初始化后修改当前配置。

### 11.1 The ConfigurationBuilder API

从版本2.4开始，Log4j提供了一个ConfigurationBuilder和一组组件构建器，可以很容易地创建一个Configuration。像LoggerConfig或Appender这样的实际配置对象可能很笨拙；由于他们需要大量关于Log4j内部的知识，这使得如果你想要创建一个配置，它们很难处理。

新的ConfigurationBuilder API（位于`org.apache.logging.log4j.core.config.builder.api`包中）允许用户通过构建组件definitions在代码中创建配置，并不需要直接处理实际的配置对象。组件definitions被添加到ConfigurationBuilder中，并且一旦收集完所有的定义，所有的实际配置对象（如Logger和Appender）都被构造。

ConfigurationBuilder为可配置的基本组件提供了便利方法，如Logger，Appender，Filter，Properties等。但是，Log4j2的插件机制意味着用户可以创建任意数量的自定义组件。作为折衷，ConfigurationBuilder API仅提供有限数量的“强类型”便利方法，如newLogger()，newLayout()等。如果组件没有便利方法，则可以使用通用`builder.newComponent()`方法配置。

例如，构建器不知道可以在特定组件上配置哪些子组件，例如RollingFileAppender与RoutingAppender。要在RollingFileAppender上指定触发policy，您可以使用`builder.newComponent()`。

使用ConfigurationBuilder API的示例在后面的章节中。

### 11.2 Understanding ConfigurationFactory

在初始化期间，Log4j2将搜索可用的ConfigurationFactories，然后选择一个要使用的。所选的ConfigurationFactory创建Log4j将使用的配置。以下是Log4j如何找到可用的ConfigurationFactories：

1. 将log4j2.configurationFactory的系统属性名设置为要使用的ConfigurationFactory的名称。
2. 可以使用ConfigurationFactory的实例调用ConfigurationFactory.setConfigurationFactory（ConfigurationFactory）。这必须在Log4j的其他调用之前调用。
3. ConfigurationFactory实现可以添加到类路径中，并配置为“ConfigurationFactory”类别中的插件。当找到多个可用的ConfigurationFactor时，Order注解可用于指定相对优先级。

ConfigurationFactories具有“受支持的类型”的概念，它基本上映射到ConfigurationFactory可以处理的配置文件的文件扩展名。如果指定了配置文件位置，则受支持类型不包含`*`或不包含匹配文件扩展名的ConfigurationFactories不会使用。

### 11.3 Initialize Log4j Using ConfigurationBuilder with a Custom ConfigurationFactory

一种以编程方式配置Log4j2的方法是创建一个使用ConfigurationBuilder所创建的自定义ConfigurationFactory创建一个Configuration。以下示例覆盖getConfiguration()方法以返回由ConfigurationBuilder创建的Configuration。这会导致当创建LoggerContext时，configuration会自动挂钩到Log4j中。在下面的例子中，因为它指定了一个受支持的`*`类型，它将覆盖所有提供的配置文件。

```java
@Plugin(name = "CustomConfigurationFactory", category = ConfigurationFactory.CATEGORY)
@Order(50)
public class CustomConfigurationFactory extends ConfigurationFactory {

    static Configuration createConfiguration(final String name, ConfigurationBuilder<BuiltConfiguration> builder) {
        builder.setConfigurationName(name);
        builder.setStatusLevel(Level.ERROR);
        builder.add(builder.newFilter("ThresholdFilter", Filter.Result.ACCEPT, Filter.Result.NEUTRAL).
            addAttribute("level", Level.DEBUG));
        AppenderComponentBuilder appenderBuilder = builder.newAppender("Stdout", "CONSOLE").
            addAttribute("target", ConsoleAppender.Target.SYSTEM_OUT);
        appenderBuilder.add(builder.newLayout("PatternLayout").
            addAttribute("pattern", "%d [%t] %-5level: %msg%n%throwable"));
        appenderBuilder.add(builder.newFilter("MarkerFilter", Filter.Result.DENY,
            Filter.Result.NEUTRAL).addAttribute("marker", "FLOW"));
        builder.add(appenderBuilder);
        builder.add(builder.newLogger("org.apache.logging.log4j", Level.DEBUG).
            add(builder.newAppenderRef("Stdout")).
            addAttribute("additivity", false));
        builder.add(builder.newRootLogger(Level.ERROR).add(builder.newAppenderRef("Stdout")));
        return builder.build();
    }

    @Override
    public Configuration getConfiguration(final LoggerContext loggerContext, final ConfigurationSource source) {
        return getConfiguration(loggerContext, source.toString(), null);
    }

    @Override
    public Configuration getConfiguration(final LoggerContext loggerContext, final String name, final URI configLocation) {
        ConfigurationBuilder<BuiltConfiguration> builder = newConfigurationBuilder();
        return createConfiguration(name, builder);
    }

    @Override
    protected String[] getSupportedTypes() {
        return new String[] {"*"};
    }
}
```

从版本2.7开始，ConfigurationFactory.getConfiguration()方法会采用额外的LoggerContext参数。

### 11.4 Reconfigure Log4j Using ConfigurationBuilder with the Configurator

自定义ConfigurationFactory的替代方法是使用Configurator进行配置。一旦构造了一个Configuration对象，它就可以传递给其中一个Configurator.initialize方法来设置Log4j的配置。

以这种方式使用Configurator允许应用程序控制何时初始化Log4j。但是，如果在Configurator.initialize()被调用之前尝试进行任何日志记录，则这些日志事件将使用默认配置。

```java
ConfigurationBuilder<BuiltConfiguration> builder = ConfigurationBuilderFactory.newConfigurationBuilder();
builder.setStatusLevel(Level.ERROR);
builder.setConfigurationName("BuilderTest");
builder.add(builder.newFilter("ThresholdFilter", Filter.Result.ACCEPT, Filter.Result.NEUTRAL)
    .addAttribute("level", Level.DEBUG));
AppenderComponentBuilder appenderBuilder = builder.newAppender("Stdout", "CONSOLE").addAttribute("target",
    ConsoleAppender.Target.SYSTEM_OUT);
appenderBuilder.add(builder.newLayout("PatternLayout")
    .addAttribute("pattern", "%d [%t] %-5level: %msg%n%throwable"));
appenderBuilder.add(builder.newFilter("MarkerFilter", Filter.Result.DENY, Filter.Result.NEUTRAL)
    .addAttribute("marker", "FLOW"));
builder.add(appenderBuilder);
builder.add(builder.newLogger("org.apache.logging.log4j", Level.DEBUG)
    .add(builder.newAppenderRef("Stdout")).addAttribute("additivity", false));
builder.add(builder.newRootLogger(Level.ERROR).add(builder.newAppenderRef("Stdout")));
ctx = Configurator.initialize(builder.build());
```

此示例显示如何创建包含RollingFileAppender的配置。

```java
ConfigurationBuilder< BuiltConfiguration > builder = ConfigurationBuilderFactory.newConfigurationBuilder();

builder.setStatusLevel( Level.ERROR);
builder.setConfigurationName("RollingBuilder");
// create a console appender
AppenderComponentBuilder appenderBuilder = builder.newAppender("Stdout", "CONSOLE").addAttribute("target",
    ConsoleAppender.Target.SYSTEM_OUT);
appenderBuilder.add(builder.newLayout("PatternLayout")
    .addAttribute("pattern", "%d [%t] %-5level: %msg%n%throwable"));
builder.add( appenderBuilder );
// create a rolling file appender
LayoutComponentBuilder layoutBuilder = builder.newLayout("PatternLayout")
    .addAttribute("pattern", "%d [%t] %-5level: %msg%n");
ComponentBuilder triggeringPolicy = builder.newComponent("Policies")
    .addComponent(builder.newComponent("CronTriggeringPolicy").addAttribute("schedule", "0 0 0 * * ?"))
    .addComponent(builder.newComponent("SizeBasedTriggeringPolicy").addAttribute("size", "100M"));
appenderBuilder = builder.newAppender("rolling", "RollingFile")
    .addAttribute("fileName", "target/rolling.log")
    .addAttribute("filePattern", "target/archive/rolling-%d{MM-dd-yy}.log.gz")
    .add(layoutBuilder)
    .addComponent(triggeringPolicy);
builder.add(appenderBuilder);

// create the new logger
builder.add( builder.newLogger( "TestLogger", Level.DEBUG )
    .add( builder.newAppenderRef( "rolling" ) )
    .addAttribute( "additivity", false ) );

builder.add( builder.newRootLogger( Level.DEBUG )
    .add( builder.newAppenderRef( "rolling" ) ) );
LoggerContext ctx = Configurator.intitialize(builder.build());
```

### 11.5 Initialize Log4j by Combining Configuration File with Programmatic Configuration

有时候你想用一个配置文件进行配置，但要做一些额外的编程配置。可能是您希望允许使用XML的灵活配置，但同时确保有一些总是存在的配置元素无法删除。

实现此目的的最简单方法是扩展标准Configuration类之一（XMLConfiguration，JSONConfiguration），然后为扩展类创建一个新的ConfigurationFactory。标准配置完成后，可以将自定义配置添加到该配置中。

下面的示例显示了如何扩展XMLConfiguration以手动添加Appender和LoggerConfig到配置。

```java
@Plugin(name = "MyXMLConfigurationFactory", category = "ConfigurationFactory")
@Order(10)
public class MyXMLConfigurationFactory extends ConfigurationFactory {

    /**
     * Valid file extensions for XML files.
     */
    public static final String[] SUFFIXES = new String[] {".xml", "*"};

    /**
     * Return the Configuration.
     * @param source The InputSource.
     * @return The Configuration.
     */
    public Configuration getConfiguration(InputSource source) {
        return new MyXMLConfiguration(source, configFile);
    }

    /**
     * Returns the file suffixes for XML files.
     * @return An array of File extensions.
     */
    public String[] getSupportedTypes() {
        return SUFFIXES;
    }
}

public class MyXMLConfiguration extends XMLConfiguration {
    public MyXMLConfiguration(final ConfigurationFactory.ConfigurationSource configSource) {
      super(configSource);
    }

    @Override
    protected void doConfigure() {
        super.doConfigure();
        final LoggerContext ctx = (LoggerContext) LogManager.getContext(false);
        final Layout layout = PatternLayout.createLayout(PatternLayout.SIMPLE_CONVERSION_PATTERN, config, null,
              null,null, null);
        final Appender appender = FileAppender.createAppender("target/test.log", "false", "false", "File", "true",
              "false", "false", "4000", layout, null, "false", null, config);
        appender.start();
        addAppender(appender);
        LoggerConfig loggerConfig = LoggerConfig.createLogger("false", "info", "org.apache.logging.log4j",
              "true", refs, null, config, null );
        loggerConfig.addAppender(appender, null, null);
        addLogger("org.apache.logging.log4j", loggerConfig);
    }
}
```

### 11.6 Programmatically Modifying the Current Configuration after Initialization

应用程序有时需要独立于实际配置的自定义日志记录。Log4j允许这样做，尽管受到一些限制：

1. 如果配置文件发生更改，配置将重新加载，手动更改将丢失。
2. 修改正在运行的配置需要同步所有被调用的方法（addAppender和addLogger）。

因此，定制配置的推荐方法是扩展其中一个标准Configuration类，覆盖setup方法首先执行super.setup()，然后在注册之前将自定义Appenders，Filters和LoggerConfigs添加到配置中使用。

以下示例将Appender和使用该Appender的新LoggerConfig添加到当前配置中。

```java
final LoggerContext ctx = (LoggerContext) LogManager.getContext(false);
final Configuration config = ctx.getConfiguration();
Layout layout = PatternLayout.createLayout(PatternLayout.SIMPLE_CONVERSION_PATTERN, config, null,
    null,null, null);
Appender appender = FileAppender.createAppender("target/test.log", "false", "false", "File", "true",
    "false", "false", "4000", layout, null, "false", null, config);
appender.start();
config.addAppender(appender);
AppenderRef ref = AppenderRef.createAppenderRef("File", null, null);
AppenderRef[] refs = new AppenderRef[] {ref};
LoggerConfig loggerConfig = LoggerConfig.createLogger("false", "info", "org.apache.logging.log4j",
    "true", refs, null, config, null );
loggerConfig.addAppender(appender, null, null);
config.addLogger("org.apache.logging.log4j", loggerConfig);
ctx.updateLoggers();
```

### 11.7 Appending Log Events to Writers and OutputStreams Programmatically

Log4j 2.5提供了将日志事件追加到Writers和OutputStreams的工具。例如，为在内部使用Log4j的JDBC驱动实现者提供了简单的集成，并且仍然希望支持JDBC API的`CommonDataSource.setLogWriter(PrintWriter)`，`java.sql.DriverManager.setLogWriter(PrintWriter)`和`java.sql.DriverManager.setLogStream(PrintStream)`。

给定任何Writer（如PrintWriter），您可以通过创建WriterAppender并更新Log4j配置来告诉Log4j将事件append到该Writer：

```java
void addAppender(final Writer writer, final String writerName) {
    final LoggerContext context = LoggerContext.getContext(false);
    final Configuration config = context.getConfiguration();
    final PatternLayout layout = PatternLayout.createDefaultLayout(config);
    final Appender appender = WriterAppender.createAppender(layout, null, writer, writerName, false, true);
    appender.start();
    config.addAppender(appender);
    updateLoggers(appender, config);
}

private void updateLoggers(final Appender appender, final Configuration config) {
    final Level level = null;
    final Filter filter = null;
    for (final LoggerConfig loggerConfig : config.getLoggers().values()) {
        loggerConfig.addAppender(appender, level, filter);
    }
    config.getRootLogger().addAppender(appender, level, filter);
}
```

您可以使用OutputStream实现相同的效果，如PrintStream：

```java
void addAppender(final OutputStream outputStream, final String outputStreamName) {
    final LoggerContext context = LoggerContext.getContext(false);
    final Configuration config = context.getConfiguration();
    final PatternLayout layout = PatternLayout.createDefaultLayout(config);
    final Appender appender = OutputStreamAppender.createAppender(layout, null, outputStream, outputStreamName, false, true);
    appender.start();
    config.addAppender(appender);
    updateLoggers(appender, config);
}
```

不同的是使用OutputStreamAppender而不是WriterAppender。