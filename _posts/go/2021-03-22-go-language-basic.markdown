---
layout: post
title:  "Go - 基础知识"
date:   2021-03-22 16:16:30 +0800
categories: go
---

* TOC
{:toc}


# 包

Go 程序都是由 `package` 构成的，类似于一个命名空间。

Go 中的每个程序，都需要在 `main` 这个特殊的 `package` 当中定义 `main()` 函数:

```go
package main

import "fmt"

func main() {
  fmt.Println("Hello, world.")
}
```

要访问包内导出的函数或变量，需要先使用 `import "fmt"` 这样的语法导入包，并在使用时加上包名，比如 `fmt.Println()`。你可以使用以下语法在一个 `import` 语句里导入多个包:

```go
package main

import (
  "fmt"
  "math/rand"
)

func main() {
  fmt.Println("My favorite number is", rand.Intn(10))
}
```

上述例子中，导入了 `math/rand` 包，按照约定，包名与导入路径的最后一个元素一致，所以在使用时只需要 `rand.Intn(10)` 就好了，不需要再加上前面的 `math/`。

此外，Go 语言当中，大写开头的符号将被导出，因此可以看到上述例子中使用到的 `Println()`, `Intn()` 等函数都是大写开头的。


# 函数

按照以下语法声明并使用函数:

```go
package main

import "fmt"

func add(x int, y int) int {
  return x + y
}

func main() {
  fmt.Println(add(42, 13))
}
```

如果两个参数的类型相同，可以省略一些类型声明:

```go
func add(x, y int) int {
  return x + y
}
```


## 多返回值

函数可以有多个返回值:

```go
package main

import "fmt"

func swap(x, y string) (string, string) {
  return y, x
}

func main() {
  a, b := swap("hello", "world")
  fmt.Println(a, b)    // world hello
}
```


## 命名返回值

函数的返回值可以被命名，命名返回值相当于在函数开头声明了一个变量并将其初始化为默认值。你可以在代码中为这个变量赋值，最后 `return` 它。看起来命名返回值的主要意义在于提高可读性，可以让调用方知道这个函数大概返回了啥东西:

```go
package main

import "fmt"

func split(number int64) (high, low int32) {
  high = int32(number >> 32)
  low = int32(number & 0xFFFFFFFF)
  return
}

func main() {
  fmt.Println(split(0x100000002))    // 1 2
}
```

上述代码中，没有参数的 `return` 语句返回已命名的返回值，这被称为 `直接返回`。直接返回语句应当仅用在上述短函数中，在长的函数中它们会影响代码的可读性。

并不是说使用了命名返回值语法的话，就一定要直接返回，你也可以像普通用法那样，写为 `return high, low`。


## defer 语句

defer 语句会在函数执行完毕后被调用，可以在这里做一些类似于清理的事情:

```go
package main

import "fmt"

func main() {
  defer fmt.Println("world")

  fmt.Println("hello")
}
```

可以有多个 defer 语句，这些语句会压入一个栈中，按照先入后出的顺序来执行:

```go
package main

import "fmt"

func main() {
  fmt.Println("counting")

  for i := 0; i < 10; i++ {
    defer fmt.Println(i)
  }

  fmt.Println("done")
}

// 输出:
// counting
// done
// 9
// 8
// 7
// 6
// 5
// 4
// 3
// 2
// 1
// 0
```


## 函数是值

函数是值，可以像其他值一样传递:

```go
package main

import (
  "fmt"
  "math"
)

func compute(fn func(float64, float64) float64) float64 {
  return fn(3, 4)
}

func main() {
  // 创建一个函数
  hypot := func(x, y float64) float64 {
    return math.Sqrt(x*x + y*y)
  }

  // 直接调用函数
  fmt.Println(hypot(5, 12))

  // 将函数作为参数传给另一个函数
  fmt.Println(compute(hypot))
  fmt.Println(compute(math.Pow))
}
```


## 闭包

函数是闭包的一种特殊情况，因此可以引用外部的值:

```go
package main

import "fmt"

func fibonacci() func() int {
  m := 0
  n := 1
  return func() int {
    result := m + n
    n = m
    m = result
    return result
  }
}

func main() {
  f := fibonacci()
  for i := 0; i < 10; i++ {
    fmt.Println(f())
  }
}
```


# 变量

使用 `var` 关键字来声明变量，变量可以声明在 `package` 范围或者函数范围:

```go
package main

import "fmt"

// package 范围的变量
var name string

func main() {
  // 函数范围的变量
  var age int

  fmt.Println(name, age)
}
```

## 零值

Go 语言当中的变量如果没有指定初始值，那么会被初始化为 "零值":
- 数值类型为 `0`
- 布尔类型为 `false`
- 字符串为 `""`（空字符串）


## 初始值

你可以使用以下语法来指定初始值，Go 会根据初始值来进行类型推导，所以这时候可以省略类型:

```go
package main

import "fmt"

var i, j int = 1, 2

func main() {
  var c, python, java = true, false, "no!"
  fmt.Println(i, j, c, python, java)  // 1 2 true false no!
}
```


## `:=` 语法

`:=` 语法用来简写 "声明变量并设置初始值" 这个操作:

```go
// 等价于 var k = 3
k := 3
```

注意 `:=` 语法只能用在函数里面，在 package 范围内不能使用。


## 基本类型

Go 语言中支持以下基本类型，下面几个比较特别:
- `uintptr`
- `rune`
- `complex64`, `complex128` : 用来表示复数，由实部和虚部构成。

```
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // uint8 的别名

rune // int32 的别名
    // 表示一个 Unicode 码点

float32 float64

complex64 complex128
```


## 对变量声明进行分组

可以使用以下语法来将一堆变量的声明放在一个组里，提高可读性:

```go
var (
  ToBe   bool       = false
  MaxInt uint64     = 1<<64 - 1
  z      complex128 = cmplx.Sqrt(-5 + 12i)
)
```


## 类型转化

使用 `T(v)` 这样的语法来将值 `v` 转换为类型 `T`，go 语言不会进行隐式转换，所有需要类型转换的地方都需要显示调用相应的转换方法。

```go
var i int = 42
var f float64 = float64(i)
var u uint = uint(f)

也可以简写为:
i := 42
f := float64(i)
u := uint(f)
```


## 常量

使用 `const` 关键字可以定义常量:

```go
package main

import "fmt"

const Pi = 3.14

func main() {
  const World = "世界"
  fmt.Println("Hello", World)
  fmt.Println("Happy", Pi, "Day")

  const Truth = true
  fmt.Println("Go rules?", Truth)
}
```


# 控制流

## for

`for` 循环的写法基本和 C 一样:

```go
package main

import "fmt"

func main() {
  sum := 0
  for i := 0; i < 10; i++ {
    sum += i
  }
  fmt.Println(sum)
}
```

你可以省略掉 初始化语句、后置语句，只留下条件语句，这时候的 `for` 就等同于 C 语言里面的 `while`，Go 没有单独提供 `while` 语句:

```go
package main

import "fmt"

func main() {
  sum := 1
  for sum < 1000 {
    sum += sum
  }
  fmt.Println(sum)
}
```

你也可以把条件语句也省略掉，这样就变成了无限循环:

```go
package main

import "fmt"

func main() {
  for {
  }
}
```


## if

Go 语言的 `if` 语句基本上和 C 语言一样:

```go
func sqrt(x float64) string {
  if x < 0 {
    return sqrt(-x) + "i"
  }
  return fmt.Sprint(math.Sqrt(x))
}
```

特别之处在于 Go 的 `if` 支持像 `for` 那样添加一个初始化语句，来声明一个只在 `if` 作用域内生效的变量:

```go
func pow(x, n, lim float64) float64 {
  if v := math.Pow(x, n); v < lim {
    return v
  } else {
    fmt.Printf("%g >= %g\n", v, lim)
  }
  // 这里开始就不能使用 v 了
  return lim
}rn lim
}
```


## switch

Go 语言的 `switch` 语法如下，`switch` 也能添加初始化语句。此外 Go 语言的 `switch` 与 C 语言相比有如下不同:
- 默认只会运行一个 `case`，相当于每个 `case` 自动加上 `break`，你可以通过 `fallthrough` 关键字来实现类似 C 语言的效果;
- `case` 不要求是整数，甚至不要求是常量。

```go
package main

import (
  "fmt"
  "runtime"
)

func main() {
  switch os := runtime.GOOS; os {
  case "darwin":
    fmt.Println("OS X.")
  case "linux":
    fmt.Println("Linux.")
  default:
    fmt.Printf("%s.\n", os)
  }
}
```

