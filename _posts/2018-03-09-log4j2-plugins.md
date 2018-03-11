---
layout: post
title: "Log4j2 -- Plugins"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 10. Plugins

### 10.1 Introduction

Log4j1.x通过在大多数配置声明中要求类属性的方式进行扩展。对于某些元素，特别是PatternLayout，添加新pattern转换器的唯一方法是扩展PatternLayout类并通过代码添加它们。Log4j2的一个目标是通过使用插件来非常容易的扩展它。

在Log4j2中，插件是通过向类声明中添加`@Plugin`注解来声明的。在初始化期间，Configuration将调用PluginManager来加载内置的Log4j插件以及任何自定义插件。PluginManager通过查找以下五个位置来查找插件：

- 类路径中的序列化插件列表文件。这些文件是在构建过程中自动生成的（更多细节见下文）。
- 每个活动OSGi bundle中的序列化插件列表文件（仅限OSGi）。激活时会添加一个BundleListener，以在log4j-core启动后继续检查新捆绑。
- 由log4j.plugin.packages系统属性指定的包的逗号分隔列表。
- 传递给静态PluginManager.addPackages方法的包（发生在Log4j配置之前）。
- 在log4j2配置文件中声明的包。

如果多个插件指定相同（不区分大小写）的name，则上面的加载顺序决定使用哪一个。例如，要覆盖由内置的FileAppender类提供的File插件，您需要将插件放置在CLASSPATH中的JAR文件中，位于log4j-core.jar之前。不建议此类操作；插件名称冲突将导致发出警告。请注意，在OSGi环境中，针对扫描bundles以获取插件的顺序通常与将bundles安装到框架中的顺序相同。简而言之，在OSGi环境中，名称冲突更加不可预测。

序列化插件列表文件由log4j-core中包含的注解处理器生成，该工具将自动扫描Log4j2插件的代码并在处理的类中输出metadata文件。无需多做配置，Java编译器将自动选取类路径上的注解处理器，除非您明确禁用它。在这种情况下，为构建过程添加另一个编译器是非常重要的，该编译器只使用Log4j2注解处理器类`org.apache.logging.log4j.core.config.plugins.processor.PluginProcessor`处理注解。要使用Apache Maven执行此操作，请将以下执行添加到您的`maven-compiler-plugin`（版本2.2或更高版本）构建插件中：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.1</version>
  <executions>
    <execution>
      <id>log4j-plugin-processor</id>
      <goals>
        <goal>compile</goal>
      </goals>
      <phase>process-classes</phase>
      <configuration>
        <proc>only</proc>
        <annotationProcessors>
          <annotationProcessor>org.apache.logging.log4j.core.config.plugins.processor.PluginProcessor</annotationProcessor>
        </annotationProcessors>
      </configuration>
    </execution>
  </executions>
