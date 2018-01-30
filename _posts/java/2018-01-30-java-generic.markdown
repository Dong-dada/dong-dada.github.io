---
layout: post
title:  "Java 泛型"
date:   2018-01-30 16:38:30 +0800
categories: java
---

* TOC
{:toc}

## 简单泛型

```java
/**
 * 泛型类示例
 */
public class Pair<T> {
    private T first;
    private T second;

    public Pair() {
        first = null;
        second = null;
    }

    public Pair(T first, T second) {
        this.first = first;
        this.second = second;
    }

    public T getFirst() {
        return first;
    }

    public T getSecond() {
        return second;
    }

    public void setFirst(T value) {
        this.first = value;
    }

    public void setSecond(T value) {
        this.second = second;
    }
}

public class Foo {
    public static void main(String[] args) {
        // 使用泛型类
        Pair<String> pair1 = new Pair<String>("dong", "dada");
        // 可以省略类型参数
        Pair<String> pair2 = new Pair("dong", "dada");

        // 使用泛型方法
        String[] words = {"hello", "haha",　"..."};
        String str = Foo.<String>min(words);
        // 可以省略类型参数
        String str2 = min(words);
    }

    /**
     * 泛型方法示例
     */
    public static <T> T min(T[] objs) {
        if (objs == null || objs.length == 0) {
            return null;
        }

        T minObj = objs[0];
        for (int i = 1; i < objs.length; ++i) {
            if (minObjs.compareTo(objs[i]) > 0) {
                minObj = objs[i];
            }
        }

        return minObj;
    }
}
```

要定义一个泛型类，则使用上述 `class Pair<T>` 的语法，尖括号中的内容即为类型变量。你可以定义多个类型变量，中间用逗号隔开。比如 `class Pair<T, U>`。

> Java 中经常使用比较短的大写字母来表示类型变量。一般使用 E (Element)来表示集合的元素类型，K 和 V 用来表示关键字和值的类型。 T(需要时还可以使用临近的字母 U 和 S) 表示 “任意类型”。

泛型方法的定义也是类似的，使用 `<T> T foo(T obj)` 这样的形式来定义方法即可。


## 类型变量的限定

上述例子中，我们希望传给 `min` 的类型应该实现 compareTo 接口，以便进行比较。如果希望对类型变量加以约束，那么可以通过如下方式来进行限定：

```java
public static <T extends Comparable> T min(T[] objs) {
    // ...
}
```

`<T extends Comparable>` 这行代码限定了传入的类型 T 必须实现 `Comparable` 接口。

如果希望限定类型实现多个接口，那么可以用 `&` 来连接多个类型：

```java
T extends Comparable & Serializable
```


## 类型擦除

从上述代码来看，Java 的泛型跟 C++ 的模板很类似。但从底层原理来看，它们是完全不同的。

Java 并不是在编译期把泛型类翻译成各种各样的类，而是进行了类型擦除。

例如之前的 Pair 类和 min 方法，经过编译后，它会被翻译为：

```java
public class Pair {
    // T 的类型被擦除，替换为限定类型 Object
    private Object first;
    private Object second;

    public Pair(Object first, Object second) {
        this.first = first;
        this.second = second;
    }

    public Object getFirst() {
        return first;
    }

    public Object getSecond() {
        return second;
    }

    public void setFirst(Object first) {
        this.first = first;
    }

    public void setSecond(Object second) {
        this.second = second;
    }
}

public class Foo {

    public static void main(String[] args) {
        Pair pair1 = new Pair("dong", "dada");
        Pair pair2 = new Pair("dong", "dada");

        String[] words = {"hello", "haha",　"..."};
        String str = min(words);
        String str2 = min(words);
    }

    // T 的类型被擦除，替换为限定类型 Comparable
    public static Comparable min(Comparable[] objs) {
        // ...
    }
}
```

经过编译后，Pair 类中的 T 都被擦除，替换为原始类型 Object. 如此一来 Pair 就可以接收各种各样的类型了(除了基本类型)。代码中使用的 `Pair<String>`, `Pair<LocalDate>` 实际上都是同一个类 `Pair`，其内部使用 Object 来存储不同类型的示例。

