
## 数据类型

Java 的数据类型有 8 种：

| type    | size   |
| ------- | ------ |
| int     | 4 byte |
| short   | 2 byte |
| long    | 8 byte |
| byte    | 1 byte |
| float   | 4 byte |
| double  | 8 byte |
| char    | 2 byte |
| boolean | 1 bit  |

值得注意的是，Java 中没有无符号 (unsigned) 形式的 int, long, short, byte 类型。

Java 中也可以用 0xFF 这样的形式来表示 16 进制数，还可以用 0b1001 这样的形式来表示二进制数。

在 Java 中, char 类型描述了 UTF-16 编码中的一个代码单元（2 个字节）。对于 UTF-16 而言，有些辅助字符可能无法用两个字节来表示，这时候会用一对连续的代码单元来编码(总计 4 个字节)。

Java 中，整形和 boolean 之间不能相互转换。

Java 中，使用关键字 final 指示常量，例如：

```java
public class Foo
{
    public static final double CM_PER_INCH = 2.54;

    public void Bar()
    {
        final double PAI = 3.1415926;
        // ...
    }
}
```

const 虽然是 Java 保留的关键字，但目前并未使用。

### 字符串

Java 没有内置的字符串类型，而是在标准 Java 类库中提供了一个预定义类 String:

```java
String greeting = "Hello";
```

String 就是 Unicode 字符序列。

值得注意的是，String 所指向的字符串是不可辨的，它也没有提供修改字符串的方法。你可以想象 Java 把各种字符串放在了公共的存储池中，String 只是指向了存储池中字符串相应的位置。 Java 设计者认为程序中很少需要修改字符串，往往是对字符串进行比较，因此他们采用了共享的方式来提高字符串操作的效率。

你可以使用 + 号连接两个字符串：

```
String message = "Hello" + " " + "World";
```

当将一个字符串与一个非字符串进行拼接时，后者会被转换成字符串：

```java
int age = 13;
String rating = "PG" + age;
```

你可以使用 equals 方法来检查两个字符串是否相等，例如：

```java
String str = "Hello";
str.equals("Hello");    // true
"hello".equals(str);    // false
"hello".equalsIgnoreCase("Hello");  // true
```

注意： 不要使用 `==` 来检测两个字符串是否相等，它们只能确定两个字符串是否放在同一个位置上，如果是在同一位置，它们必然相等。但不在同一位置的两个字符串，也有可能是相等的。

String 还可以存放一个特殊的值，名为 null, 表示目前没有任何对象与该变量关联，它不同于空串。因此，如果你要检查一个字符串是否为空，需要使用以下条件：

```java
if (str != null && str.length() != 0)
```

有些时候，需要由较短的字符串构建字符串，采用字符串拼接的方式效率比较低，每次连接都会构建一个新的字符串。使用 StringBuilder 类可以高效率的达成此目的：

```java
StringBuilder builder = new StringBuilder();
builder.append(ch);
builder.append(str);
String completedString = builder.toString();
```


## 输入输出

### 格式化输出

Java 中提供了类似于 C 语言的 printf 方法：

```java
System.out.printf("Hello, %s. Next year, you'll be %d", name, age);
```

此外，还可以使用 String.format 方法创建一个格式化的字符串，而不打印输出：

```java
String message = String.format("Hello, %s. Next year, you'll be %d", name, age);
```

### 文件输出与输出

要想对文件进行读取，就需要用一个 File 对象来构造一个 Scanner 对象，如下所示：

```java
Scanner in = new Scanner(Path.get("myfile.txt"), "UTF-8");
```

要想写入文件，则需要狗仔一个 PrintWriter 对象：

```java
PrintWriter out = new PrintWriter("myfile.txt", "UTF-8");
```


## 大数值

如果基本的整数和浮点数精度不能够满足需求，那么可以使用 java.math 包中的两个很有用的类：`BigInteger` 和 `BigDecimal`，这两个类可以处理包含任意长度数字序列的数值。

