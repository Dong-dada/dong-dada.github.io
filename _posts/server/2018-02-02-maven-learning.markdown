---
layout: post
title:  "Maven 基础知识"
date:   2018-02-02 14:30:30 +0800
categories: server
---

* TOC
{:toc}


## Maven 是什么

Maven 是一个构建工具，能够帮我们完成从清理、编译、测试到生成报告、再到打包、部署的一系列动作。而无需一遍一遍地输入各种命令。看起来有点儿像 Chromium 所使用的 GN 构建工具。

Maven 还是一个依赖管理工具，一般的 Java 应用都会借助许多第三方开源类库，这些类库都可以通过依赖的方式导入到项目中来。随着依赖项的增多，版本不一致、版本冲突、依赖臃肿等问题就会接踵而来。Maven 提供了一些机制可以让我们有序管理依赖，轻松解决依赖问题。

Maven 还是一个项目信息管理工具，可以帮我们找到分散在项目中各个角落的项目信息，包括项目描述、开发者列表、版本控制系统地址、许可证、缺陷管理系统地址等。通过 Maven 自动生成的站点，以及一些现有的插件，我们还能够轻松获得项目文档、测试报告、静态分析报告、源码版本日志报告等项目信息。

Maven 还为 Java 开发者提供了一个免费的中央仓库，在其中几乎可以找到任何流行的开源类库。


## Maven 使用入门

这一节介绍一个用 Maven 进行构建的简单例子。

### pom.xml

就如同 Makefile 之于 Make, build.xml 之于 Ant, BUILD.gn 至于 GN 一样。Maven 也需要有一个文件来定义项目的基本信息，这个文件就是 pom.xml, 其中 pom 是 Project Object Model 项目对象模型 的意思。

先建立一个 hello-world 文件夹，然后在文件夹下新增一个 pom.xml 文件，其内容如下：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.dada.mvn</groupId>
  <artifactId>hello-world</artifactId>
  <version>1.0-SNAPSHOT</version>

  <name>My Hello World Project</name>

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

project 根元素中的属性不是必须的，其作用是让第三方工具更容易识别。重点看下 project 的子元素：
- modelVersion : POM 模型的版本，对于 maven2 和 maven 3 来说，它必须是 4.0.0；
- groupId : 指明该项目属于哪个组，这个组往往是指所在的组织或者公司。假如我是谷歌的员工(可惜不是)，在 chrome 这个项目组里，那么 groupId 就可以是 com.google.chrome.
- artifactId : 定义了在当前组中的唯一 ID, 比如在给 chrome 开发书签相关的功能，那么 artifactId 就可以是 bookmark.
- version : 指定当前书签项目的版本，1.0-SNAPSHOT 表示该项目还处于开发中，是不稳定的版本，随着项目的继续，这个版本号也会升级。
- name : 不是必须的元素，提供给用户一个友好的项目名称；
- dependencies : 类似于 GN 中的 deps, 指明了 hello-world 这个项目所依赖的其他项目，这里我们依赖了 junit 这个测试框架。

上述内容中最重要的是 groupId, artifactId, version 这三个元素，它们定义了一个项目的坐标，在 Maven 的世界中，任何 jar, pom, war 都是基于这些坐标来进行区分的。

### 编写主代码

默认情况下，Maven 假设项目代码位于 `hello-world/src/main/java` 目录，我们遵循这一约定，按照包名在 `hello-world/src/main/java/com/dada/mvn/helloworld` 目录下创建一个 HelloWorld.java 文件：

```java
package com.dada.mvn.helloworld;

public class HelloWorld {
    public String sayHello() {
        return "Hello Maven";
    }

    public static void main(String[] args) {
        System.out.println(new HelloWorld().sayHello());
    }
}
```

接着，在 hello-world 目录执行命令行 `mvn clean compile`，使用 Maven 来编译上述代码，如无意外，将在 `hello-world/target/classes/com/dada/mvn/helloworld` 目录下生成编译后的 HelloWorld.class 目标文件。

### 编写测试代码

接下来，我们为上述示例创建一个测试代码，用来演示 Maven 的测试功能。

