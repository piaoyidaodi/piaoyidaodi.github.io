---
layout: post
title: "Maven -- Learn Maven I"
categories: Log4j2
tag: Java-Libs
---
> `《Maven实战》读书记一记 - I`

## 1. maven的安装和配置

### 1.1 安装目录分析

#### 1.1.1 M2_HOME

maven根目录的目录内容及作用：

- `bin`：该目录包含了mvn的运行脚本以配置java命令，准备好classpath和相关的java系统属性，然后执行java命令。其中mvn为shell脚本，mvn.bat为Windows平台bat脚本。该目录还包含mvnDebug和mvnDebug.bat，其增加了MAVEN_DEBUG_OPTS配置，在运行maven时开启debug以调试maven本身。此外还包括m2.conf文件，这是classworlds的配置文件。
- `boot`：以maven3.0为例，只包含plexus-classworlds-2.x.x.jar。plexus-classworlds是一个类加载器框架，相比java类加载器，它提供了更加丰富的语法，maven使用该框架加载自己的类库。
- `conf`：包含非常重要的settings.xml。直接修改会全局定制maven的行为，一般应复制该文件至`~/.m2/`目录下修改，在用户范围定制maven的行为。
- `lib`：包含maven运行所需的所有java类库。

#### 1.1.2 ~/.m2

使用`mvn help:system`，打印所有的java系统属性和环境变量。

在用户目录下可发现.m2文件夹，默认情况下放置了本地仓库`.m2/repository`，所有的maven构件都存储到该仓库。

### 1.2 maven安装最佳实践

- **配置MAVEN_OPTS**：mvn命令实际上执行了java命令，可使用MAVEN_OPTS环境变量使用java参数。通常需设置MAVEN_OPTS的值为`-Xms128m -Xmx512m`，防止项目过大引起OutOfMemoryError。
- **配置用户settings.xml**：用户可选择配置`M2_HOME/conf/settings.xml`或`~/.m2/settings.xml`。前者是全局范围的，后者只有该用户才会受影响。且用户范围的settings.xml方便升级。

## 2. maven使用入门

### 2.1 编写POM（Project Object Model）

Maven项目的核心为`pom.xml`，HelloWorld项目示例如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>cc.joyjon.mvn</groupId>
    <artifactId>hello-world</artifactId>
    <version>1.0-SNAPSHOT</version>
    <name>Maven Hello World Project</name>
</project>
```

- `XML头`：代码第一行是XML头，指定了xml文档的版本和编码。
- `project元素`：是所有pom.xml的根元素，声明了一些POM相关的命名空间及xsd元素，虽然这些属性非必须，但是可以帮助第三方工具快速编辑POM。
- `modelVersion元素`：指定了当前POM模型版本，对于maven2和maven3只能为4.0.0。
- `groupId,artifactId,version`定义了项目的基本坐标，任何jar，pom或war都以这些基本左边区分。
- `groupId元素`：定义了项目所属的**组**。
- `artifactId元素`：定义了在**当前项目组**中的唯一ID。
- `version元素`：定义了项目版本，SNAPSHOT为快照，说明项目尚不稳定处于开发阶段。
- `name元素`：声明了对用户更友好的项目名称。

`xmlns、xmlns:namespace-prefix、xsi:schemaLocation`的联系区别：

> `xmlns`为**XMLNamespace**的缩写，翻译为xml的命名空间，解决了文档的命名冲突问题；`xmlns:namespace-prefix="namespaceURI"`其中namespace-prefix为自定义前缀，在这个XML文档中保证前缀不重复即可，namespaceURI是这个前缀对应的XMLNamespace的定义，如示例中`xsi`与后面的URI的内容绑定；`xsi:schemaLocation`属性其实是`http://www.w3.org/2001/XMLSchema-instance`里的schemaLocation属性。

### 2.2 编写主代码

maven的项目主代码位于`src/main/java`目录中，会被打包到最终构件中，而测试代码位于`src/main/test`中，不会打包。主代码示例如下：

```java
package cc.joyjon.mvntest.helloworld;

public class HelloWorld{
    public String sayHello(){
        return "Hello Maven";
    }

    public static void main(String[] args){
        System.out.print(new HelloWorld().sayHello());
    }
}
```

注意两点：

- 大多数情况下，无需额外配置，maven会自动搜寻约定文件夹寻找项目主代码。
- 一般来说项目中的java类包应基于配置文件中的`groupId`和`artifactId`。

执行`mvn clean compile`：`clean`告诉maven清理输出目录`target`，`compile`告诉maven编译项目主代码。首先执行`clean:clean`删除`target`目录，接着执行`resources:resources`（未定义资源暂时略过），最后执行`complier:complier`，将主代码编译至`target/classes`目录。其中`clean:clean`等对应了maven插件及插件目标。

### 2.3 编写测试代码

pom.xml示例代码如下所示，为项目增加Junit单元测试：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>cc.joyjon.mvn</groupId>
    <artifactId>hello-world</artifactId>
    <version>1.0-SNAPSHOT</version>
    <name>Maven Hello World Project</name>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.7</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

添加了`dependencies`元素，可包含多个`dependency`元素。其中`scope`为依赖范围，表示只对测试有效，即在主代码中`import JUnit`会造成编译错误，默认为`compile`。

一个典型的单元测试包括三个步骤：

- 准备测试类及数据。
- 执行要测试的行为。
- 检查结果。

在JUnit3中约定所有的测试方法都以test开头，Junit4中所有测试方法应以@Test进行标注，但仍遵循这一约定。测试代码如下所示：

