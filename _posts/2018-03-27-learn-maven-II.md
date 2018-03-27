---
layout: post
title: "Maven -- Learn Maven II"
categories: Log4j2
tag: Java-Libs
---
> `《Maven实战》读书记一记 - II`

## 4. 仓库

maven可以在某个位置统一存储所有的maven项目共享的构件，这个位置就是仓库。

### 4.1 仓库的分类

maven仓库分为本地仓库和远程仓库两类。当本地仓库存在需求构件时会直接使用；否则在远程仓库中查找；如果两者都没有则报错。

本地仓库通过在用户目录中的`.m2/repository`中设置的settings.xml，且设置localRepository标签定义本地仓库。若A，B项目均无法从远程仓库获取，且A依赖B，则需要先安装B到本地仓库，再执行构建。

用户只能有一个本地仓库，但可以配置访问多个远程仓库。

maven的安装文件自带了中央仓库的配置，可在`$M2_HOME/lib/maven-model-builder-3.0.jar`中的`org/apache/maven/model/pom-4.0.0.xml`中定义所有maven项目均会继承的超级POM。

### 4.2 配置远程仓库

如果需要配置额外的远程仓库，如JBoss maven仓库，可以在pom中如下设置：

```xml
<project>
    ...
    <repositories>
        <repository>
            <id>jboss</id>
            <name>JBoss Repo</name>
            <url>http://repository.jboss.com</url>
            <releases>
                <enable>true</enable>
                <updatePolicy></checksumPolicy>
                <checksumPolicy></checksumPolicy>
            </releases>
            <snapshots>
                <enable>false</enable>
                <updatePolicy></checksumPolicy>
                <checksumPolicy></checksumPolicy>
            </snapshots>
            <layout>default</layout>
        </repository>
    </repositories>
    ...
</project>
```

在repositories下可以配置多个repository声明多个仓库。其中的id必须是唯一的，避免与默认中心仓库id为central的覆盖。

其中的release和snapshots控制maven对于发布版和快照版构件的下载。元素updatePolicy用来配置maven从远程仓库检查更新的频率，默认为daily，其他值可能为never, always, interval X（每隔X分钟检查一次）；元素checksumPolicy用来配置maven检查校验和文件的策略，默认为warn（构建时输出警告信息），其他可能的值为fail（遇到校验和错误时就让构建失败），ingore（使完全校验和错误）。

### 4.3 快照版本 

快照版本有助于，在项目所依赖的另外一个库处于不稳定开发期时，可以不用重复配置pom文件，就可以在每次构建项目时，maven自动检查获取该快照版本的最新状态。默认情况下，maven一天检查一次更新，可使用`mvn clean install-U`强制更新。

快照版本只应该在组织内部的项目或模块间依赖使用，此时组织内对所有依赖有较深的理解和控制。项目不应当依赖任何外部的快照版本。

### 4.4 镜像库

在settings.xml中设置镜像库，如下所示：

```xml
<settings>
    ...
    <mirrors>
    <mirror>
        <id>maven.net.cn</id>
        <name>ont of central mirrors in China</name>
        <url>http://maven.net.cn/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
    </mirrors>
    ...
</settings>
```

mirrorOf标签指定为了中央仓库的镜像，任何对中央仓库的请求都会转移到该镜像。另外三个元素与一般仓库配置一致。当mirrorOf设置为*时，代表所有的仓库请求都会被转至代理仓库。

## 5. 生命周期

maven的生命周期是抽象的，其实际行为都是由插件完成的，该思想与设计模式中的模板方法较为相似。对所有构建过程进行抽象和统一，这个生命周期包含了**项目清理、初始化、编译、测试、打包、集成测试、验证、部署和站点生成**等几乎所有的构建步骤。每个构建步骤可以绑定一个或多个插件行为，maven为大多数构建步骤编写并绑定了默认插件。

### 5.1 生命周期详解

maven拥有三套相互独立的生命周期：`clean, default, site`。

- clean，生命周期的目的是清理项目。
- default，生命周期的目的是构建项目。
- site，生命周期的目的是建立项目站点。

**clean生命周期**包含三个阶段：

- `pre-clean`：执行一些清理前需要完成的工作。
- `clean`：清理上一次构建生成的文件。
- `post-clean`：执行一些清理后需要完成的工作。

**default生命周期**定义了真正构建所需要执行的所有步骤，是生命周期中最核心的部分，阶段如下：