Maven 假设测试代码位于 `hello-world/src/test/java` 目录，我们遵循这一约定，按照包名在 `hello-world/src/test/java/com/dada/mvn/helloworld` 目录下创建一个 HelloWorldTest.java 文件：

```java
package com.dada.mvn.helloworld;

import static org.junit.Assert.assertEquals;
import org.junit.Test;

public class HelloWorldTest
{
    @Test
    public void testSayHello() {
        HelloWorld helloWorld = new HelloWorld();
        String result = helloWorld.sayHello();

        assertEquals("Hello Maven", result);
    }
}
```

接着，在 hello-world 目录执行命令行 `mvn clean test`，使用 Maven 来执行上述测试代码，如无意外可以看到测试通过的输出。

因为我们之前在 pom.xml 里配置了对 junit 的依赖，因此 Maven 会自动搜索并下载正确版本的 junit, 从而完成测试。

### 打包和运行

直接在 `hello-world` 目录下执行命令行 `mvn clean package`, 即可打包该项目，如无意外可以在 `hello-world/target` 目录下看到一个 `hello-world-1.0-SNAPSHOT.jar` 文件。

有了 jar 包，就可以把它复制到其它项目的 ClassPath 来给其他项目使用了。如果希望其它 Maven 项目能够直接搜索并下载到它，就可以使用 `mvn clean install` 命令行，将这个 jar 安装到 Maven 的 **本地仓库** 中(只是在本地可以用 Maven 来搜索使用，还没有传到远程仓库)。

使用 `java -jar hello-world-1.0-SNAPSHOT.jar` 尝试运行这个 jar 包，会出现 `hello-world-1.0-SNAPSHOT.jar中没有主清单属性` 这样的提示，其原因是因为我们没有指定 main 方法所在的类，解压 jar 包，查看其中的 MANIFEST.MF 文件，可以看到其中并没有 Main-Class 信息。

要指定 Main-Class, 需要在 pom.xml 里添加如下内容：

