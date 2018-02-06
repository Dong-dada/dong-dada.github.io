---
layout: post
title:  "Java 基础知识"
date:   2018-01-22 16:08:30 +0800
categories: java
---

* TOC
{:toc}

最近在学 Java, 这篇文章大多来自于 《Java 核心技术》(第 10 版) 这本书。

## 基本数据类型

Java 中有 8 种基本数据类型：

| type    | size   | range                            |
| :------ | :----- | :------------------------------- |
| int     | 4 byte | -2147483648 - 2147483647 (20 亿) |
| short   | 2 byte | -32768 ~ 32767                   |
| long    | 8 byte | 很大                             |
| byte    | 1 byte | -128 ~ 127                       |
| float   | 4 byte | ± 3.40282347E+38F                |
| double  | 8 byte | ± 1.79769313486231570E+308       |
| char    | 2 byte | \u0000 ~ \uffff                  |
| boolean | 1 bit  | true or false                    |

注意：
- Java 中没有无符号 (unsigned) 形式的 int, long, short, byte 类型；
- Java 中可以用 0xFF 来表示十六进制数 ，用 0b101010 来表示二进制数；
- 一般而言，一个 char 代表一个 Unicode 字符，某些特殊的 Unicode 字符需要两个 char 来表示，具体可以参考之后的介绍；
- boolean 和 int 类型之间不能进行相互转换，也就是说 `if (1)` 这样的语句不能通过编译。

### char 与 Unicode

在 Unicode 设计之初，人们认为两个字节的代码宽度足以对世界上各种语言的所有字符进行编码，因此 1991 年发布的 Unicode 1.0 中只占用了 65536 个代码值中的不到一半。

所以设计 Java 的时候，将 char 用两个字节来表示。

经过一段时间后， Unicode 的字符超过了 65536 个，其主要原因是增加了大量的汉语、日语和韩语中的表意文字。因此原有的 16 位的 char 就不能满足描述所有 Unicode 字符的需要了。

在 Unicode 标准中，**码点** 用来表示某个字符在编码表中的位置。 Unicode 把所有码点分为 17 个代码级别 (code plane)，也称为 代码平面。

第一个代码级别被称为 **基本的多语言级别(basic multilingual plane)**，码点范围是 `U+0000 ~ U+FFFF`。剩下的 16 个代码级别，其范围是 `U+10000 ~ U+10FFFF`。

需要注意的是，在基本级别中，并不是所有的编码值都对应于一个 Unicode 码点，其中还包含了一些辅助字符。 比如 `D800 ~ DBFF` 表示的是 UTF-16 的高半区，`DC00-DFFF` 表示的是 UTF-16 的低半区。

UTF-16 编码采用不同长度的编码表示所有 Unicode 码点。对于基本的多语言级别，每个字符用 2 个字节表示，通常被称为 **代码单元**(code unit)。 对于更高级别，则需要采用两个连续的代码单元(也就是 4 个字节)来进行编码。这时候之前说过的 `D800 ~ DBFF`, `DC00 ~ DFFF` 这些空闲区域就派上用场了，比如 `U+1D546` 这个特殊字符，可以用两个代码单元 `U+D835` 和 `U+DD46` 来表示。

对于 Java 而言，一个 char 字符固定为 2 个字节大小，它表示了 UTF-16 编码中的一个代码单元。换句话说，对于高级别的字符来说，需要两个 char 来表示。

我们强烈建议不要在程序中使用 char 类型，除非确实需要处理 UTF-16 代码单元。

上述内容可参考原文，以及 [维基百科](https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84)

## 字符串

从概念上讲，Java 字符串就是 Unicode 字符序列 （UTF-16 编码）。

Java 没有内置的字符串类型，而是通过标准库提供了一个 `String` 类(类似于 C++ 的 `std::wstring`)。