对于 `min` 而言，因为它已经进行了类型限定 `T extends Comparable`, 因此 T 会被擦除，用 Comparable 来替换。

### 翻译泛型表达式

紧接着就带来一个问题，当类型被擦除之后，如何从 Object 或 Comparable 来向下转型得到实际的类型 String 或 LocalDate 呢？

答案是 Java 会帮你做泛型表达式的翻译。比如你调用如下代码：

```java
Employee buddy = buddies.getFirst();
```

其中 `buddies.getFirst()` 会返回一个 Object 类型的变量。编译器会自动帮你插入 Employee 的强制类型转换。

### 桥方法

除了上述翻译外，Java 还会自动生成 *桥方法(bridge method)* 来帮你保障多态的正确性。

比如 `Pair.setSecond()` 方法。经过类型擦除之后会变为 `Pair.setSecond(Object value)`

假设你从 Pair 继承了一个新的类 `DateInterval` :

```java
public class DateInterval extends Pair<LocalDate> {
    public void setSecond(LocalDate second) {
        // 只有在 second 大于 first 的时候才允许设置。
        if (second.compareTo(getFirst()) >= 0) {
            super.setSecond(second);
        }
    }
}

// 上述代码等价于
public class DateInterval extends Pair {
    // ...
}
```

因为类型擦除的存在，这时候会存在两个 `setSecond` 方法：

```java
// Pair 类中经过类型擦除之后的方法
void setSecond(Object second);

// DateInterval 中实现的方法，它的原本意思是覆盖父类的 setSecond(LocalDate second)
void setSecond(LocalDate second);
```

注意，因为参数类型不同， `DateInterval.setSecond` 是一个新的方法，没有覆盖 `Pair.setSecond`，因此无法表现出多态行为。

当我们希望以多态的方式来调用时：

```java
DateInterval interval = new DateInterval(...);
Pair<LocalDate> pair = interval;    // 把 interval 赋值给基类引用 pair
pair.setSecond(aDate);              // 使用基类引用 pair 来调用 setSecond
```

上述例子中，我们用基类引用 `pair` 来调用 `setSecond`，此时只会调用 `Pair.setSecond` 而不会调用 `DateInterval.setSecond` (因为它是一个新的方法，不具备多态性，或者说，它不在虚表里面)。

为了解决这一问题，Java 编译器会在 `DataInterval` 中生成一个桥方法：

```java
public void setSecond(Object second) {
    setSecond((Date)second);
}
```

桥方法具有和 `Pair.setSecond` 一致的参数，因此具备多态性。


## 约束与局限性

**约束一：** 不能用基本类型实例化类型参数： 也就是说，不能写 `Pair<double>` 这样的代码。因为类型擦除后 Pair 会用 Object 来存数据，但基本类型不继承 Object。

**约束二：** 运行时类型查询只适用于原始类型： 也就是说，在运行时查询 `Pair<String>` 或 `Pair<LocalDate>` 的类型，得到的结果都是 `Pair`，而无法知道具体的类型是什么。

```java
Pair<Double> pair = new Pair<>(3.141, 2.718);

// 编译错误，Pair<Double> 不能作为 instanceof 的参数
if (pair instanceof Pair<Double>) {}

// 只能判断是否是 Pair
if (pair instanceof Pair) {}

// 编译警告，只能判断 obj 是否是 Pair 类型，无法确定它是否是 Pair<String>
Object obj = pair;
Pair<String> pair2 = (Pair<String>)obj;

// 将返回 true, 尽管类型不同，但经过擦除后，getClass 返回的都是 Pair 类。
Pair<String> pair3 = new Pair<>("hello", "dada");
if (pair.getClass() == pair3.getClass()) {};
```

**约束三：** 不能创建参数化类型的数组：也就是说，下面的代码将导致编译错误：

```java
Pair<String>[] table = new Pair<String>[]; // error
```

如果允许上述代码编译通过，那么会引起以下问题：

```java
Pair<String>[] table = new Pair<String>[10];

Object[] objs = table;

// 把 Pair<Double> 赋值给了 Pair<String>
objs[0] = new Pair<Double>(3.141, 2.718);
```