```xml
<project>
  <!-- ... -->

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>1.2.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.dada.mvn.helloworld.HelloWorld</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

在 pom.xml 中新增了一个 maven-shade-plugin 插件，它会按照配置来修改 jar 包中的内容。

再次运行 `mvn clean package` 命令行，如无意外将生成一个新的 jar 包，并且可以用 `java -jar` 命令来运行。

### archetype 生成项目骨架

每次新建项目都手动编写 pom.xml, 手动建立目录结构，太麻烦了，因此 Maven 提供了一个命令可以简化上述流程。

```
mvn archetype:generate
```

随后会以交互式的方式来让你指定 groupId, artifactId, version, 包名，随后按照这些信息来生成你需要的骨架。

archetype 命令中只提供了几种默认的选项，我们可以根据实际情况来编写自己的 archetype, 实际的开发中项目组内可能已经有自己的一套 archetype 供我们使用。

### 小结

这一节我们用一个简单的 Hello World 例子介绍了 Maven 的一些基础知识点。
- Maven 使用 pom.xml 来描述项目；
- Maven 使用 groupId, artifactId, version 作为一个项目的坐标，来标识这个项目；
- 可以在 pom.xml 中对项目的依赖进行配置，Maven 会根据依赖配置从远程仓库中下载对应的 jar 包；
- 可以在 pom.xml 中配置一些插件，对构建过程进行控制，比如利用 maven-shade-plugin 来设置 Main-Class.

这一节用到了如下几个命令：
- mvn clean compile
- mvn clean test
- mvn clean package
- mvn clean install
- mvn archetype:generate


## 坐标和依赖

### 坐标详解

之前已经介绍过坐标的概念，Maven 的坐标是由一系列元素定义的，它们是 groupId, artifactId, version, packaging, classifier. 例如：

```xml
<groupId>org.sonatype.nexus</groupId>
<artifactId>nexus-indexer</artifactId>
<version>2.0.0</version>
<packaging>jar</packaging>
```

- groupId: 定义当前 Maven 项目隶属的实际项目。需要注意的是，groupId 不应该直接对应于项目隶属的公司。因为 artifactId 只能定义 Maven 项目(模块)，如果 groupId 定义为公司的话，artifactId 就不好定义了。上述例子中，用反向域名的方式定义了 nexus 这一实际项目，而 nexus-indexer 是这个项目中的某个模块；
- artifactId: 定义实际项目中的一个 Maven 项目(模块)，推荐用实际项目名作为前缀，比如 nexus 是实际项目，则定义 nexus-indexer 为 artifactId;
- version : 定义该 Maven 项目的版本；
- packaging : 定义该 Maven 项目的打包方式。默认为 jar, 也可以按需求改为 war;
- classifier : 用来帮助定义构建输出的一些附属构件，不能直接定义它，因为附属构建是通过插件来构建的。比如主构建是 nexus-indexer-2.0.0.jar, 那么它可能会使用一些插件生成如 nexus-indexer-2.0.0-javadoc.jar, nexus-indexer-2.0.0-sources.jar 等附属构件。

Maven 打包后生成的文件名与坐标是对应的，规则为 `artifactId-version[-classifier].packaging`, 比如 `nexus-indexer-2.0.0-javadoc.jar`。

### 依赖的配置

正如之前介绍的，你需要在 pom.xml 中增加 dependency 元素来描述该 Maven 项目的依赖。

一个依赖声明可以包含如下元素：

```xml
<project>
    <dependencies>
        <dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>...</version>
            <type>...</type>
            <scope>...</scope>
            <optional>...</optional>
            <exclusions>
                <exclusion>
                ...
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
</project>
```

- groupId, artifactId, version : 指明所依赖的 Maven 项目的坐标；
- type : 依赖的类型，对应于 packaging;
- scope : 依赖的范围，稍后介绍；
- optional : 标记依赖是否可选，稍后介绍；
- exclusions : 用来排除传递性依赖，稍后介绍；

### 依赖范围

在一开始的 hello-world 项目中，我们为 junit 的依赖声明了 `<scope>test</scope>` 元素。它表示 junit 只用于 test, 在 compile, package 等其它条件下不依赖 junit.

scope 这个元素的作用就在于此，你可以用它来控制依赖的范围。 Maven 有以下几种依赖范围：
- compile : 使用该范围的话，在编译、测试、运行期间都会引入该项目。这是默认的依赖范围；
- test : 使用该范围的话，只有在测试期间才会引入该项目；
- provided : 表示已经提供了该依赖，因此只有在编译、测试期间会引入该项目，在运行时不会引入。比如 servlet-api.jar, 运行时 tomcat 已经提供了该项目，因此不需要重复引入；
- runtime : 只有在 测试、运行 期间会引入该项目，在编译期间不会引入。比如 JDBC 驱动实现，在编译期间只需要使用 JDBC 接口就行，因此不需要在编译期引入它；
- system : 类似 provided, 但需要通过 systemPath 元素来显式指定依赖文件的路径，应该谨慎使用，因为它与本地路径所绑定；
- import : 导入依赖范围，以后介绍吧。

下面用图表的方式来描述上述依赖范围：

```
            | compile |  test   | runtime | Sample      |
 scope      |         |         |         |             |
------------+---------+---------+---------+-------------+
compile     |    +    |    +    |    +    | spring-core |
------------+---------+---------+---------+-------------+
test        |         |    +    |         | junit       |
------------+---------+---------+---------+-------------+
provided    |    +    |    +    |         | servlet-api |
------------+---------+---------+---------+-------------+
runtime     |         |    +    |    +    | JDBC driver |
------------+---------+---------+---------+-------------+
system      |    +    |    +    |         | local jar   |
```

### 传递性依赖

传递性依赖很好理解，就是 A 依赖于 B, B 依赖于 C, 那么在使用 Maven 构建 A 的过程中，Maven 会帮你引入 B, 随后根据 B 的依赖引入 C.

传递性依赖和依赖范围也有关系，比如 A 以 test 方式依赖于 B, B 以 compile 方式依赖于 C, 那么只有在 test 期间 A 才会依赖于 C，有点儿取交集的意思。

### 依赖调解

有了传递性依赖之后，就可能引入另一个问题。比如项目 A 有两条依赖关系：

```
A -> B -> C -> X(1.0)