```java
// String 对象
String s;       // s == null
String e = "";  // e != null, 空字符串
String greeting = "Hello";

// 子串
s = greeting.substring(0, 3);

// 拼接
String expletive = "Expletive";
String PG13 = "deleted";
String message = expletive + PG13;
message = "Expletive" + PG13;
message = "Hello" + " " + "World!";

// join
String all = String.join("/", "S", "M", "L", "XL");
// all 的值为 S/M/L/XL

// 检查字符串是否相等
if (greeting.equals(message)) {}
if (greeting.equals("Hello")) {}
if (greeting.equalsIgnoreCase("hello")) {} // 忽略大小写
if ("Hello".equals(greeting)) {}           // 字符串也可以调用 .equals

// 不要使用 == 来比较字符串
if ("hello" == greeting) {
    // probably false
}

// 返回代码单元数量，也就是 char 的数量
int n = greeting.length();

// 获取指定位置的代码单元，也就是 char
char first = greeting.charAt(0);
char last = greeting.charAt(greeting.length() - 1);

// 返回码点数量，可能小于 length()
int cpCount = greeting.codePointCount(0, greeting.length());

// 获取第 n 个码点，码点用 int 而不是 char 来表示
int offset = greeting.offsetByCodePoints(0, n);
int cp = greeting.codePointAt(offset);

// 返回一个代表码点的 int 数组
int[] codePoints = str.codePoints().toArray();

// 用码点数组构造 String
String str = new String(codePoints, 0, codePoints.length);
```

注意：
- String 对象是不可变的，一旦构建了一个 String 对象，那么其包含的字符串就无法被修改；
- 不要用 `==` 来比较两个字符串，因为 `==` 只会判断两个变量是否引用了同一位置的字符串，有可能两个字符串的内容相同，但存储在不同的地方，此时 `==` 仍然会返回否；
- 码点和代码单元的区别，上一小节已经介绍过了，这里不再重复；

### 使用 StringBuilder 来构建字符串

有时候，需要由较短的字符串构成字符串，比如提取文件中的若干单词拼成字符串，这时候采用字符串连接的方式效率比较低，因为每次连接字符串，都会构建一个新的 String 对象。使用 `StringBuilder` 可以避免这个问题。

```java
StringBuilder builder = new StringBuilder();
builder.append(ch);     // appends a single character
builder.append(str);    // appends a string
builder.appendCodePoint(cp);    // appends a code point

builder.setCharAt(0, ch);

builder.insert(0, "hello");

builder.delete(0, 1);

String completedString = builder.toString();
```


## 数组

数组是 Java 原生支持的一种数据结构，类似于 C++ 中的数组，可以用下标来访问。

```java
int n = 100;

// 创建 int 数组，所有数据都初始化为 0
int[] a = new int[n];

// 创建 int 数组，并使用列表来初始化
int[] smallPrimes = {2, 3, 5, 7, 11, 13};

// 初始化匿名数组
new int[] {17, 19, 23, 29, 31};

// 创建 String 数组，所有数据都初始化为 null
String[] names = new String[10];

// 拷贝组数，第二个参数是新数组的长度，多余的元素将被赋值为 0
int[] copiedNumbers = Arrays.copyOf(smallPrimes, 2 * smallPrimes.length);

// 数组排序，只能排基本类型
Arrays.sort(copiedNumbers);

// 二分查找，成功返回 index, 否则返回负数
int index = Array.binarySearch(smallPrimes, 13);

// 比较数组是否相等
if (Array.equals(copiedNumbers, smallPrimes)) {}

// 使用下标访问数组
for (int i = 0; i < names.length; ++i) {
    System.out.println(names[i]);
}

// 使用 for each 循环遍历数组
for (String name : names) {
    System.out.println(name);
}
```

注意：
- 数组也是一个对象，也继承自 Object;
- 数组的长度允许为 0, 比如 `int[] result = new int[0]`；
- Java 中的数组不允许通过 `a + 1` 的方式的到数组的下一个元素；


## 类

```java
public class Employee {
    // 静态成员变量
    private static int nextId = 1;

    // 常量
    private static final double MIN_SALARY = 10000;

    // fields, 也就是 C++ 中的成员变量
    private String name;
    private double salary;
    private LocalDate hireDay;
    private final StringBuilder evaluations;

    // constructor 构造函数
    public Employee(String name, double salary, int year, int month, int day) {
        this.name = name;
        this.salary = salary;
        this.hireDay = LocalDate.of(year, month, day);
    }

    public Employee(String name, int year, int month, int day) {
        this.name = name;
        this.salary = MIN_SALARY;
        this.hireDay = LocalDate.of(year, month, day);
    }

    // 静态方法
    public static int getNextId() {
        return nextId;
    }

    // 访问器
    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }

    public LocalDate getHireDay() {
        return hireDay;
    }

    public StringBuilder getEvaluations() {
        return evaluations;
    }

    // 修改器
    public void setName(String name) {
        this.name = name;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }

    // 其它方法 ...
}

public class Main {
    public static void main(String[] args) {
        Employee[] staff = new Employee[3];
        staff[0] = new Employee("Carl Cracker", 75000, 1987, 12, 15);
        staff[1] = new Employee("Harry Hacker", 50000, 1989, 10, 1);
        staff[2] = new Employee("Tony Tester", 40000, 1990, 3, 15);

        for (Employee e : staff)
            System.out.println(e.getName());
    }
}
```

