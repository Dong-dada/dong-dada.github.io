---
layout: post
title:  "服务端开发小技巧"
date:   2019-03-08 17:10:30 +0800
categories: server
---

* TOC
{:toc}

## MyBatis DO 中的特殊类型处理办法

有时候会在 DO 中定义一些特殊类型，比如枚举、自定义数据结构等，直接交给 MyBatis 处理的话会报错，除非定义对应的 TypeHandler.

TypeHandler 是一个类，感觉用起来比较麻烦。

初步的想法是重写 DO 中的 getter/setter 方法，在其中加上普通类型 <--> 特殊类型的转换。这样在代码中就可以直接用这些特殊类型来与 DO 交互：

```java
public enum Grade {
    PRIMARY_SCHOOL,
    JUNIOR_HIGH_SCHOOL,
    HIGH_SCHOOL,
}

@Data
public class Student {
    private Long id;
    private String name;
    private Integer grade;

    public Grade getGrade() {
        switch(grade) {
            case 1:
                return Grade.PRIMARY_SCHOOL;
            case 2:
                return Grade.JUNIOR_HIGH_SCHOOL;
            case 3:
                return Grade.HIGH_SCHOOL;
            default:
                return null;
        }
    }

    public void setGrade(Grade grade) {
        switch(grade) {
            case Grade.PRIMARY_SCHOOL:
                this.grade = 1;
                break;
            case Grade.JUNIOR_HIGH_SCHOOL:
                this.grade = 2;
                break;
            case Grade.HIGH_SCHOOL:
                this.grade = 3;
                break;
            default:
                break;
        }
    }
}
```

不过 MyBatis 会调用 DO 的 getter/setter, 所以虽然字段还是普通类型，MyBatis 拿到的仍然是特殊类型。

纠结之下决定给 getter/setter 起个新名字，叫做 assign/retrieve, 表示它们在 get/set 之外还做了一些转换动作：

```java
@Data
public class Student {
    private Long id;
    private String name;
    private Integer grade;

    public Grade retrieveGrade() {
        switch(grade) {
            case 1:
                return Grade.PRIMARY_SCHOOL;
            case 2:
                return Grade.JUNIOR_HIGH_SCHOOL;
            case 3:
                return Grade.HIGH_SCHOOL;
            default:
                return null;
        }
    }

    public void assignGrade(Grade grade) {
        switch(grade) {
            case Grade.PRIMARY_SCHOOL:
                this.grade = 1;
                break;
            case Grade.JUNIOR_HIGH_SCHOOL:
                this.grade = 2;
                break;
            case Grade.HIGH_SCHOOL:
                this.grade = 3;
                break;
            default:
                break;
        }
    }
}
```

这样可以让使用 DO 的代码方便一些，也不需要写 TypeHandler 来做转换。

同样的思路可以应用在一些 RPC 接口上，一些编程规范认为不应当在参数或返回值中使用枚举。可以考虑用上述思路来避免一些反序列化导致的问题。