A -> D -> X(2.0)
```

A 通过 B 和 D 这两条依赖，分别依赖了 X 的 1.0 和 2.0 版本，这时候应该以哪个版本为准呢？

Maven 定义了依赖调解的第一原则：路径最近者优先。对于上述例子，因为 X(2.0) 的路径更短，因此引入 X(2.0) 而不是 X(1.0)；

如果路径长度一样，Maven 还定义了依赖调解的第二原则：第一声明者优先。也就是在 POM 中先声明的那个为优先选择；

### 可选依赖

可选依赖的意思可能不是那么直接。比如以下依赖关系：

```java
A --> B --> mysql-connector(可选依赖)
      |
      +---> postgresql(可选依赖)
```

B 分别使用 MySQL 和 PostgreSQL 实现了两个互斥的特性，用户只会使用其中一种。比如 B 是一个持久层隔离工具包，它支持多种数据库。在构建 B 这个项目的时候，就需要这两种数据库的驱动程序，但实际使用的时候，只会依赖一种数据库。

这时候，就可以在 B 的 pom.xml 里，把对这两种数据库驱动的依赖设定为可选依赖。意思是由使用方 A 来决定使用哪种数据库：

```xml
<!-- B 项目的 pom.xml -->
<project>
    <<artifactId>project-b</artifactId>

    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.10</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>8.4-701.jdbc3</version>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>

<!-- A 项目的 pom.xml -->
<project>
    <artifactId>project-a</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.dada.mvn</groupId>
            <artifactId>project-b</artifactId>
            <version>1.0.0</version>
        </dependency>
        <!-- 决定使用哪个数据库 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.10</version>
        </dependency>
    </dependencies>
</project>
```

### 排除依赖

有时候，你的项目 A 依赖了另一个第三方的项目 B，而 B 又依赖了一个不稳定的 C(0.1). 这个时候你会希望把 C(0.1) 去掉，改为依赖稳定版本的 C(1.0)。

这种情况下，就可以先排除对 C(1.0) 的依赖，然后重新声明对 C(1.0) 的依赖：

```xml
<project>
    <groupId>com.dada.mvn</groupId>
    <artifactId>project-a</artifactId>
    <version>1.0.0</version>

    <dependencies>
        <dependency>
            <groupId>com.dada.mvn</groupId>
            <artifactId>project-b</artifactId>
            <version>1.0.0</version>
            <exclusions>
                <!-- 排除 B 对原有的 C 的依赖 -->
                <exclusion>
                    <groupId>com.dada.mvn</groupId>
                    <artifactId>project-c</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- 重新声明对 C(1.0) 的依赖 -->
        <dependency>
            <groupId>com.dada.mvn</groupId>
            <artifactId>project-c</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