注意：
- `public class Employee` 这行代码中，指定了 `Employee` 类的访问级别为 `public`。Java 中包含了 4 种访问级别，稍后详细介绍；
- `private final StringBuilder evaluations;` 这行代码中，`final` 用于表示这个 field 不可变。这个不可变类似于 C++ 中的 `const int* p`，也就是说 p 指向的内容可以被改变，但 p 本身不能再指向别的变量。对应于 Java，你可以调用 `this.evaluations.append()` 来修改 `this.evaluations`, 但不能改变 `this.evaluations` 所引用的 StringBuilder 实例，因此上述例子中没有提供 `setEvaluations()` 修改器。但是一旦你通过 `getEvaluations()` 访问器得到了 evaluations, 你就可以通过 `StringBuilder.append()` 等方法来修改它了。
- `private static int nextId = 1;` 这行代码声明了一个静态变量，跟 C++ 是一致的。

### Java 中的访问权限

Java 中包含了 4 中访问级别，这几种访问级别可以用来描述 类本身、类成员 的访问权限，具体可以参考下图：

```
            | Class | Package | Subclass | Subclass | World
            |       |         |(same pkg)|(diff pkg)|
------------+-------+---------+----------+----------+--------
public      |   +   |    +    |    +     |     +    |   +
------------+-------+---------+----------+----------+--------
protected   |   +   |    +    |    +     |     +    |
------------+-------+---------+----------+----------+--------
no modifier |   +   |    +    |    +     |          |
------------+-------+---------+----------+----------+--------
private     |   +   |         |          |          |

+ : accessible
blank : not accessible
```

- public : 谁都可以访问；
- protected : 不能被其他包中的非子类访问；
- no modifier : 只能被当前包的类访问；
- private : 只能被当前类访问；

声明类，声明类成员的时候，应该按照上述规则来合理使用访问权限。

### 传参

**Java 总是采用值传递的方式来传参**， 只是传入的参数如果是一个引用的话，那么会以值的方式复制这个引用。

```java
public static void tripleValue(double x) {
    // doesn't work
    // 并不能起到将 x 乘 3 的效果
    x = 3 * x;
}

public static void tirpleSalary(Employee x) {
    // works
    // x 是一个引用的复制，修改它可以起到修改原有实例的作用
    x.raiseSalary(200);
}

public static void swap(Employee x, Employee y) {
    // doesn't work
    // x, y 是引用的复制，交换他们并不能起作用
    // 换句话说，你不能在函数体内部，通过传参的方式修改外部变量的引用，
    // 因为这个引用是按值传递的
    Employee temp = x;
    x = y;
    y = temp;
}
```

### finalize 方法

由于 Java 中有自动的垃圾回收器，不需要人工回收内存，所以 Java 不支持析构器。

但你可以定义一个 finalize 方法，在对象被回收时做一些资源清理的工作，比如关闭文件句柄。


## 包

Java 使用包 (package) 来把类组织起来，包跟 C++ 中的 namespace 类似，可以用来确保类名的唯一性。

Sun 公司建议将公司的因特网域名(独一无二的)以逆序的形式作为包名，并且对于不同的项目使用不同的子包。

需要说明的是，嵌套的包之间没有任何关系，例如 java.util 包和 java.util.jar 包毫无关系，每个包都拥有独立的类集合。

另外需要说明的是，并不是一定要导入包，才能使用包中的类，比如你可以用以下两种方式来使用 LocalDate :

```java
// 使用完整包名
java.time.LocalDate today = java.time.LocalDate.now();

// 先导入包，然后再使用包里的类
import java.time.LocalDate

LocalDate today = LocalDate.now();
```