Go 中的 switch 要灵活一些，case 依次从上倒下逐个做判断，直到匹配成功时停止:

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  fmt.Println("When's Saturday?")
  today := time.Now().Weekday()
  switch time.Saturday {
  case today + 0:
    fmt.Println("Today.")
  case today + 1:
    fmt.Println("Tomorrow.")
  case today + 2:
    fmt.Println("In two days.")
  default:
    fmt.Println("Too far away.")
  }
}
```

Go 中的 switch 也可以不写条件，这种形式能够把一长串 if else 写的更清晰:

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  t := time.Now()
  switch {
  case t.Hour() < 12:
    fmt.Println("Good morning!")
  case t.Hour() < 17:
    fmt.Println("Good afternoon.")
  default:
    fmt.Println("Good evening.")
  }
}
```


# 指针

Go 里面也有指针的概念，与 C 语言的指针是类似的，其零值是 `nil`:

```go
package main

import "fmt"

func main() {
  i, j := 42, 2701

  p := &i           // 指针指向 i
  fmt.Println(*p)   // 通过指针读取 i 的值
  *p = 21           // 通过指针修改 i 的值
  fmt.Println(i)    // 21

  p = &j            // 指针指向 j
  *p = *p / 37      // 通过指针对 j 进行除法运算
  fmt.Println(j)    // 73
}
```

与 C 语言不同的是，Go 不支持指针运算:

```go
package main

func main() {
  i := 42

  p := &i
  p += 1      // 报错: invalid operation: p += 1 (mismatched types *int and int)
}
```


# 结构体

使用以下语法可以声明结构体:

```go
package main

import "fmt"

type Vertex struct {
  X int
  Y int
}

func main() {
  vertex := Vertex{1, 2}
  fmt.Println(vertex)
}
```

指针可以指向结构体，此时可以使用 `(*p).X` 来访问结构体的 `X` 字段，不过 Go 支持简写为 `p.X`，看起来结构体指针就像个普通的结构体实例一样。

```go
// ...

func main() {
  vertex := Vertex{1, 2}
  p := &vertex
  p.X = 1e9
  fmt.Println(vertex)
}
```

可以使用以下语法来创建一个结构体实例:

```go
// ...

func main() {
  var (
    v1 = Vertex{1, 2}   // 创建一个 Vertex 类型的结构体
    v2 = Vertex{X: 1}   // Y:0 被隐式地赋予
    v3 = Vertex{}       // X:0 Y:0
    p = &Vertex{1, 2}   // 创建一个 *Vertex 类型的结构体（指针）
  )

  fmt.Println(v1, v2, v3, p)
}
```


# 数组

总的来说 Go 当中 **数组** 的功能非常弱，其大小是在编译期确定好的，当然也不能支持 append 等操作。不过 Go 提供了 **切片** 这个功能，对数组操作做了封装，可以实现类似于 C++ 标准库中 `std::vector` 的功能。

类型 `[n]T` 表示拥有 n 个 `T` 类型的值的数组，Go 语言中的数组大小是 **不可变** 的。

```go
package main

import "fmt"

func main() {
  // 声明一个长度为 2 的字符串数组，并访问其元素
  var a [2]string
  a[0] = "Hello"
  a[1] = "World"
  fmt.Println(a)

  // 声明一个长度为 6 的整数数组，并初始化其元素
  primes := [6]int{2, 3, 5, 7, 11, 13}
  fmt.Println(primes)
}
```

Go 语言中的数组很简单，其大小必须是编译期常量，你没法在运行期直接创建出数组，必须依赖于 **切片**.


## 切片

类型 `[]T` 表示一个元素类型为 `T` 的切片。切片是数组的一个视图，由起止下标来界定：

```go
package main

import "fmt"

func main() {
  primes := [6]int{2, 3, 5, 7, 11, 13}

  // primes[1:4] 创建了一个切面，其范围为 [1 ~ 4)，这是个左闭右开区间
  var slice []int = primes[1:4]
  fmt.Println(slice)    // [3, 5, 7]
}
```

切片只是数组的视图，如果修改切片中的元素，数组元素也会被修改:

```go
package main

import "fmt"

func main() {
  names := [4]string {
    "John",
    "Paul",
    "George",
    "Ringo",
  }
  fmt.Println(names)

  a := names[0:2]
  b := names[1:3]
  fmt.Println(a, b)

  b[0] = "XXX"
  fmt.Println(a, b)
  fmt.Println(names)    // [John XXX George Ringo]
}
```

