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