</plugin>
```

随着配置的处理，相应的插件将被自动配置和初始化。Log4j2使用了几个不同类别的插件，这些插件将在以下各节中介绍。

### 10.2 Core

Core插件是那些直接由配置文件中的元素表示的插件，例如Appender，Layout，Logger或Filter。符合下一段中规定的规则的自定义插件可以简单地在配置中引用，只要它们合适的配置为由PluginManager加载即可。

每个Core插件必须声明一个用`@PluginFactory`或`@PluginBuilderFactory`注解的静态方法。前者用于提供所有选项作为方法参数的静态工厂方法，后者用于构造一个新的`Builder<T>`类，其字段用于注入属性和子节点。要允许Configuration将正确的参数传递给方法，必须将方法的每个参数注解为以下属性类型之一。每个属性或元素注解都必须包含配置中必须存在的名称，以便将配置项与其各自的参数相匹配。对于插件构建器，如果在注解中没有指定名称，那么将默认使用这些字段的名称。Log4j Core中有很多插件，可用作更复杂场景的示例，包括分层构建器类（例如，参见FileAppender）。有关更多详细信息，请参阅使用插件构建器扩展Log4j。

#### Attribute Types

**PluginAttribute**

该参数必须可以使用TypeConverter从字符串转换。大多数内置类型已经被支持，但是也可以提供自定义的TypeConverter插件来支持更多的类型。请注意，可以在构建器类字段中使用PluginBuilderAttribute以更简单的提供默认值。

**PluginElement**

该参数可能表示一个复杂的对象，它本身具有可配置的参数。这也支持注入一组元素。

**PluginConfiguration**

当前的Configuration对象将作为参数传递给插件。

**PluginNode**

当前被解析的Node将作为参数传递给插件。

**PluginValue**

当前Node的值或其属性名称为value的值。

#### Constraint Validators

Plugin工厂字段和参数可以在运行时使用Bean Validation spec的约束验证器自动验证。以下注解捆绑在Log4j中，但也可以创建自定义ConstraintValidators。

**Required**

这个注解验证一个值是非空的。这包括对null以及其他几种情况的检查：空CharSequence对象，空数组，空Collection实例和空Map实例。

**ValidHost**

这个注解验证一个值对应于一个有效的主机名。这与使用`InetAddress::getByName`的验证相同。

**ValidPort**

此注解验证一个值对应于0到65535之间的有效端口号。

### 10.3 Converters

PatternLayout使用转换器来呈现由conversion pattern标识的元素。每个converter都必须在`@Plugin`注解中指定其类别为Converter，并有一个静态newInstance方法，该方法接受一个String数组作为其唯一参数并返回一个Converter实例，并且必须有一个`@ConverterKeys`注解，其中包含将导致Converter被选中的一组converter pattern。意图处理LogEvent的Converter必须扩展LogEventPatternConverter类，并且必须实现一个接受LogEvent和StringBuilder作为参数的格式化方法。Converter应该将其操作结果附加到StringBuilder。

第二种类型的Converter是FileConverter，它必须在`@Plugin`注解的category属性中指定的FileConverter。虽然类似于LogEventPatternConverter，而不是单一格式的方法，但这些Converter将有两个变体：一个接受一个Object，另一个接收一个Object数组而不是LogEvent。两者都以与LogEventPatternConverter相同的方式附加到提供的StringBuilder。这些Converter通常由RollingFileAppender使用来构建要输出日志的文件的名称。

如果多个Converter指定了相同的ConverterKeys，则上面的加载顺序决定将使用哪一个。例如，要覆盖由内置的DatePatternConverter类提供的`%date`转换器，您需要将插件放置在CLASSPATH中的JAR文件中，位于log4j-core.jar之前。不建议这样做，pattern ConverterKeys冲突会导致发出警告。尝试为您的自定义pattern converter使用独特的ConverterKeys。

### 10.4 KeyProviders

Log4j中的一些组件可以提供执行数据加密的功能。这些组件需要密钥才能执行加密。应用程序可以通过创建一个实现SecretKeyProvider接口的类来提供密钥。

### 10.5 Lookups

Lookup也许是最简单的插件。它们必须在插件注解中将他们的type声明为Lookup，并且必须实现StrLookup接口。他们将有两种方法：一种接受String键并返回一个String值的查找方法；第二种接受LogEvent和String key并返回一个String。 Lookup可以通过指定`${name:key}`来引用查找，其中name是在Plugin注解中指定的名称，而key是要查找的项目的名称。

### 10.6 TypeConverters

TypeConverters是一种用于在插件工厂方法参数中将字符串转换为其他类型的meta-plugin。其他插件可以通过`@PluginElement`注解注入；现在，被类型转换系统支持的任何类型都可以用在`@PluginAttribute`参数中。枚举类型的转换将按需支持，不需要自定义的TypeConverter类。大量的内置Java类已经被支持。

与其他插件不同，TypeConverter的插件名称完全是cosmetic的。适当的类型转换器通过Type接口而不是仅仅通过`Class<?>`对象查找。请注意TypeConverter插件必须有一个默认的构造函数。

### 10.7 Developer Notes

如果插件类实现了Collection或Map，则不使用工厂方法。而是使用默认构造函数实例化类，并将所有子配置节点添加到Collection或Map中。