`import` 语句更像是 C++ 中的 `using namespace`，而不是 `#include`。Java 中并没有与 `#include` 相对应的概念，你不需要在代码中通过类似 `#include` 的方式来告诉编译器类定义的信息，Java 会自己查看每个文件，得到其中的类定义。当你使用 `java.time.LocalDate` 这种方式访问一个类的时候，Java 会根据包名来找到对应的文件，从而得知 `LocalDate` 的类定义。

`import` 语句的唯一好处是方便，可以使用简短的名字而不是完整包名来引用一个类，它跟 `#include` 的作用完全不同。

### 将类放入包中

你需要把包的名字放在源文件开头，以便告诉编译器当前类所属的包是哪个：

```java
package com.dada;

public class Employee {
    //...
}
```

这样，就可以通过 `com.dada.Employee` 来使用这个类了。

如果没有指定 `package com.dada`, 那么 `Employee` 类将被放在默认包 (default package) 中，就像在 C++ 中没有指定 namespace 一样。

你应该按照包名来组织文件，比如把 Employee.java 放在 `com/dada/` 目录中。


## 继承

```java
public class Manager extends Employee {
    // 新增 fields
    private double bonus;

    // 构造函数
    public Manager(String name, double salary, int year, int month, int day) {
        super(name, salary, year, month, day);
        bonus = 0;
    }

    // 新增 method
    public void setBouns(double bonus) {
        this.bonus = bonus;
    }

    // 覆盖基类 method
    public double getSalary() {
        return super.getSalary() + bonus;
    }
}
```

注意：
- Java 中使用 `extends` 来继承另一个类，与 C++ 不同的是，Java 只能继承一个类，不能多重继承；
- Java 中使用 `super` 来访问父类方法；
- Java 中覆盖基类的方法，不需要 virtual 关键字，因为覆盖是默认的行为。
- Java 中多态的表现与 C++ 类似，这里不再赘述；

### 阻止继承

如果你不希望其他人继承你的类，或者不希望基类的某个方法被覆盖，那么可以使用 final 关键字：

```java
// 将 Executive 类标识为不可继承
public final class Executive extends Manager {

}

public class Employee {
    // 将 getName 表示为不可覆盖，
    public final String getName() {
        return name;
    }
}
```

### 抽象类

Java 中可以使用 abstract 关键字将某个方法标识为抽象方法，类似于 C++ 中的 `virtual void foo() = 0;` 你无需提供该方法的实现。

一旦类中有某个方法是抽象的，那么类本身也应该声明为抽象类：

```java
public abstract class Person {
    private String name;
    public Person(String name) {
        this.name = name;
    }

    public abstract String getDescription();

    public String getName() {
        return name;
    }
}
```

与 C++ 类似，抽象类中也可以包含成员变量和成员函数，但你不能直接创建抽象类的实例。

抽象类的用处应该不多，因为 Java 中提供了接口机制来对类的功能进行描述。接口不包含 fields, 因此语义上更加单纯一些。

### Object -- 所有类的基类

Object 是 Java 中所有类的始祖。它包含了以下几个方法：
- `equals()` : 检查一个对象是否等于另外一个对象。Object 中的实现是判断他们的引用是否相等，这种方法不一定有效，大多数情况下我们需要判断的是两个不同引用的 Object 是否具有相同的状态，关于相同性检查的方法，我们后面会介绍；
- `getClass()` : 返回一个对象所属的类，这一点会在后面介绍；
- `hashCode()` : 返回一个整型值。需要注意的是如果 `equals()` 返回 true, 那么两个对象的 `hashCode()` 返回值也应当一致。因此如果你自定义了 equals 方法，那么同时也应当自定义 `hashCode()` 方法。Object 类的实现是返回对象的存储地址。
- `toString()` : 返回一个描述对象的字符串；

自定义 `equals()` 的例子：

```java
public class Employee {
    // ...

    public boolean equals(Object otherObject) {
        // 自身
        if (this == otherObject) return true;

        if (otherObject == null) return false;

        // 不是同一个类
        if (getClass() != otherObject.getClass()) return false;

        Employee other = (Employee)otherObject;
        return name.equals(other.name) &&
               salary == other.salary &&
               hireDay.equals(other.hireDay);
    }
}
```