声明数组的时候必须指定大小，以下语法支持根据元素数量创建数组，随后再创建一个相应大小的切面，如果数组的元素内容是固定的话，使用下面的语法会比较简单(不用指定数组大小了):

```go
package main

import "fmt"

func main() {
  q := []int{2, 3, 5, 7, 11, 13}
  fmt.Println(q)

  r := []bool{true, false, true, true, false, true}
  fmt.Println(r)

  s := []struct {
    i int
    b bool
  } {
    {2, true},
    {3, false},
    {5, true},
    {7, true},
    {11, false},
    {13, true},
  }
  fmt.Println(s)
}
```

切片的下限默认是 0，上限默认是切片长度：

```go
package main

import "fmt"

func main() {
  s := []int{2, 3, 5, 7, 11, 13}

  s = s[:]
  fmt.Println(s)  // [2 3 5 7 11 13]

  s = s[1:4]
  fmt.Println(s)  // [3 5 7]

  s = s[:2]
  fmt.Println(s)  // [3 5]

  s = s[1:]
  fmt.Println(s)  // [5]
}
```

## 扩展切片

切片有 **长度** 和 **容量** 的概念，长度很容易理解，就是切片中所包含的元素个数；容量也比较简单，就是从切片的起始到数组的末尾的长度。容量这个东西主要是对切片的扩展进行限制，一个长度为 0 的切片可以把长度扩展到 10，前提是它有足够的容量。长度可以通过 `len()` 函数获取，容量可以通过 `cap()` 函数获取:

```go
package main

import "fmt"

func main() {
  s := []int{2, 3, 5, 7, 11, 13}
  printSlice(s)

  // 截取长度为 0 的切片，此时容量是 6
  s = s[:0]
  printSlice(s)

  // 扩展长度为 0 的切片，这时候容量不会变，还是 6, 因为起始位置没变
  s = s[:4]
  printSlice(s)

  // 舍弃前两个值，因为起始位置改变了，所以容量变成了 4
  s = s[2:]
  printSlice(s)
}

func printSlice(s []int) {
  fmt.Printf("len = %d, cap = %d %v\n", len(s), cap(s), s)
}
```

切片的零值是 `nil`，对一个值为 `nil` 的切片执行 `len()`, `cap()` 操作，都会得到 0:

```go
package main

import "fmt"

func main() {
  var s []int
  fmt.Println(s, len(s), cap(s))  // [] 0 0
  if s == nil {
    fmt.Println("nil!")           // nil!
  }
}
```


## 创建切片

前面提到 Go 语言在创建数组时只能使用编译期常量来指定数组大小。如果希望根据运行期变量来创建指定大小的数组，可以利用切面。Go 提供了一个 `make()` 函数来创建切片，它将根据你在运行期指定的大小创建一个数组，并用零值初始化数组元素，随后返回一个切面给你:

```go
package main

import "fmt"

func main() {
  size := 5

  s := make([]int, size)
  printSlice("s", s)    // s len=5 cap=5 [0 0 0 0 0]
}

func printSlice(s string, x []int) {
  fmt.Printf("%s len=%d cap=%d %v\n", s, len(x), cap(x), x)
}
```


## 切片的切片

切片可以包含任何类型，比如用切片的切片来表示二维数组:

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  // 用切片的切片来表示棋盘
  board := [][]string {
    []string{"_", "_", "_"},
    []string{"_", "_", "_"},
    []string{"_", "_", "_"},
  }

  // 修改棋盘中的元素
  board[0][0] = "X"
  board[2][2] = "O"
  board[1][2] = "X"
  board[1][0] = "O"
  board[0][2] = "X"

  for i := 0; i < len(board); i++ {
    fmt.Printf("%s\n", strings.Join(board[i], " "))
  }
}
```


## 向切片追加元素

可以使用 `append()` 函数向切片追加元素，前面提到过切片只是数组的一个视图，向切片追加元素，是否会影响到原先的数组呢？这取决于追加元素时是否超过了切片的容量，如果没有超过，那么向切片追加元素就相当于修改原始数组的元素值；如果超过了，那么 Go 会分配一个更大的新的数组，把原先数组的内容拷贝过来，这种情况下追加元素就不会影响原始数组了。

```go
package main

import "fmt"