使用 BigInteger.valueOf 方法可以将普通数值转换为大数值：

```java
BigInteger a = BigInteger.valueOf(100);
```

遗憾的是，你不能使用熟悉的 `+`, `*` 等算数运算符来处理大数值，而需要使用 `add`, `multiply` 等方法：

```java
BigInteger c = a.add(b);    // c = a + b
BigInteger d = c.multiply(b.add(BigInteger.valueOf(2)));    // d = c * (b + 2)
```


## 数组

Java 中使用如下方式来声明数组：

```java
int[] a;
```

上述语句只是声明了变量 a, 并没有将 a 初始化为一个真正的数组，应该使用 new 操作符来创建数组：

```java
int[] a = new int[100];
```

除了使用 for 循环来遍历数组，Java 还支持 for each 循环：

```java
for (int element : a)
    System.out.println(element);
```

另外还有如下比较方便的方式来初始化数组：

```java
int[] a = {1, 3, 5, 7, 9};
new int[] {1, 3, 5, 7, 9};
```

Java 中数组变量的拷贝是浅拷贝：

```java
int[] a = {1, 3, 5, 7, 9};
int[] b = a;    // a 和 b 引用了同一个数组
```

如果需要深拷贝，则应该使用 `Arrays.copyOf` 方法：

```java
int[] c = Arrays.copyOf(a, a.length);
```


## 类

Java 中，最简单的类定义形式为：

```java
class ClassName {
    field1
    field2
    ...
    constructor1
    constructor2
    ...
    method1
    method2
    ...
}
```

例如：

```java
class Employee {
    // instance fields
    private String name;
    private double salary;
    private LocalDate hireDay;

    // constructor
    public Employee(String n, double s, int year, int month, int day) {
        name = n;
        salary = s;
        hireDay = LocalDate.of(year, month, day);
    }

    // a method
    public String getName() {
        return name;
    }

    // more methods
    ...

    public void raiseSalary(double byPercent) {
        double raise = this.salary * byPercent / 100;
        this.salary += raise;
    }
}
```

如果你希望某个 field(C++ 中的成员变量) 一经设置就不再被改变，那么你可以用 `final` 关键字来修饰它：

```java
class Employee {
    private final String name;

    public Employee(String n) {
        // 必须在构造函数中初始化 final fields
        name = n;
    }
}
```

如果你希望某个 field 属于类而非实例(类似于 C++ 中的类成员变量)，那么你可以用 `static` 关键字来修饰它：

```java
class Employee {
    private static int nextId;

    private int id;

    public void setId() {
        // nextId 作为类成员变量，可以被所有实例访问
        id = nextId;
        nextId++;
    }

    // 静态方法
    public static int getNextId() {
        return nextId;
    }
}
```

### 对象析构与 finalize 方法

由于 Java 中有自动的垃圾回收器，不需要人工回收内存，所以 Java 不支持析构器。

如果某些对象使用了内存之外的其他资源，例如文件，那么当资源不再需要时，应该将其回收。

这时候，可以为类提供一个 finalize 方法。该方法将在垃圾回收器清除对象之前调用。

在实际情况中，不要依赖于使用 finalize 方法回收任何短缺的资源，因为你很难知道这个方法什么时候才会被调用。


## 传参

Java **总是** 采用值传递的方式来传参。也就是说，方法得到的是所有参数值的一个拷贝，因此，方法不能修改传递给它的任何参数变量的内容。

基本数据类型不用说，但对象作为参数时，传递的是对象的引用(引用作为值来传递给方法)：

```java
class Employee {
    public static void tripleSalary(Employee x) {
        x.raiseSalary(200);
    }
}
```

上述例子中 x 所表示的是 Employee 实例的引用，传参时，传递的是这个引用的值而非实例的值。

因此，下面的交换操作将无法正常工作：