Java 语言要求 equals 方法具有以下特性：
- 自反性： x.equals(x) 应当返回 true;
- 对称性： x.equals(y) 应当与 y.equals(x) 一致；
- 传递性： 如果 x.equals(y) 返回 true, y.equals(z) 返回 true, 那么 x.equals(z) 也应当返回 true；
- x.equals(null) 应当返回 false。

自定义 `hashCode()` 的例子：

```java
public class Employee {
    // ...

    public int hashCode() {
        return Objects.hash(name, salary, hireDay);
    }
}
```

`Objects.hash()` 方法将组合所有参数的散列码，得到一个新的散列码。


## 泛型数组列表 ArrayList

Java 中提供了一个类似于 C++ 中 `std::vector<>` 的容器，即 ArrayList.

```java
ArrayList<Employee> staff = new ArrayList<Employee>();

// 预先分配足够大的空间来容纳元素
staff.ensureCapacity(100);

// 在数组尾部添加新元素
staff.add(new Employee("Harry Hacker", ...));
staff.add(new Employee("Tony Tester", ...));

// 在数组指定位置插入元素
staff.add(1, new Employee("Dongdada", ...));

// 移除数组中指定位置的元素
staff.remove(1);

// 回收多余的空间
staff.trimToSize();

// 访问数组元素
Employee harry = staff.get(0);
Employee tony = staff.get(1);

staff.set(0, tony);
staff.set(1, harry);

// 使用 for 循环访问数组元素
for (int i = 0; i < staff.size(); ++i) {
    Employee e = staff.get(i);
    // ...
}

// 使用 for each 访问数组元素
for (Employee e : staff) {
    // ...
}

// 转换为普通数组
Employee[] x = new Employee[staff.size()];
staff.toArray(x);
```


## 对象包装器及自动装箱

ArrayList 只能接受对象作为类型参数，对于 int 这样的基本类型，它无法容纳，也就是说 `ArrayList<int>` 是不合法的。

Java 提供了一种称为对象包装器的机制，可以把基本类型转换为相应的对象。这些对象包装器类拥有很明显的名字： `Integer`, `Long`, `Float`, `Double`, `Short`, `Byte`, `Character`, `Void`, `Boolean`。

对象包装器是不可变的，一旦构造了包装器，就不能再修改包装器中的值：

```java
ArrayList<Integer> list = new ArrayList<>();

// 自动装箱，相当于 list.add(Integer.valueOf(3))
list.add(3);

// 自动拆箱，相当于 int n = list.get(i).intValue();
int n = list.get(0);
```


## 可变参数

在 `System.out.printf()` 这样的方法中，可以接受多个参数：

```java
System.out.printf("%d, %s", n, "Widget");
```

定义可变参数的方法为：

```java
public static double max(double... values) {
    double largest = Double.NEGATIVE_INFINITY;
    for (double v : values)
        if (v > largest)
            largest = v;

    return largest;
}
```

`double... values` 相当于 `double[] values`，Java 会帮你把参数转换为一个数组。


## 枚举类

Java 中可以像 C++ 那样定义枚举：

```java
public enum Size{
    SMALL,
    MEDIUM,
    LARGE,
    EXTRA_LARGE
};
```

所有的枚举类型都继承自 Enum 类，它提供了 `toString` 方法可以直接返回字符串形式的枚举值，比如上述例子中 `Size.SMALL.toString()` 方法将返回 `"SMALL"`；

此外 Enum 还提供了 values 方法，它将返回一个包含所有枚举值的数组： `Size[] values = Size.values()`。

此外你还可以在枚举中增加一些 field, 构造器，方法：

```java
public enum Size {
    SMALL("S"),
    MEDIUM("M"),
    LARGE("L"),
    EXTRA_LARGE("XL");

    // 简称
    private String abbreviation；

    // 构造器声明为 private, 因为只在构造枚举常量的时候被使用。
    private Size(String abbreviation) {
        this.abbreviation = abbreviation;
    }

    public String getAbbreviation() {
        return this.abbreviation;
    }
}
```


## 反射

Java 运行时系统会为所有对象维护一个被称为运行时的类型标识，有点儿类似于 C++ 的 `type_info`。

能够分析类能力的程序称为反射(reflective)。反射机制的功能极其强大，它可以用来：
- 在运行时分析类的能力；
- 在运行时查看对象；
- 实现通用的数组操作代码；
- 利用 Method 对象，它类似于 C++ 中的函数指针；