func main() {
  numbers := []int{1, 2, 5, 7, 9}

  s := numbers[1:4]
  printSlice(s)         // [2 5 7] len=3, cap=4

  s = append(s, 8)
  printSlice(s)         // [2 5 7 8] len=4, cap=4
  printSlice(numbers)   // [1 2 5 7 8] len=5, cap=5
                        // 如果容量足够，那么向切片追加元素，将修改数组相应位置的元素

  s = append(s, 42)
  printSlice(s)         // [2 5 7 8 42] len=5, cap=8
  printSlice(numbers)   // [1 2 5 7 8] len=5, cap=5
                        // 容量不足，会分配一个更大的数组，这时候原先的数组就不受影响了
}

func printSlice(s []int) {
  fmt.Printf("%v len=%d, cap=%d \n", s, len(s), cap(s))
}
```


## 使用 range 访问切片

可以在 for 循环中使用 `range` 关键字来遍历切片，每次迭代可以获取 元素的下标 及 元素值的一份副本：

```go
package main

import "fmt"

var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
  for i, v := range pow {
    fmt.Printf("2**%d = %d\n", i, v)
  }
}
```

可以用占位符来忽略元素下标:

```go
for _, v := range pow {
  fmt.Printf("%d\n", v)
}
```


# Map

使用以下语法定义 map, map 的零值为 nil:

```go
// 定义一个 key 为 string, value 为 int 的 map
var m map[string]int
```

使用以下语法来创建并初始化 map:

```go
package main

import "fmt"

type Vertex struct {
  Latitude  float64
  Longitude float64
}

func main() {
  var m = map[string]Vertex{
    "Bell Labs": {40.68433, -74.39967},
    "Google":    {37.42202, -122.08408},
  }
  fmt.Println(m["Bell Labs"])
}
```

也可以使用 `make()` 函数，创建一个空的 map, 随后再填充值进去:

```go
func main() {
  m := make(map[string]Vertex)
  m["Bell Labs"] = Vertex{
    40.68433, -74.39967,
  }
  fmt.Println(m["Bell Labs"])
}
```

可以按照以下方法增删改查元素:

```go
package main

import "fmt"

func main() {
  m := make(map[string]int)

  // 增加元素
  m["Answer"] = 42
  fmt.Println("The value:", m["Answer"])

  // 修改元素
  m["Answer"] = 48
  fmt.Println("The value:", m["Answer"])

  // 删除元素
  delete(m, "Answer")
  fmt.Println("The value:", m["Answer"])

  // 获取元素，ok = true 表示元素存在，如果元素不存在，v 为零值
  v, ok := m["Answer"]
  fmt.Println("The value:", v, "Present?", ok)
}
```


# 方法

Go 语言里没有 "类"，只有结构体，但你可以为结构体添加方法，来实现类似于 "类" 的效果:

```go
package main

import (
  "fmt"
  "math"
)

type Vertex struct {
  X, Y float64
}

func (v Vertex) Abs() float64 {
  return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
  v := Vertex{3, 4}

  // 给结构体定义方法之后，就可以通过 v.Abs() 这种形式去调用方法了
  fmt.Println(v.Abs())
}
```

可以看到上述代码中，在定义结构体方法时需要在方法名之前增加一个 `(v Vertex)`，这个被称为 **接收者参数**。除此之外结构体方法和普通的函数没啥区别。


## 非结构体也可以添加方法

如以下代码所示，非结构体以外的其它类型也可以添加方法。不过添加的方法必须与接受者的类型定义在同一个 `package` 内。所以没法直接给 `float64` 添加方法，需要用 `type` 关键字转换一下:

```go
package main

import "fmt"

type MyFloat float64

func (f MyFloat) Abs() float64 {
  if f < 0 {
    return float64(-f)
  }
  return float64(f)
}

func main() {
  f := MyFloat(-3.1415926)
  fmt.Println(f.Abs())
}
```


## 为结构体指针添加方法

为结构体添加的方法里，操作的是结构体的 **副本** 而不是结构体本身，这意味着添加的方法对于结构体而言是只读的。如果希望添加的方法能够修改结构体的成员，可以按照下面的代码，为结构体指针添加方法:

```go
package main

import (
  "fmt"
  "math"
)

type Vertex struct {
  X, Y float64
}

func (v Vertex) Abs() float64 {
  return math.Sqrt(v.X * v.X + v.Y * v.Y)
}

func (v *Vertex) Scale(f float64) {
  v.X = v.X * f
  v.Y = v.Y * f
}

