---
layout: post
title:  "Maven 小技巧"
date:   2018-06-05 14:44:30 +0800
categories: server
---

* TOC
{:toc}


## 指定编码，避免警告

```xml
<project>
    <!-- ... -->
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```


## 使用 java 8

```xml
<project >
    <!-- ... -->
    <build>
        <plugins>
            <plugin>
                <!-- using java 8 -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.3</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```


## 在父 pom 中统一版本号

父 pom:

```xml
<project >
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.dada.learning</groupId>
    <artifactId>mvn-test</artifactId>
    <version>1.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>mvn-test-start</module>
        <module>mvn-test-service</module>
        <module>mvn-test-api</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.dada.learning</groupId>
                <artifactId>mvn-test-start</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.dada.learning</groupId>
                <artifactId>mvn-test-api</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.dada.learning</groupId>
                <artifactId>mvn-test-service</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

子 pom:

```xml
<project >
    <parent>
        <artifactId>mvn-test</artifactId>
        <groupId>com.dada.learning</groupId>
        <version>1.0.1-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <modelVersion>4.0.0</modelVersion>
    <!-- 继承父 pom 的 groupId, version, 无需再次指定 -->
    <artifactId>mvn-test-service</artifactId>
    <name>mvn-test-service</name>

    <dependencies>
        <dependency>
            <groupId>com.dada.learning</groupId>
            <artifactId>mvn-test-api</artifactId>
            <!-- 父 pom 中通过 dependencyManagement 设置了 version, 无需再次指定 -->
        </dependency>
    </dependencies>
</project>
```

1. 子模块中设置父 pom 之后，父 pom 的 groupId, version 将被继承。无需在子模块中定义这两个坐标；
2. 父 pom 中通过 dependencyManagement 设置了每个子模块的版本号，这样子 pom 在发生模块间依赖时就无需再指定版本号；
3. relativePath 的作用是设置父 pom 的相对位置，默认值即为 `../pom.xml`, maven 会先根据相对路径找父 pom, 找不到再去本地仓库、远程仓库里找。

上述配置麻烦的地方在于，如果父 pom 中的版本号发生了变化，就需要同步更新子 pom 中的 parent.version, 我还没找到什么技巧可以隐式完成这种更新。
> 仔细想想这点是无法直接做到的，你必须在子 pom 里显式指定父 pom 的版本号，否则子 pom 不知道怎么定位父 pom. 换句话说，父 pom 的坐标是无法被继承的，必须显式指定。

发生上述情况之后，之前的父 pom 因为已经 deploy 到了仓库里，子模块还是可以引用到老的父 pom, 就可能发生问题。

总的来说，应该遵守如下规则:
- 项目中所有模块都应该保持一致的版本号(从父 pom 继承，不应该在子 pom 中手动指定);
- 每当父 pom 的版本号发生变化，就应该更新子 pom 中的 parent.version;

据传 [versions-maven-plugin](http://www.mojohaus.org/versions-maven-plugin/plugin-info.html) 可以较为方便地手动完成版本号的统一更新。这点暂时没空了解，以后有空补上。


## 单独编译子模块

一个多模块的项目中，有时候希望单独编译某个子模块，但是直接 `mvn clean compile` 会报错，因为该模块所依赖的其它模块没有被编译。

所幸 maven 提供了如下几个选项

```
-pl, --projects
        只编译指定模块
-am, --also-make
        编译指定模块，及该模块依赖的其它模块
-amd, --also-make-dependents
        编译指定模块，及依赖该模块的其它模块
```

所以单独编译子模块时，应该根据情况在项目根目录下执行如下命令：

```
# 编译 mvn-test-service 模块，及其所依赖的其它模块
mvn clean compile -pl mvn-test-service -am

# 编译 mvn-test-api 模块，及依赖该模块的其它模块
mvn clean compile -pl mvn-test-api -amd

# 结合使用可能是更常见的情况
mvn clean compile -pl mvn-test-api -am -amd
```