</project>
```

### 使用 Maven 属性归类依赖

在 Maven 中可以定义属性，然后引用属性。有点儿类似于在 C 语言中定义一个宏 PI, 用它来替换实际的值 3.1415926, 方便阅读的同时，减少维护的麻烦。

比如 springframework 有许多依赖，版本都是一样的，我们就可以用属性来对这些版本号做归类：

```xml
<project>
    ...

    <properties>
        <!-- 定义一个 springframework 版本号的属性 -->
        <springframework.version>2.5.6</springframework.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <!-- 使用属性来代表版本号 -->
            <version>${springframework.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>sprinf-beans</artifactId>
            <!-- 使用属性来代表版本号 -->
            <version>${springframework.version}</version>
        </dependency>
        ...
    </dependencies>
</project>
```

### 优化依赖

这一小节介绍几个命令，可以来查看项目间的依赖关系：

- `mvn dependency:list` : 列出已解析依赖；
- `mvn dependency:tree` : 以树形结构列出已解析依赖；
- `mvn dependency:analyze` : 分析当前项目的依赖是否存在潜在问题；

### 小结

这一节首先介绍了 Maven 中坐标的组成，然后介绍了 Maven 中依赖的细节。

- 除了 groupId, artifactId, version 之外，还有一个 packaging 也是坐标之一，它标识了这个项目的打包方式，默认是 jar;
- 依赖的配置中, scope 指明了依赖的范围，这决定了这个依赖会在 编译、测试、打包的哪个环节被引入；
- 依赖的配置中，optional 指明了这个依赖是可选的，需要由上层决定具体使用哪个依赖；
- 依赖的配置中，exclusions 可以帮你排除一些不想依赖的项目，并将其替换为更合适的版本；

另外，我们还介绍了 Maven 调解依赖的两个原则：
- 路径最近者优先。当依赖树中存在两个相同项目时，引入路径比较近的那个项目；
- 第一声明者优先。如果路径一样，那么引入先被声明的那个项目；

另外，我们简单介绍了 Maven 中的属性，你可以定义一个属性，接着去引用它。

最后，我们介绍了几个命令来帮助你优化依赖关系：
- mvn dependency:list
- mvn dependency:tree
- mvn dependency:analyze


## 仓库

Maven 在解析 POM 中的依赖的时候，会根据依赖项的坐标去本地仓库里寻找对应的项目，如果本地仓库里没有，就从远程仓库里下载该项目。这是 Maven 工作的基本原理。

从上述原理可以看到，Maven 没有把依赖项目存储到每个项目里，而是有一个仓库来存储所有这些项目。

### 仓库的分类

之前已经说过，仓库分为本地和远程两类。对于远程仓库而言，存在一些特殊目的仓库：
- 中央仓库： Maven 核心自带的远程仓库；
- 私服： 在局域网内自行架设的远程仓库；
- 一些公共仓库： 比如 Java.net Maven 仓库、JBoss Maven 库；

私服和中央仓库的区别有点儿类似于 公司内自己架设的 Git 仓库 和 GitHub 的区别。前者只给内部使用，后者是所有人都可以用的。

### 仓库的布局

Maven 仓库基于简单的文件系统存储，它的文件布局跟项目的坐标是对应的。

比如一个项目的坐标为：

```xml
<groupId>com.dada.mvn</groupId>
<artifactId>hello-world</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>jar</packaging>
```

那么该项目就会保存在仓库的 `仓库目录\com\dada\mvn\hello-world\1.0-SNAPSHOT` 文件夹中。

本地仓库目录可以在 Maven 的 settings.xml 里配置，这里就不罗嗦了。

Maven 会在上述文件夹中保存已经打好的 jar 包，以及该项目的 POM 文件。你可以尝试运行 `mvn clean install` 命令，把自己的项目安装到本地仓库里，然后检查下 Maven 仓库里是否按照上述规则保存了你的项目。

### 远程仓库的配置

默认情况下，Maven 会从中央仓库中下载所需要的构件。很多时候这个无法满足你的需求，可能某个构件存在于 JBoss 仓库里，这时候你可以在 POM 中加入 JBoss 仓库的配置：

```xml
<project>
    <repositories>
        <repository>
            <id>jboss</id>
            <name>JBoss Repository</name>
            <url>http://repository.jboss.com/maven2/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
            <layout>default</layout>
        </repository>
    </repositories>
</project>
```

### 远程仓库的认证

大部分私服为了安全性的考虑，不会让你直接连上它，需要提供用户名密码来访问。你可以在 Maven 的 settings.xml 里配置认证信息：

```xml
<settings>
    <servers>
        <id>repository-id</id>
        <username>repo-user</username>
        <password>repo_pwd</password>
    </servers>
</settings>
```

### 部署到远程仓库

除了把自己的构件安装到 Maven 本地仓库，你还可以把它部署到远程仓库里，供其它人使用。

首先需要在 POM 中配置构件的部署地址：

```xml
<project>
    ...

    <distributionManagement>
        <repository>
            <id>proj-releases</id>
            <name>Proj Release Repository</name>
            <url>http://192.168.1.100/content/repositories/proj-release</url>
        </repository>
        <snapshotRepository>
            <id>proj-snapshots</id>
            <name>Proj Snapshot Repository</name>
            <url>http://192.168.1.100/content/repositories/proj-snapshots</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```

配置完毕后，运行 `mvn clean deploy` 命令，既可把该项目部署到对应的远程仓库中。

### 快照机制

我们之前开发的 hello-world 例子中，其版本号是 `1.0.0-SNAPSHOT`, 这表示该构件是 `1.0.0` 的快照版本。

Maven 的快照机制可以方便协作开发。比如 A 和 B 两个模块都在开发，A 依赖了 B, 那么每当 B 有进一步的改动之后，A 都需要获取到 B 的最新版本。

每次 B 有所修改，Maven 都会为构件打上时间戳，在 Maven 构件 A 的时候，就只会拉取最新版本的 B。

默认情况下 Maven 一天检查一次远程仓库中的快照文件有没有更新，如果有，就把它拉取到本地仓库。如果你希望强制拉取，那么可以执行 `mvn clean install-U`。

### 仓库搜索服务

还有一个问题是，我们怎么从仓库中找到需要的构件，比如需要一个负责登录的构件，怎么知道远程仓库里有没有这个构件，这个构件的坐标是什么。

这就需要有人 Maven 仓库搜索服务。

知名的公共仓库搜索服务有 [Sonatype Nexus](http://repository.sonatype.org), [Jarvana](http://www.jarvana.com/jarvana), [MVNbrowser](http://www.mvnbrowser.com), [MVNrepository](http://mvnrepository.com)

公司内部也许也会有这样的服务，方便你查询公司内部的 Maven 私服。

### 小结

这一节介绍了 Maven 中仓库的一些细节：
- 仓库的布局是简单的文件系统布局，构件路径跟项目的坐标相对应；
- 除了 Maven 自己的中央仓库，公司内部还可以搭建自己的私服，也有其它一些公共的 Maven 远程仓库；
- 快照机制可以帮助我们在协作开发时，更方便地与他人共享最新版本的构件；
- 可以利用仓库搜索服务来查询远程仓库中的构件信息；


## 生命周期和插件

Maven 用生命周期这一概念来抽象构件过程中的各个阶段。每个阶段有不同的插件来完成实际的构建过程。

### 三套生命周期

Maven 有三套独立的生命周期，分别是 clean, default, site. 每套生命周期中都包含了若干个阶段 (phase), 这些阶段是由顺序的，后面的阶段依赖于前面的阶段。

你可以 Maven 中直接执行某个阶段，比如 `mvn pre-clean`, 那么只会执行 clean 生命周期的 `pre-clean` 阶段。如果你执行了后面的阶段，比如 `mvn post-clean`, 那么前面的阶段也会被执行。

Maven 命令中的参数代表了所需要执行的阶段而不是生命周期。比如 `mvn clean` 不是说要执行 clean 生命周期的所有阶段，而是执行 clean 生命周期的 clean 阶段。类似的, `mvn clean install` 意思是先执行 clean 生命周期的 clean 阶段，再执行 default 生命周期的 install 阶段。类似的，你还可以调用 `mvn clean install site-deploy` 继续让它执行 site 生命周期的 site-deploy 阶段。

clean 生命周期用于清理上次构建生成的文件，它分为如下几个阶段：
- pre-clean : 准备清理；
- clean : 进行清理；
- post-clean : 扫尾工作；

default 生命周期用于构建项目，它是所有生命周期中最核心的部分，它分为如下几个阶段：
- validate
- initialize
- generate-sources
- process-sources : 处理项目主资源文件。一般是把 `src/main/resources` 目录的内容进行变量替换等工作后，复制到项目输出的主 classpath 目录中。
- generate-resources
- process-resources
- compile : 编译项目的主源码。一般是把 `src/main/java` 目录的代码编译到项目输出的主 classpath 目录中。
- process-classes
- generate-test-sources
- process-test-sources : 类似于 process-sources, 但它处理的是测试资源文件
- generate-test-sources
- test-compile : 类似于 compile, 但它编译的是测试代码；
- process-test-classes
- test : 使用单元测试框架运行测试，该代码不会被打包和部署。
- prepare-package
- package : 对编译完成的代码进行打包。
- pre-integration-test
- integration-test
- post-integration-test
- verify
- install : 将包安装到本地 Maven 仓库，共本地其它 Maven 项目使用
- deploy : 将最终的包复制到远程仓库，共其它开发人员使用。

site 生命周期用于建立和发布项目站点，Maven 可以根据 POM 所包含的信息自动生成一个站点，方便团队交流和发布项目信息。它分为如下几个阶段：
- pre-site
- site : 生成项目站点文档
- post-site
- site-deploy : 将生成的项目站点发布到服务器上