func main() {
  v := Vertex{3, 4}
  v.Scale(10)
  fmt.Println(v)
}
```

当然，即使你不需要修改结构体成员，也可以把方法添加到结构体指针上，这样可以避免结构体的拷贝。


# 接口

Go 语言中使用以下语法来定义 **接口**，接口是一种类型，可以用来声明变量，如果一个实例实现了该接口，就可以把这个实例赋值给接口变量:

```go
package main

import (
  "fmt"
  "math"
)

// 定义接口类型
type Abser interface {
  Abs() float64
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
  if f < 0 {
    return float64(-f)
  }
  return float64(f)
}

type Vertex struct {
  X, Y float64
}

func (v *Vertex) Abs() float64 {
  return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
  // 声明接口变量
  var a Abser

  // MyFloat 实现了 Abser 接口，因此 MyFloat 实例能够被赋值给 Abser 变量
  a = MyFloat(3.1415926)
  fmt.Println(a.Abs())

  // *Vertex 实现了 Abser 接口，因此 *Vertex 实例能够被赋值给 Abser 变量
  a = &Vertex{3, 4}
  fmt.Println(a.Abs())

  // 下面一行会报错:
  // Type does not implement 'Abser' as 'Abs' method has a pointer receiver
  //a = Vertex{3, 4}
}
```

上面代码的注释中说明，是 `*Vertex` 这个结构体指针添加了 `Abs()` 方法，而不是 `Vertex`，所以只有 结构体指针 才能被赋值给接口变量。


## 接口与隐式实现

Go 语言里还有个特别的地方，上述代码中 `MyFloat` 和 `*Vertex` 都没有直接跟 `Abser` 接口发生关联，你不需要像 Java 之类的语言那样显式地声明 `MyFloat implements Abser`，Go 认为只要一个类型实现了接口中定义的所有方法，那么这个类型就能够被赋值给接口变量。

换句话说，**实现不需要知道接口的存在，你可以在任何包中提供接口实现，而不需要引用接口定义代码**。


## 接口值

前面提到过，接口作为一种类型可以声明变量，接口变量也叫做接口值。在内部，接口值可以看做包含 值 和 类型 的一个元组:

```
(value, type)
```

通过接口值调用方法的时候，实际上是调用上述元组中 `type` 的同名方法。这意味着即使上述元组中的 `value` 为 nil, 调用也能成功:

```go
package main

import (
  "math"
)

type Abser interface {
  Abs() float64
}

type Vertex struct {
  X, Y float64
}

func (v *Vertex) Abs() float64 {
  if v == nil {
    return 0
  }
  return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
  // 声明一个 Vertex 指针，此时它为 nil
  var v *Vertex

  // 将 nil 传给 a, 此时 a 的内容为 (type: *Vertex, value: nil)
  var a Abser = v

  // Abs() 能够调用成功，只是传入的 *Vertex 为 nil
  a.Abs()
}
```


## 空接口

不包含任何方法的接口就是一个空接口，它可以保存任何类型的值:

```go
package main

import "fmt"

func main() {
  var i interface{}
  describe(i)

  i = 42
  describe(i)

  i = "hello"
  describe(i)
}

func describe(i interface{}) {
  fmt.Printf("(%v, %T)\n", i, i)
}
```


## 类型断言

Go 提供了以下语法来获取接口底层的具体值，类似于 C++ 的 `dynamic_cast` 向下转型操作:

```go
// 将接口 i 转换为实现 t
// 如果转换失败，将导致 panic
t := i.(T)

// 将接口 i 转换为实现 t, 转换失败则 ok 为 false, 不会导致 panic
t, ok := i.(T)
```

示例如下:

```go
package main

import (
  "fmt"
)

func main() {
  var i interface{} = "hello"

  s := i.(string)
  fmt.Println(s)

  s, ok := i.(string)
  fmt.Println(s, ok)

  f, ok := i.(float64)
  fmt.Println(f, ok)

  // 下面语句将导致 panic: interface conversion: interface {} is string, not float64
  f = i.(float64)
  fmt.Println(f)
}
```

也可以用 `switch` 语句来获取类型和值:

```go
package main

import "fmt"

func do(i interface{}) {
  switch v := i.(type) {
  case int:
    fmt.Printf("(%T, %v)\n", v, v)
  case string:
    fmt.Printf("(%T, %v)\n", v, v)
  default:
    fmt.Println("Unknown type!")
  }
}