Java 提供了 Class 类来访问类型信息，java.lang.reflect 包提供了 Constructor, Method, Field 类来访问类的构造器、方法、域 等信息。

```java
Employee e = new Employee(...);

// 通过对象的 getClass 方法获取 Class 对象
Class cl = e.getClass();

// 获取父类的 Class 对象
Class superClass = cl.getSupperClass();

// 通过类名获取对应的 Class 对象
Class cl = Class.forName("java.util.Random");

// 通过 .class 方法获取 Class 对象
Class cl = int.class;
Class cl = double[].class;

// 获取类名
String className = cl.getName();

// 创建实例
Object obj = cl.newInstance(...);

// 获取构造器信息
Constructor[] constructors = cl.getDeclaredConstructors();
for (Constructor c : constructors) {
    // 构造函数名
    String name = c.getName();

    // 获取修饰符，比如 public, static, final
    String modifiers = Modifier.toString(c.getModifiers());

    // 获取参数信息
    Class[] paramTypes = c.getParameterTypes();
    for (int i = 0; i < paramTypes.length; ++i) {
        String paramName = paramTypes[i].getName();
    }
}

// 获取 Method 信息
Method[] methods = cl.getDeclaredMethods();
for (Method m : methods) {
    // 返回值信息
    Class retType = m.getReturnType();

    // 修饰符和参数信息
    // ...
}

// 获取 fields 信息
Field[] fields = cl.getDeclaredFields();
for (Field f : fields) {
    Class type = f.getType();
    String name = f.getName();
    String modifiers = Modifier.toString(f.getModifiers());

    // 获取 employee 对象中，指定 field 的内容
    // 先设置访问权限，否则无法访问 private 的 field
    f.setAccessible(true);
    Object value = f.get(employee);

    // 还可以设置 employee 对象的指定 field 的内容：
    f.set(employee, value);
}
```

可以看到，利用反射机制，我们可以在运行时查看任意类的详细信息，甚至还可以修改它。

我们可以利用这一机制来实现一个通用的 toString 方法，这样就不用每次创建一个新的类都自己来定义 `toString()` 了。

另外，假如你想编写一个通用的方法，用于拷贝一个数组。那么在拷贝数组时，你也可以利用反射来获取原始的类型信息，从而完成新元素的创建。

### 利用反射调用任意方法

C++ 中可以通过函数指针来执行任意函数。Java 没有提供指针，所以不能像 C++ 那样方便地调用所有方法。

不过利用反射，可以完成类似的操作，其关键在于 `Method.invoke()` 方法：

```java
// 从 Class 中获取到感兴趣的 Method，例如获取 Employee.getName 方法
Method method = cl.getMethod("getName");

// 把 Method 传递到某个地方
// ...

// 在另外的地方调用该方法
String name = method.invoke(obj);
```


## 接口

接口用于描述类具有什么功能，一个类可以实现多个接口。例如 Arrays 类中的 sort 方法需要每个对象都实现了 Comparable 接口：

```java
// 定义 Comparable 接口
public interface Comparable {
    int compareTo(Object other);
}

// 实现 Comparable 接口
class Employee implements Comparable {
    // ...

    public int compareTo(Object otherObject) {
        Employee other = (Employee)otherObject;
        // 比较薪资来返回大小
        return Double.compare(salary, other.salary);
    }
}

public class Main {
    public static void main(String[] args) {
        Employee[] staff = new Employee[3];
        // ...

        Arrays.sort(staff);
    }
}
```

注意：
- interface 中的方法不需要指定 public 等访问限定符，因为默认都是 public；
- 可以使用 instanceof 来检查一个对象是否实现了某个接口；
- 接口之间也可以继承，但也只能单继承；
- 接口之中不能包含 fields, 或静态方法，但可以包含常量。
- 可以为接口方法提供一个默认实现，并用 default 修饰符标记此方法，请看稍后的代码示例：
- 有一点跟 C++ 很不一样的是，你不能在继承之后，把方法的 public 改为 private，可以参考接下来的代码示例；

```java
// default 作为默认实现
public interface MouseListener {
    default void mouseClicked(MouseEvent event) {}
    default void mousePressed(MouseEvent event) {}
}
```

```java
class Worker implements Runnable {
    // run 方法在 Runnable 接口中是 public, 你不能把它修改为 private
    private void run() {
        // ...
    }
}
```

