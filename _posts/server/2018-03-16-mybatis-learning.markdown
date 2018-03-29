---
layout: post
title:  "MyBatis 基础知识"
date:   2018-03-16 15:54:30 +0800
categories: server
---

* TOC
{:toc}


## 简介

MyBatis 是一个 ORM(Object Relational Mapping, 对象关系映射) 框架。与其他 ORM 框架不同的是，MyBatis 并没有把 Java 对象与数据库表关联起来，而是将 Java 方法与 SQL 语句关联。


## 简单使用

先看一个简单的 MyBatis 的例子，在这之前你需要完成 MySQL 的安装。

新建一个 Maven 项目，在 POM 中导入 MyBatis 的 jar 包：

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>3.3.0</version>
</dependency>
```

准备一个数据库名为 mybatis, 然后创建一个表 country, 插入一些测试数据:

```sql
use mybatis;

CREATE TABLE 'country' {
    'id' int NOT NULL AUTO_INCREMENT,
    'name' varchar(255) NULL,
    'code' varchar(255) NULL,
    PRIMARY KEY ('id')
};

INSERT INTO country ('name', 'code') VALUES ('中国', 'CN'), ('美国', 'US'), ('俄罗斯', 'RU'), ('英国', 'GB'), ('法国', 'FR');
```

### MyBatis 配置文件

在 `src/main/resources` 下创建一个资源文件 `mybatis-config.xml`, 用于对 MyBatis 进行配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 设置 MyBatis 环境 -->
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="" value="" />
            </transactionManager>

            <!-- 配置 mysql 连接 -->
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://localhost:3306/test" />
                <property name="username" value="root" />
                <property name="password" value="password" />
            </dataSource>
        </environment>
    </environments>

    <!-- 设置 mapper 文件 -->
    <mappers>
        <mapper resource="com/dada/learning/mybatis/mapper/CountryMapper.xml" />
    </mappers>
</configuration>
```

### Mapper.xml 文件

配置 MyBatis 环境这一点很好理解，就是告诉 MyBatis 要连接哪个数据库，用户名密码是什么。比较难理解的是这里的 mapper 文件。我们来看一下 CountryMapper.xml 这个文件里有什么：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.dada.learning.mybatis.mapper.CountryMapper">
    <select id="selectAll" resultType="com.dada.learning.mybatis.model.Country" >
        select id, name, code from country
    </select>
</mapper>
```

可以看到 mapper 文件里把 sql 语句转化成了一个类似于方法描述的东西，先执行 sql 语句，然后发返回值存储到 `com.dada.learning.mybatis.model.Country` 这个对象里。

### Model 对象

`Country` 对象非常简单：

```java
public class Country {
    private Long id;
    private String name;
    private String code;

    // getter, setter 略...
}
```

唯一需要注意的是，不要使用基本类型，因为数据库表可能会有 NULL 值，如果是基本类型的话无法表示。

### 测试一下

到这里，MyBatis 要干的事儿就完成了，我们编写一个测试用例来调用一下：

```java
public class CountryMapperTest {
    private static SqlSessionFactory sqlSessionFactory;