func main() {
  do(13)      // (int, 13)
  do("hello") // (string, hello)
  do(3.14)    // Unknown type!
}
```

## 常见接口

### Stringer

Stringer 是个比较常见的接口，实现这个接口后，`fmt` 之类的包可以调用 `String()` 方法来打印值:

```go
type Stringer interface {
    String() string
}
```

示例如下:

```go
package main

import "fmt"

type IPAddr [4]byte

func (addr IPAddr) String() string {
  return fmt.Sprintf("%d.%d.%d.%d", addr[0], addr[1], addr[2], addr[3])
}

func main() {
  hosts := map[string]IPAddr{
    "loopback":  {127, 0, 0, 1},
    "googleDNS": {8, 8, 8, 8},
  }
  for name, ip := range hosts {
    fmt.Printf("%v: %v\n", name, ip)
  }
}
```

### error

`error` 是个内建接口，用来表示错误状态:

```go
type error interface {
    Error() string
}
```

我们想让函数返回错误码的话，首先要定义一个结构体，让这个结构体实现 `error` 接口，调用方调用我们的函数，拿到错误码，最后通过 `Error()` 方法获取错误信息:

```go
package main

import (
  "fmt"
  "time"
)

// 定义错误码结构体，并实现 error 接口
type JsonError struct {
  When time.Time
  What string
}

func (e *JsonError) Error() string {
  return fmt.Sprintf("at %v, %s", e.When, e.What)
}

// 在我们的函数里抛出 error
func parseNumber(text string) (float64, error) {
  return 0, &JsonError{time.Now(), "Not yet implemented"}
}

func main() {
  number, err := parseNumber("10")
  if err != nil {
    fmt.Println(err)
    return
  }

  fmt.Println(number)
}
```

### Reader

`io` 包指定了 `io.Reader` 接口，它表示从数据流的末尾进行读取，接口中有个 `Read()` 方法，它的参数是一个字节切片，`Read()` 方法会填充这个切片，然后把填充了的字节数返回给调用方，在遇到数据流结尾时，它会返回一个 `io.EOF` 错误:

```go
type Reader interface {
  Read(p []byte) (n int, err error)
}
```

Go 标准库里有许多 `Reader` 接口的实现，比如字符串、文件、网络连接、压缩、加解密之类的:

```go
package main

import (
  "fmt"
  "io"
  "strings"
)

func main() {
  r := strings.NewReader("Hello, Reader!")

  // 创建一个 8 字节的切片
  bytes := make([]byte, 8)
  for {
    // 读取后的结果会保存到切面里面
    n, err := r.Read(bytes)
    if err == io.EOF {
      break
    }

    fmt.Printf("%q\n", bytes[:n])
  }
}
```

有种常见的模式是一个 io.Reader 包装另一个 io.Reader，然后通过某种方式修改其数据流。比方说把一个字符串流包装到一个解密的流里面:

```go
package main

import (
  "io"
  "os"
  "strings"
)

type rot13Reader struct {
  r io.Reader
}

func (rot13 *rot13Reader) Read(bytes []byte) (int, error) {
   n, err := rot13.r.Read(bytes)
   if err != nil {
     return 0, err
  }

  // 按照 https://en.wikipedia.org/wiki/ROT13 解密
  for i := 0; i < n; i++ {
    switch  {
    case bytes[i] >= 'A' && bytes[i] <= 'M':
      fallthrough
    case bytes[i] >= 'a' && bytes[i] <= 'm':
      bytes[i] = bytes[i] + 13
    case bytes[i] >= 'N' && bytes[i] <= 'Z':
      fallthrough
    case bytes[i] >= 'N' && bytes[i] <= 'z':
      bytes[i] = bytes[i] - 13
    }
  }
  return n, nil
}

func main() {
  s := strings.NewReader("Lbh penpxrq gur pbqr!")
  r := rot13Reader{s}
  io.Copy(os.Stdout, &r)	// You cracked the code!
}
```


# 并发

## Goroutine

Go 程（goroutine）是由 Go 运行时管理的轻量级 "线程"。使用关键字 `go` 就可以启动并执行一个 goroutine:

```go
go f(x, y, z)
```

注意，f, x, y, z 的求值发生在当前 goroutine 中，而 f 的执行发生在新的 goroutine 中。

```go
package main

import (
	"fmt"
	"time"
)

func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func main() {
  // 在当前 goroutine 中执行 say 函数
	say("hello")

  // 在新的 goroutine 中执行 say 函数
	go say("world")
}
```


## Channel

Channel 是两个 goroutine 之间交换数据的管道:

```go
// 创建一个可以传输 int 类型数据的 channel
ch := make(chan int)