```java
class Employee {
    public static void swap(Employee x, Employee y) {
        // 只是交换了 x, y 这两个引用的值，而不是交换它们指向的内容
        Employee Temp = x;
        x = y;
        y = temp;
    }
}
```

调用 `Employee.swap(x, y)` 之后，x 和 y 并不会交换，因为他们的引用是以值的方式来传递的，上述代码等价于以下 C++ 代码：

```cpp
class Employee {
public:
    static void swap(Employee* x, Employee* y) {
        // 只是交换 x y 这两个指针，并没有交换指针指向的内容
        Employee* temp = x;
        x = y;
        y = temp;
    }
}
```


## 包

Java 允许使用包 (package) 将类组织起来。使用包的主要原因是确保类名的唯一性，只要将同名的类放在不同的包中，就不会产生冲突。 Sun 公司建议将公司的因特网域名以逆序的形式作为包名，并且对于不同的项目使用不同的子包。

一个类可以使用它所属包中的所有类，以及其它包中的公有类(public class), 我们可以采用两种方式来访问另一个包中的公有类：

```java
// 使用完整包名
java.time.LocalDate today = java.time.LocalDate.now();

// 使用 import 语句导入特定的类或者整个包：
import java.time.LocalDate;
LocalDate today = LocalDate.now();
```

注意：import 与 C++ 的 `#include` 并没有共同之处。在 Java 中，如果你显式给出包名，如 java.util.Date, 就不必使用 import, 而在 C++ 中，你无法避免使用 `#include` 指令。

因此，import 的唯一作用是方便，它更像 C++ 中的 `using namespace`, 可以让你用更简短的名字而不是完整的包名来引用一个类。

### 将类放入包中

要想将一个类放入包中，就必须把包的名字放在源文件的开头，例如：

```java
package com.horstmann.corejava;

public class Employee {
    // ...
}
```

### 包作用域

我们之前的例子中，并没有为 Employee 类指定包作用域，因此它只能被同样包中的其他类访问。

你可以用 public 关键字来将 Employee 类指定为公有，这样其他包也可以访问这个类。

你可以用 private 关键字来将某个类指定为私有，这样的话这个类只能被定义它的类所访问(Java 也支持在类中定义类)。


## 继承

Java 中的类继承需要使用 extends 关键字：

```java
public class Manager extends Employee {
    private double bonus;

    public void SetBouns(double bonus) {
        this.bouns = bouns;
    }
}
```

值得注意的是，Java 中所有的继承都是公有继承，而不像 C++ 那样还有私有继承和保护继承。

另外，子类不能直接访问基类的 private fields:

```java
public class Manager extends Employee {
    public Manager(String name, double salary, int year, int month, int day) {
        // 使用 super 关键字来构造基类
        super(name, salary, year, month, day);
        bonus = 0;
    }

    // 复写基类的 getSalary 方法，
    public double getSalary() {
        // 不能直接访问基类中的 salary field, 只能通过基类的共有方法 getSalary 来访问。
        double baseSalary = super.getSalary();
        return baseSalary + bonus;
    }
}
```

注意，不需要使用 virtual 关键字来修饰 `getSalary`，在 Java 中，动态绑定是默认的处理方式。如果你希望让基类的某个方法不被动态绑定，那么你可以为它指定 final 关键字。

类似的，你还可以在定义类时指定 final 关键字，这将告知编译器，此类不希望被继承：

```java
public final class Executive extends Manager {
    // ...
}
```

### 抽象类

你可以使用 `abstract` 关键字来将类声明为抽象类，抽象类中可以包含一些没有实现的抽象方法：

```java
public abstract class Person {
    // 抽象类中也可以有 fields
    private String name;

    // 抽象类中也可以有构造函数
    public Person(String name) {
        this.name = name;
    }

    // 抽象类中也可以有成员函数
    private String getName() {
        return name;
    }

    // 抽象方法，无需实现
    public abstract String getDescription();
}
```

实际的操作中，抽象类应该表示类似于接口的概念，不应该有成员和普通方法。