```java
package cc.joyjon.mvntest.helloworld;

import static org.junit.Assert.assertEquals;
import org.junit.Test;

public class HelloWorldTest{
    @Test
    public void testSayHello(){
        HelloWorld helloWorld=new HelloWorld();
        String result=helloWorld.sayHello();
        assertEquals("HelloWorld",result);
    } 
}
```


运行测试方法`mvn clean test`时，实际执行了`clean:clean`、`resources:resources`、`complier:complier`、`resources:testResources`、`compiler:testCompile`。暂时需要了解的是，在maven执行测试之前，会首先自动执行项目主资源的处理、编译，测试资源的处理、编译等。

历史原因maven的核心插件之一，compiler插件默认只支持编译java1.3，因此需要配置其支持java5。代码如下：

```xml
<project>
...
    <build>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.5</source>
                <target>1.5</target>
            </configuration>
        </plugin>
    </build>
...
</project>
```

按上述操作执行之后，运行`surefire:test`任务，其中surefire是maven中负责执行测试的插件，负责运行测试用例并输出测试报告。

### 2.4 打包和运行

maven的默认打包类型为jar，通过执行`mvn clean package`进行打包。在打包前会执行编译、测试等操作，其中`jar:jar`任务是jar插件的jar目标将项目主代码打包成hello-world-1.0-SNAPSHOT.jar文件并放置与target目录中。

通过使用`mvn clean install`将项目输出的jar安装到本地仓库。

任何一个maven项目需经历compile, test, package, install的过程。

### 2.5 使用Archetype生成项目骨架

在maven3中使用`mvn archetype:generate`命令可解析生成最稳定的版本，在maven2中不行。

命令实际上运行了`maven-archetype-plugin`插件，运行后输出很多Archetype，每一个前面都会有对应的编号，同时命令行会提示一个默认编号，其对应的Archetype为`maven-archetype-quickstart`，回车选择后，Maven会提示创建项目的groupId、artifactId、version以及package。

## 3. 坐标和依赖

### 3.1 坐标详解

maven的坐标元素包括：groupId, artifactId, version, packaging, classifier。其默认的中央仓库地址为`http://repo1.maven.org/maven2`。

- `groupId`：定义当前maven项目隶属的实际项目。注意，maven项目和实际项目不一定是一对一关系，比如SpringFramework实际项目对应的maven项目有很多，如spring-core等。
- `artifactId`:定义实际项目中的一个maven项目（模块），推荐使用实际项目名称作为artifactId的**前缀**。
- `version`:定义maven项目中当前所处的版本。
- `packaging`:定义maven项目的打包方式。
- `classifier`:定义构建输出的一些附属构件。如果主构件为nexus-indexer-2.0.0.jar，若生成文档代码nexus-indexer-2.0.0-doc.jar，则此时doc为附属构件。

其中前三项为必须定义的，packaging是可选，而classifier是不能直接定义的。

### 3.2 依赖范围

maven在编译项目主代码时使用一套classpath；在执行测试时使用另一套classpath；实际运行时又用一套classpath；在`scope`中定义：

- `compile`：**编译依赖范围，默认使用**。使用此范围的依赖，在编译、测试、运行时都有效。
- `test`：**测试依赖范围**。只对于测试classpath有效。典型如JUnit。
- `provided`：**已提供依赖范围**。对于编译和测试时都有效，而运行时无效。典型的如servlet-api，因为容器已提供。
- `runtime`：**运行时依赖范围**。测试和运行时有效，编译主代码时无效。典型为JDBC驱动实现，项目主代码编译只需要JDK提供的JDBC接口即可。
- `system`：**系统依赖范围**，与`provided`依赖范围一致，但使用`system`范围的依赖必须通过systemPath元素显示的指定依赖文件的路径。其与本机系统绑定，应谨慎使用。
- `import`：**导入依赖范围**（2.0.9及以上）。对以上三类classpath不造成影响。

### 3.3 传递依赖

传递依赖性简而言之就是，项目中所需求一个包可能会依赖其他开源类库。如项目生命了对spring-core的依赖，而spring-core对commons-logging的依赖。

当第二直接依赖范围为compile时，传递性依赖范围与第一直接依赖范围一致；当第二直接依赖的范围是test时，依赖不会得到传递；当第二直接依赖的范围是provided时，只传递第一直接依赖范围也为provided的依赖，且传递性依赖范围同样为provided；当第二直接依赖的范围是runtime时，传递性依赖的范围与第一直接依赖的范围一致，但compile例外，此时传递性依赖的范围为runtime。

如果项目依赖于同一个库的多个版本，maven依赖调解的第一原则是：**路径最近者优先**；第二原则为：**第一声明者优先**。

### 3.4 最佳实践

- 排除依赖。若需要排除某一个依赖中的传递依赖，可在该dependency中设置exclusions（每一个exclusions可以包含多个exclusion），并设置排除项的groupId和artifactId。
- 归类依赖。对依赖中相同的部分进行唯一的声明，方便修改和维护。可以在该pom文件中，设置properties标签，并在其内设置以引用名作为标签的子标签，在其他部分引用。如`<properties><version>2.1</versjion></properties>`。
- 优化依赖。maven完成自动解析后的依赖成为已解析依赖，使用`mvn dependency:list`可查看已解析依赖；通过`mvn dependency:tree`分析当前依赖树；通过`mvn dependency:analyze`分析当前项目的依赖，使用时需注意**Used undeclared dependencies**（项目中使用但未进行显式声明的依赖）和**Unused declared dependencies**（项目中未使用，但显式声明的依赖）。对于其中的分析，未显式声明的应显式声明；未使用的应作为参考，不能简单的删除，这是因为`dependency:analyze`只能分析编译主代码和测试代码需要用到的依赖。