// 将 v 发送至信道 ch。
ch <- v

// 从 ch 接收值并赋予 v。
v := <-ch
```

默认情况下，发送和接收操作在另一端准备好之前会阻塞，看一个具体的例子：

```go
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}

  // 将和送入 channel
	c <- sum
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)

  // 从 c 中接收
	x, y := <-c, <-c

	fmt.Println(x, y, x+y)
}
```

在上面的例子中，首先启动了两个 goroutine, 它们会通过 channel 发送计算结果，如果接收端还没有准备好(`x, y := <-c, <-c` 还没有被调用)，则发送操作就会被阻塞。


### Channel 的缓冲区

Channel 可以带有缓冲区，及时接收端还没有准备好，发送端也可以把数据先放到缓冲区里，这样就不会阻塞发送端。上述例子中缓冲区的大小是 0，因此发送端被阻塞了。以下是一个使用了缓冲区的例子:

```go
package main

import "fmt"

func main() {
  // make 的第二个参数指定了缓冲区大小
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2
	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```

缓冲区只是延缓了阻塞发生的时间，如果缓冲区满了，那么发送端还是会被阻塞，如果缓冲区为空，那么接收端也还是会被阻塞。


### 关闭 channel

发送端如果没有数据要发送了，可以调用 `close(ch)` 函数来关闭 channel，接收端可以使用以下语法来判断管道是否被关闭了:

```go
// 管道被关闭的话 ok 为 false
v, ok := <-ch
```

使用 `for i := range ch` 可以不断从 channel 中接收数据，直到对方关闭 channel:

```go
package main

import (
	"fmt"
)

func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}

func main() {
	c := make(chan int, 10)

  // 启动一个 goroutine 来输出斐波那契数列
	go fibonacci(cap(c), c)

  // 不断从 channel 中获取数字，直到对方关闭
	for i := range c {
		fmt.Println(i)
	}
}
```


### 使用 select 语句等待多个 channel 被激活

`select` 语句使一个 Go 程可以等待多个通信操作。`select` 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行：

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}

    // 通过 quit 管道来停止 goroutine
		quit <- 0
	}()
	fibonacci(c, quit)
}
```


### select 语句的 default case

`default` 会在所有 case 都不满足，也就是所有 channel 都被阻塞的情况下被调用。所以它可以实现 `尝试从 channel 中读取数据，读不到的话就怎样怎样` 的逻辑:

```go
select {
case i := <-c:
    // 使用 i
default:
    // 从 c 中接收会阻塞时执行
}
```


## sync.Mutex 互斥量

如果我们不需要管道通信，只是想保证每次只有一个 Go 程能够访问一个共享的变量，可以使用 go 标准库提供的 `sync.Mutex` 这一数据结构，它提供了 `Lock()` 和 `Unlock()` 两个方法:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type SafeCounter struct {
	v   map[string]int
  // 通过 sync.Mutex 来保护 SafeCounter
	mux sync.Mutex
}

func (c *SafeCounter) Inc(key string) {
	c.mux.Lock()
	
  // Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	c.v[key]++

	c.mux.Unlock()
}

func (c *SafeCounter) Value(key string) int {
	c.mux.Lock()
  // 把 Unlock 放到 defer 里面，这样整个函数都会被 mutex 保护
	defer c.mux.Unlock()
	
  return c.v[key]
}

func main() {
	c := SafeCounter{v: make(map[string]int)}
	for i := 0; i < 1000; i++ {
		go c.Inc("somekey")
	}

	time.Sleep(time.Second)
	fmt.Println(c.Value("somekey"))
}
```


# 包管理

![]( {{site.url}}/asset/go-modules.svg )

- package: 同一个文件夹下的代码，属于同一个 package
- module: 一个类似于仓库、模块的东西，module 之间可以有依赖关系。module 的根目录有个 `go.mod` 文件，其中定义了当前 module 的一些信息，也定义了对其他 module 的依赖(`require` 关键字)
- 代码中可以通过 `import` 关键字导入其他 `package`

此外 Go 还提供了一些命令，方便对 module 进行操作:

```
// 创建一个名为 example.com/hello 的 module
go mod init example.com/hello

// 运行当前目录的 module
go run .

// 根据代码中的 `import` 关键字，修改 go.mod, 引入正确的依赖
go mod tidy
```