---
layout: post
title:  "Servlet 基础知识"
date:   2018-03-14 11:15:30 +0800
categories: server
---

* TOC
{:toc}


最近在看 《Spring 实战》 看到 Spring MVC 的部分有点看不懂了，需要学习一下 Servlet 相关知识，所以找了本 《Head First Servlet & JSP》 来看。

## 简介

简单来说 Servlet & JSP 是用来做网站(生成网页)的一种技术\框架。类似的框架还有 CGI(Common Gateway Interface, 公共网关接口)。

先来看一张图，了解 Servlet 在 Web 应用中的位置：

![]( {{site.url}}/asset/servlet-architecture.jpg )

可以看到 Servlet 处于 HTTP Server 和 Database 中间，这种说法不一定准确，可能在 Database 和 Servlet 之间还有一些别的服务之类的。重要的是，HTTP Server 在收到浏览器发来的 HTTP 请求后，会交给 Servlet, 由 Servlet 决定如何响应这个请求，它可能会查询数据库，填充一些值之后，返回给浏览器一个网页；或者不查数据库，直接找到一个静态页面返回给浏览器。

另外 HTTP Server 是什么我还不太确定，我认为它会监听本地的端口(比如 80, 443), 收到 HTTP 请求之后，通过接口之类的方式，把这些请求发送给 Servlet。换句话说 Servlet 无法独立工作，得寄宿在 HTTP Server 里才行，因此也有地方把它叫做 web 容器。常见的 容器是 Tomcat.


## 例子

先来看一个 Servlet 的具体例子。这个例子需要用 Maven 来构建，用 Tomcat 作为 Web 容器，因此在编码之前需要先安装好这两个程序。

以下是项目目录结构：

```
.
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── dada
        │           └── learning
        │               └── SimpleServlet.java
        └── webapp
            └── WEB-INF
                └── web.xml
```

以下是 pom.xml 里的内容：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.dada.learning</groupId>
    <artifactId>SimpleServlet</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <name>SimpleServlet</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- 引入 servlet 依赖，因为 tomcat 已经提供了 servlet 的 jar 包，所以 scope 设为 provided -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <!-- 利用 maven 插件来打 war 包 -->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <warSourceDirectory>src/main/webapp</warSourceDirectory>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

以下是 SimpleServlet.java 文件里的内容：

```java
package com.dada.learning;

import java.io.IOException;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class SimpleServlet extends HttpServlet {

    private static final long serialVersionUID = -4751096228274971485L;

    @Override
    protected void doGet(HttpServletRequest reqest, HttpServletResponse response) throws ServletException, IOException {
        response.getWriter().println("Hello World!");
    }
    
    @Override
    public void init() throws ServletException {
        System.out.println("Servlet " + this.getServletName() + " has started");
    }

    @Override
    public void destroy() {
        System.out.println("Servlet " + this.getServletName() + " has stopped");
    }
    
}
```

以下是 web.xml 里面的内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>

<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" 
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
    version="3.1">
    
    <!-- 给 Tomcat Manager 展示用的名字 -->
    <display-name>Simple Servlet Application</display-name>

    <!-- 配置 servlet -->
    <servlet>
        <servlet-name>simpleServlet</servlet-name>
        <servlet-class>com.dada.learning.SimpleServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    
    <!-- /hello 这条请求将被发送给 simpleServlet 这个 servlet 来处理 -->
    <servlet-mapping>
        <servlet-name>simpleServlet</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>

</web-app>
```

运行 `mvn clean package`, 即会在 target 目录生成一个名为 SimpleServlet-1.0.0.war 的 WAR 包。将其改名为 `SimpleServlet.war`, 放到 Tomcat 的 webapps 目录，启动 Tomcat, 最后访问 http://localhost:8080/SimpleServlet/hello (注意大小写)即可查看效果。

上述过程中，把 `SimpleServlet.war` 放到 Tomcat webapps 目录的过程，也就是发布 war 包到 tomcat 服务器的过程，看起来非常简单。

上述内容中提到的 WAR 包，可以看到 WAR 包和 JAR 包之间一个比较明显的区别是，WAR 包里多了一个 web.xml 文件，这个文件对 servlet 进行了配置。不知道 WAR 包还有没有什么别的意义，现阶段可以认为它是配置 Servlet 用的。