    @BeforeClass
    public static void init() {
        try {
            // 将 mybatis-config.xml 加载为 sqlSessionFactory
            Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            reader.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Test
    public void testSelectAll() {
        // 创建一个 SqlSession 对象
        SqlSession sqlSession = sqlSessionFactory.openSession();

        // 调用 CountryMapper.xml 里的方法
        List<Country> countryList = sqlSession.selectList("selectAll");
        
        // 打印查询的结果
        printCountryList(countryList);
    }

    private void printCountryList(List<Country> countryList) {
        for (Country country : countryList) {
            System.out.printf("%-4d%4s%4s\n", country.getId(), country.getCountryName(), country.getCountryCode());
        }
    }
}
```

### 小结

可以看到，使用 MyBatis 访问数据库相对比较简单：
- 创建 MyBatis 配置文件，描述如何连到数据库、配置 mapper;
- 创建 mapper 文件，把 sql 语句转化为用 xml 形式描述的一个 Java 方法；
- 创建 SqlSession 对象来调用 mapper 文件中定义的 sql 语句：
    + 使用 MyBatis 配置文件来创建 SqlSessionFactory;
    + 调用 SqlSessionFactory.openSession() 来获取一个 SqlSession 对象；
    + 调用 `SqlSession.selectList("selectAll")` 等方法来调用 mapper 中定义的 sql 语句；

上述例子中的 `Country` 对象不是必须的，如果 mapper 中定义的 sql 语句只是返回 String, 那么就无需 model, 不过大部分情况下还是需要用到 model 的，毕竟数据库表的设计是结合业务来的，往往需要一个 model 类来代表业务数据。

上述例子中有一个小细节，注意到在 CountryMapper.xml 里指定 resultType 的时候，直接输入了 `Country` 类。这是 MyBatis 自动帮你做了数据库表和 Country 类的映射，能这么做的原因是数据库的列名 `name`, `code` 和 `Country` 类的成员名一一对应，因此 MyBatis 可以做这种映射，如果不是完全一致的话，就需要手动配置数据库列名和 model 类成员之间的映射关系了。

从上述过程可以看出来，仅需要一个 mapper 文件，MyBatis 就能够帮你把 sql 语句转换为对应的 Java 调用。


## 使用接口来调用方法

上一节中调用 sql 语句，使用的是 `SqlSession.selectList("selectAll")` 这样的形式，MyBatis 3.0 开始，支持把 mapper 文件映射到一个接口上，你可以通过接口来调用 mapper 中定义的 sql 语句，更简单直接一些。

注意之前 mapper 中有一个 namespace 属性：

```xml
<mapper namespace="com.dada.learning.mybatis.mapper.CountryMapper">
    <!-- ... -->
</mapper>
```

上一节的例子中没有提供接口定义，你可以创建一个接口：

```java
package com.dada.learning.mybatis.mapper;

public interface CountryMapper {
    List<Country> selectAll();
}
```

接着，你就可以直接调用接口中的方法了：

``` java
// 获取 SqlSession 对象
SqlSession sqlSession = sqlSessionFactory.openSession();

// 获取 CountryMapper 接口
CountryMapper countryMapper = sqlSession.getMapper(CountryMapper.class);

// 直接调用接口中的方法
List<Country> countryList = countryMapper.selectAll();
```


## resultType 的映射

第一节中我们介绍过，因为 Country 的成员名与表的列名一一对应，所以 MyBatis 可以自动帮我们把表项映射到 Country 对象上。假如名称不对应，就需要手动配置 resultType, 比如 User 类的成员以驼峰方式命名，User 表的列以下划线方式命名，这时候可以在 Mapper.xml 中进行如下指定：

```xml
<mapper namespace="com.dada.learning.mybatis.mapper.UserMapper">
    <resultMap id="userMap" type="com.dada.learning.mybatis.model.SysUser">
        <id property="id" column="id" />
        <result property="userName" column="user_name" />
        <result property="userPassword" column="user_password" />
        <result property="userEmail" column="user_email" />
        <result property="userInfo" column="user_info" />
        <result property="headImg" column="head_img" jdbcType="BLOB" />
        <result property="createTime" column="create_time" jdbcType="TIMESTAMP" />
    </resultMap>

    <!-- 第一种方法，通过 resultMap 来设置数据库 column 和对象 field 之间的映射关系 -->
    <select id="selectById" resultMap="userMap" >
        select * from sys_user where id = #{id}
    </select>

    <!-- 第二种方法，将数据库 column 设置别名为对象 field 的名字，MyBatis 会自动完成映射 -->
    <select id="selectAll" resultType="com.dada.learning.mybatis.model.SysUser" >
        select id,
            user_name userName,
            user_password userPassword,
            user_email userEmail,
            user_info userInfo,
            head_img headImg,
            create_time createTime
        from sys_user
    </select>
</mapper>
```

第一种方法比较笨重，需要定义一个 resultMap, 然后把类的 property 和表的 column 显式关联起来。

第二种方法利用了 sql 语句的别名功能，也就是把列起一个与 property 一样的别名，MyBatis 能够通过别名自动建立起关联。


## 增删改查的样例

### select

之前我们已经演示过 select 的例子，不过都是单表查询，再来看一个多表查询的例子：

```xml
<mapper namespace="com.dada.learning.mybatis.mapper.UserMapper">
    <!-- 多表关联的查询，通过 userId 查询角色，先查到 user, 再查 sys_user_role 得到 role id, 再查 sys_role 表 -->
    <select id="selectRolesByUserId" resultType="com.dada.learning.mybatis.model.SysRole" >
        select
            r.id,
            r.role_name roleName,
            r.enabled,
            r.create_by createBy,
            r.create_time createTime
        from sys_user u
        inner join sys_user_role ur on u.id = ur.user_id
        inner join sys_role r on ur.role_id = r.id
        where u.id = #{userId}
    </select>
</mapper>
```

可以看到在 select 标签里也可以使用多表查询的 sql 语句。值得关注的是 `#{userId}` 这个东西，它对应了接口方法的参数名： `SysRole selectRoleByUserId(Long userId)`.

### insert

先看一个简单的 insert 例子：

```xml
<mapper namespace="com.dada.learning.mybatis.mapper.UserMapper">
    <insert id="insert">
        insert into sys_user (
            id, user_name, user_password, user_email,
            user_info, head_img, create_time)
        values (
            #{id}, #{userName}, #{userPassword}, #{userEmail},
            #{userInfo}, #{headImg, jdbcType=BLOB}, #{createTime, jdbcType=TIMESTAMP})
    </insert>
</mapper>
```

其实跟之前 select 差不多。不过上面的例子没有考虑到返回主键的情况——有时候我们需要先插入一个数据，得到这次插入后生成的主键，然后拿着这个主键做进一步的操作。