- `validate`。
- `initialize`。
- `generate-sources`。
- `process-sources`：处理项目主资源文件，一般来说是对`src/main/resources`目录的内容进行变量替换后等工作后，复制到项目输出的主classpath目录中。
- `generate-resources`。
- `process-resources`。
- `compile`：编译项目的主源码，一般来说是编译`src/main/java`目录下的java文件到项目输出的主classpath目录中。
- `process-classes`。
- `generate-test-sources`。
- `process-test-sources`：处理项目测试资源文件。一般来说是对`src/test/resources`目录的内容进行变量替换后等工作后，复制到项目输出的主classpath目录中。
- `generate-test-resources`。
- `process-test-resources`。
- `test-compile`：编译项目的测试源码，一般来说是编译`src/test/java`目录下的java文件到项目输出的测试classpath目录中。
- `process-test-classes`。
- `test`：使用单元测试框架运行测试，测试代码不会被打包或部署。
- `prepare-package`。
- `package`：接受编译好的代码，打包成可发布的格式，如jar。
- `pre-integration-test`。
- `integration-test`。
- `post-integration-test`。
- `verify`。
- `install`：将包安装到maven本地仓库，供本地maven项目使用。
- `deploy`：将最终的包复制到远程仓库，供其他开发人员和maven项目使用。

**site生命周期**包含以下阶段：

- `pre-site`：执行一些在生成项目站点之前需要完成的工作。
- `site`：生成项目站点文档。
- `post-site`：执行一些在生成项目站点之后需要完成的工作。
- `site-deploy`：将生成的项目站点发布到服务器上。

**命令行与生命周期**，命令行执行maven任务最主要的方式就是调用maven生命周期函数。各个生命周期是相互独立的，而生命周期的阶段是有前后依赖关系的。

- `mvn clean`：调用clean生命周期的clean阶段。实际执行阶段为clean生命周期的pre-clean和clean阶段。
- `mvn test`：调用default生命周期的test阶段。实际执行阶段为default生命周期的validate, initialize等直到test的所有阶段。
- `mvn clean install`：调用clean生命周期的pre-clean和clean阶段，以及default生命周期从validate到install的所有阶段。
- `mvn clean deploy site-deploy`：调用clean生命周期的clean阶段、default生命周期的deploy阶段，以及site生命周期的site-deploy阶段，该命令结合了maven所有三个生命周期。

### 5.2 插件

一个插件具有多项功能，每一个功能就是一个插件目标。写法为`pre:post`格式。如compile:compile是maven-compile-plugin的compile目标。

maven的生命周期阶段与插件的目标互相绑定，以完成某个具体的构建任务。如编译这一任务对应了default生命周期的compile阶段，而maven-compile-plugin的compile目标可完成该任务。

maven的**内置绑定**如下图所示：

clean, site生命周期阶段与插件目标默认绑定关系如下图所示：
![clean site](/assets/img/maven/20180327_maven_clean_site.png)

default生命周期阶段与插件目标默认绑定关系如下图所示（打包为jar）：
![clean site](/assets/img/maven/20180327_maven_default.png)

maven可进行**自定义绑定**，常见的例子是创建源码jar包，需要用户配置maven-source-plugin并使用jar-no-fork目标完成打包，配置示例如下：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-source-plugin</artifactId>
            <version>2.1.1</version>
            <executions>
                <execution>
                    <id>attach-sources</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>jar-no-fork</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

配置中每个executions下的每个execution子元素可配置一个执行任务，示例中配置了一个id为attach-sources的任务并通过phase配置绑定到verify生命周期阶段上，再通过goals配置要执行的插件目标。如果不绑定phase则拥有默认绑定阶段。

### 5.3 插件配置

maven命令中使用-D参数并伴随一个参数键=参数值来配置插件目标的参数，如`mvn install -Dmaven.test.skip=true`。参数`-D`是Java自带的，其功能是通过命令行设置一个java系统属性。

很少改变的全局参数、插件任务的特定参数在pom文件中配置。

使用maven-help-plugin获取插件详情，如`mvn help:describe -Dplugin=org.apache.maven.plugins:maven-compiler-plugin:2.1`获取maven-compiler-plugin2.1版本的信息。

maven插件仓库的配置使用pluginRepositories和pluginRepository标签配置插件，其余子元素的含义与依赖远程仓库完全一样。

## 6. 聚合与继承

### 6.1 聚合

通过maven聚合，将多个maven模块统一构建。创建另外一个maven项目，示例如下所示：

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>cc.joyjon.mvn</groupId>
    <artifactId>combine</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>Combine</name>
    <modules>
        <module>part1</module>
        <module>part2</module>
    </modules>
</project>
```

适用于其他模块相同的groupId, version。使用**packaging为pom**。modules中定义了聚合的模块，每一个module的值都是一个当前pom的相对目录。通常将聚合模块放在项目目录的最顶层，其他模块作为聚合模块的子目录存在。

### 6.2 继承