在 C++ 中常常这么做，比如一个类实现了 MouseListener 接口，用以处理鼠标事件，显然这种情况下我不希望外部使用我这个类的时候，把 `mouseClicked()` 也作为一个 public 方法暴露出去，这时候就会把它声明为 private。

然而 Java 不允许这么做。可能它认为接口描述的是一个类可以做什么，如果一个类实现了接口，却不暴露这个接口中的方法，那就违背了实现接口的初衷。

更进一步考虑，我们一般不会直接暴露类给调用方，调用方应该是持有接口来访问类对象的。如此一来我们对外提供的应该是一个接口，只要不向外提供 MouseListener 接口，那么外部也就没法访问 `mouseClicked()` 方法了。

使用内部类也许也是一种比较符合逻辑的做法，通过内部类来实现接口，利用内部类来处理 mouseListener 中的各种事件，也是一种方法。


## 对象克隆

Object 类有一个 protected 的方法 `clone()` 显然这个方法用于克隆一个对象。

值得注意的是它的访问级别为 protected, 这样的话我们无法直接调用它。想想也能明白，Object 作为所有类的始祖，并不了解子类的具体实现，因此它实现 `clone()` 时只能对每个 field 进行拷贝，对于某些引用 field，它无法执行深拷贝，所以它被设置为 protected 状态，由子类来确定是否有引用需要进行深拷贝，如果有，则进行特殊处理。

现在看来，如果我们希望某个类支持克隆，那么我们应该实现 clone 方法，解决深拷贝问题，并将其访问权限设置为 public。

然而这样还不行，我们还得让类来实现 Cloneable 接口，这个接口里并没有指定什么方法，它只是作为一个标记，指明这个类的设计者，也就是我们，认为这个类是支持拷贝的。 `Object.clone()` 会检查我们是否实现了这个接口，如果没有实现则抛出异常。

以下是实现 clone() 方法的例子：

```java
class Employee implements Cloneable {
    // ...

    public Employee clone() throws CloneNotSupportedException {
        // 利用 Object.clone 进行浅拷贝
        Employee cloned = (Employee) super.clone();

        // 深拷贝那些属于引用的 field
        cloned.hireDay = (Date)hireDay.clone();

        return cloned;
    }
}
```

上述例子中，如果 field 没有实现 Cloneable 接口，就会抛出 CloneNotSupportedException 异常。


## lambda 表达式

Java8 中引入了 lambda 表达式的支持。 其语法为：

```java
(parameters) -> {
    expression;
}

// 下列 lambda 表达式用于比较两个 String 的长度
(String first, String second) -> {
    if (first.length() < second.length())
        return -1;
    else if (first.length() > second.length())
        return 1;
    else
        return 0;
}
```

你无需指定返回值的类型，Java 会根据上下文进行推导。

### 函数式接口

之前我们介绍过的 `Comparator` 接口，它只有一个抽象方法。这种情况下可以把 lambda 表达式直接转换为对应的接口对象：

```java
// 将 lambda 表达式转换为接口
Comparator<String> comp = (String first, String second) -> {
    // ...
}

Arrays.sort(words, comp);
```

这种由 lambda 转换过来的接口，被称为函数式接口(functional interface)。

来看另外一个例子：

```java
ActionListener listener = (ActionEvent event) -> {
    System.out.println("At the tone, the time is " + new Date());
};
Timer t = new Timer(1000, listener);
```

上述例子中，因为返回值是 ActionListener 接口类型，Java 可以推断出 lambda 的参数类型应该是 `ActionEvent` 这种情况下可以省略参数类型，写成 `(event) -> { ... }`，此外，当只有一个参数的时候，小括号也可以省略，因此还可以继续简写为 `event -> {...}`。 如果表达式很简单，大括号也可以省略，简写为 `event -> ...`，所以见到类似 `x -> return xxx` 的情况不要懵圈儿了。

注意：
- Java 不像 C++ 那样提供了 `std::function` 用于以变量的形式接收 lambda 表达式。在 Java 中，只能把 lambda 转换为函数式接口来存储到变量中，并没有新增类似于 `std::function` 的类型。
- Java 的 java.util.function 包中提供了许多通用的函数式接口，比如 `BiFunction<T, U, R>` 描述了参数类型为 T 和 U, 返回类型为 R 的函数。如果你见到某个函数需要以 `BiFunction` 作为参数，也不要懵圈儿了。