为了避免上述问题，Java 不允许创建参数化类型的数组。

不过可以使用 `ArrayList<Pair<String>>` 这样的形式来创建 `Pair<String>` 的集合：

```java
ArrayList<Pair<Double>> table = new ArrayList<>();
table.add(new Pair<Double>(3.141, 2.718));

// 将引起编译错误，无法把 table 转换为 ArrayList<Object> 类型
ArrayList<Object> objs = table;
```

编译期不允许把 `ArrayList<Pair<Double>` 转换为 `ArrayList<Object>`, 因此不会有之前提到的问题。

需要注意的是，只是不能写 `new Pair<String>[10]`, `Pair<String>[] table;` 这样的声明还是可以的。

**约束三：** 不能实例化类型变量： 也就是说，不能调用 `new T()` 或 `new T[]` 来实例化类型变量。

其原因也是类型擦除， `new T()` 会被擦除为 `new Object()`, 这显然不符合我们的预期。

Java SE 8 之后比较好的解决方法是让调用者提供一个构造器表达式，比如：

```java
class Pair <T> {
    public static <T> Pair<T> makePair(Supplier<T> constr) {
        return new Pair<>(constr.get(), constr.get());
    }
}

Pair<String> p = Pair.makePair(String::new);
```

在 Java SE 8 之前，比较传统的方式是通过 Class.newInstance 方法来构造泛型对象：

```java
public static <T> Pair<T> makePair(Class<T> cl) {
    try {
        return new Pair<>(cl.newInstance(), cl.newInstance());
    } catch (Exception ex) {
        return null;
    }
}

Pair<String> p = Pair.makePair(String.class);
```

**约束四：** 泛型类的静态上下文中类型变量无效： 也就是说，你不能在静态域或方法中引用类型变量。比如下面的代码无法达到预期目的：

```java
public class Singleton<T> {
    private static T singleInstance;    // 编译错误

    public static T getSingleInstance() {
        if (singleIntance == null) {
            // 构造 singleInstance
        }

        return singleInstance;
    }
}
```

其原因还是类型擦除，经过类型擦除后，实际上只会有一个 `Singleton` 类，所以 `Singleton<String>` 和 `Singleton<Manager>` 都会共享同一个类，这显然不对劲，所以 Java 不允许类型变量 T 用在静态域或静态方法中。


## 泛型类型的继承规则

考虑两个类 Base 以及继承自它的 Derived。需要注意的是 `Pair<Base>` 和 `Pair<Derived>` 之间没有继承关系。更具体来说，**不管 S 和 T 之间有什么联系，`Pair<S>` 和 `Pair<T>` 通常都没有什么联系**。

之所以有上面的限制，是因为一旦允许 `Pair<Base>` 和 `Pair<Derived>` 的转换，就会有一些奇怪的事情发生：

```java
Pair<Derived> derivedPair = new Pair<>();

// 假设可以进行 Pair<Derived> 到 Pair<Base> 的转换
Pair<Base> basePair = derivedPair;

// 把 base 对象设置到 pair 上
basePair.setFirst(new Base());

// base 对象被强转为 Derived
Pair<Derived> first = derivedPair.getFirst();
```

然而数组却可以进行类型转换：

```java
Derived[] derivedArray = ...;
Base[] baseArray = derivedArray;
```

然而，当你给 baseArray 赋值一个 Derived 对象的时候，会抛出一个 `ArrayStoreException` 异常。

上述内容所说的是，同一个泛型类的不同类型之间没有继承关系。但泛型类本身是可以扩展另一个泛型类的：

```java
class Base <T> {

}

class Derived <T> extends Base<T> {

}

public class Main {
    public static void main(String[] args) {
        Derived<String> derived = new Derived<>();
        Base<String> base = derived;
    }
}
```

`Derived<String>` 继承自 `Base<String>`。

类似的，`ArrayList<T>` 继承自接口 `List<T>`。因此 `ArrayList<Derived>` 和 `List<Derived>` 有继承关系。但 `ArrayList<Base>` 和 `ArrayList<Derived>` 没有关系。