### 方法引用

有时候，可能已经有现成的方法可以完成你想要传递到其他代码的某个动作，这时候你可以直接把方法引用传递给需要接口的地方。例如：

```java
Timer t = new Timer(1000, event -> System.out::println);
```

`System.out::println` 就是所谓的方法引用。它等价与 lambda 表达式 `x -> System.out.println(x)`。

类似的例子还有：

```java
Arrays.sort(strings, String::compareToIgnoreCase);
```

你还可以引用某个类的构造器，例如：

```java
ArrayList<String> names = ...;
Stream<Person> stream = names.stream.map(Person::new);
```

### 闭包

ok, 上述内容已经介绍了一些 lambda 表达式的知识，你知道了如何编写一个 lambda 表达式，如何简写它。接下来讨论下 lambda 中比较麻烦的一件事，也就是如何引用外部变量。

Java 把 lambda 表达式引用的外部变量称为自由变量，有以下几个规则：
- lambda 表达式中不能修改自由变量，否则多个地方调用同一个 lambda 表达式时可能会出现问题；
- lambda 表达式捕获自由变量后，外部不许修改这个自由变量；
- lambda 表达式中使用 this 关键字时，是指创建这个 lambda 表达式的方法的 this 参数；

lambda 表达式的重要意义在于它能够完成延迟执行(deferred execution)。毕竟如果你可以直接执行一段代码的话，你就没必要使用 lambda 表达式把它包起来。

要实现延迟执行，关键是要实现类似于 `std::bind()` 的函数，在 C++ 中，它接受一个 `std::function`，并将函数的几个参数绑定起来，返回一个新的 `std::function`。新的 `std::function` 调用时就不需要再填充已经绑定过的参数了。

然而我在书中没有读到类似的实现，这一点还得以后再确认一下。


## 内部类

Java 中提供了类似于 C++ 中内嵌类的机制：

```java
package com.dada;

import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.util.Date;

public class TalkingClock {
    private int interval;
    private boolean beep;

    public TalkingClock(int interval, boolean beep) {
        this.interval = interval;
        this.beep = beep;
    }

    public void start() {
        ActionListener listener = new TimePrinter();
        Timer t = new Timer(interval, listener);
        t.start();
    }

    // TimePrinter 内部类，用于实现 ActionListener 接口
    public class TimePrinter implements ActionListener {
        @Override
        public void actionPerformed(ActionEvent e) {
            System.out.println("At the tone, the time is " + new Date());

            // 注意 beep 是 TalkingClock 类的 field
            if (beep) {
                Toolkit.getDefaultToolkit().beep();
            }
        }
    }
}
```

上述代码非常简单，跟 C++ 的内嵌类唯一的区别在于，在内部类里可以访问外部类的 fields. Java 编译器会在创建内部类的时候把外部对象的引用传递过去。

下面更进一层，看看局部内部类。

### 局部内部类

局部内部类，也就是把类定义写在函数内部的做法：

```java
public void start(boolean beep) {
    class TimerPrinter implements ActionListener {
        public void actionPerformed(ActionEvent event) {
            System.out.println("At the tone, the time is " + new Date());
            // 这里访问的是局部变量
            if (beep)
                Tookkit.getDefaultToolkit().beep();
        }
    }

    ActionListener listener = new TimePrinter();
    Timer t = new Timer(interval, listener);
    t.start();
}
```

局部内部类的优点在于可以访问局部变量。不过需要注意的是，这些局部变量事实上应该是不可变的。

### 匿名内部类

匿名内部类更进一步，你不需要命名一个新的类了：

```java
public void start(int interval, boolean beep) {
    ActionListener listener = new ActionListener() {
        public void actionPerformed(ActionEvent event) {
            System.out.println("At the tone, the time is " + new Date());
            // 这里访问的是局部变量
            if (beep)
                Tookkit.getDefaultToolkit().beep();
        }
    }

    Timer t = new Timer(interval, listener);
    t.start();
}
```

上述代码的含义是，创建一个实现 ActionListener 接口的类的新对象。

多年来，Java 程序员习惯的做法是使用匿名内部类是吸纳事件监听器和其它回调。如今最好还是使用 lambda 表达式。
