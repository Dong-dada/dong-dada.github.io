---
layout: post
title:  "Linux - bash 脚本"
date:   2021-03-18 20:23:30 +0800
categories: linux
---

* TOC
{:toc}


本文参考自 [Bash 脚本教程](https://wangdoc.com/bash/index.html)


# echo 命令

如果要输出多行文本，需要把多行文本放在引号里面:

```bash
$ echo "<HTML>
    <HEAD>
          <TITLE>Page Title</TITLE>
    </HEAD>
    <BODY>
          Page body.
    </BODY>
</HTML>"
```

`-n` 参数可以取消末尾的回车符，比如下面的例子，让两个 `echo` 命令的输出连在了一起，而不是分成两行:

```bash
$ echo -n a;echo b
ab
```

`-e` 参数会解释引号(双引号和单引号)里面的转义字符:

```bash
# 不加 -e
$ echo "Hello\nWorld"
Hello\nWorld

# 加上 -e
$ echo -e "Hello\nWorld"
Hello
World
```


# 一行里有多个命令

分号（;）是命令的结束符，使得一行可以放置多个命令，上一个命令执行结束后，再执行第二个命令。

```bash
$ clear; ls
```


# 命令的组合符 `&&` 和 `||`

`&&` 表示 如果 Command1 命令运行成功，则继续运行 Command2 命令。

```bash
# cat 命令执行失败，ls 命令就不会运行
$ cat filelist.txt && ls -l filelist.txt
cat: filelist.txt: No such file or directory
```

`||` 表示 如果 Command1 命令运行失败，则继续运行 Command2 命令。

```bash
# cat 命令执行失败，则执行 touch 命令创建文件
$ cat filelist.txt || touch filelist.txt
cat: filelist.txt: No such file or directory
```


# 快捷键

- `Ctrl + L`：清除屏幕并将当前行移到页面顶部。
- `Ctrl + C`：中止当前正在执行的命令。
- `Ctrl + U`：从光标位置删除到行首。
- `Ctrl + K`：从光标位置删除到行尾。


# 模式扩展

Shell 接收到用户输入的命令以后，会根据空格将用户的输入，拆分成一个个词元（token）。然后，Shell 会扩展词元里面的特殊字符，扩展完成后才会调用相应的命令。

Bash 是先进行扩展，再执行命令。因此，扩展的结果是由 Bash 负责的，与所要执行的命令无关。命令本身并不存在参数扩展，收到什么参数就原样执行。这一点务必需要记住。

模块扩展的英文单词是 `globbing`，这个词来自于早期的 Unix 系统有一个 `/etc/glob` 文件，保存扩展的模板。

模式扩展与正则表达式的关系是，模式扩展早于正则表达式出现，可以看作是原始的正则表达式。它的功能没有正则那么强大灵活，但是优点是简单和方便。

## 波浪线扩展

`~` 会自动扩展成当前用户的主目录:

```bash
# 进入 /home/me/foo 目录
$ cd ~/foo
```

`~+` 会扩展成当前目录，等同于 `pwd`:

```bash
$ cd ~/foo
$ echo ~+
/home/me/foo
```


## `?` 字符扩展

`?` 字符代表文件路径里面的任意单个字符，不包括空字符。比如，`Data???` 匹配所有 `Data` 后面跟着三个字符的文件名。

```bash
# 存在文件 a.txt 和 b.txt
$ ls ?.txt
a.txt b.txt
```

如果要指定多个字符，就要用多个问号，问号的数量和字符的数量是完全一样的:

```bash
# 存在文件 a.txt、b.txt 和 ab.txt
$ ls ??.txt
ab.txt
```

`?` 字符扩展属于文件名扩展，只有文件确实存在的前提下，才会发生扩展。如果文件不存在，扩展就不会发生:

```bash
# 当前目录有 a.txt 文件
$ echo ?.txt
a.txt

# 当前目录为空目录，扩展不起作用
$ echo ?.txt
?.txt
```


## `*` 字符扩展

`*` 字符代表文件路径里面的任意数量的任意字符，包括零个字符:

```bash
# 存在文件 a.txt、b.txt 和 ab.txt
$ ls *.txt
a.txt b.txt ab.txt
```

注意，`*` 不会匹配隐藏文件（以 `.` 开头的文件），即 `ls *` 不会输出隐藏文件。如果要匹配隐藏文件，需要写成 `.*`:

```bash
# 显示所有隐藏文件
$ echo .*
```

注意，`*` 字符扩展属于文件名扩展，只有文件确实存在的前提下才会扩展。如果文件不存在，就会原样输出。

```bash
# 当前目录不存在 c 开头的文件，扩展不起作用
$ echo c*.txt
c*.txt
```

`*` 只匹配当前目录，不会匹配子目录，如果想要匹配子目录，必须写成 `*/*.txt` 这样的形式，有几层子目录，就要写几层星号:

```bash
# 子目录有一个 a.txt
# 无效的写法
$ ls *.txt

# 有效的写法
$ ls */*.txt
```

`**` 能够匹配零个或多个子目录，因此 `**/*.txt` 可以匹配顶层的文本文件和任意深度子目录的文本文件。


## 方括号扩展

方括号扩展的形式是[...]，只有文件确实存在的前提下才会扩展。如果文件不存在，就会原样输出。它表示匹配括号之中的任意一个字符。比如，[aeiou]可以匹配五个元音字母中的任意一个:

```bash
# 存在文件 a.txt 和 b.txt
$ ls [ab].txt
a.txt b.txt

# 只存在文件 a.txt
$ ls [ab].txt
a.txt
```

如果要匹配隐藏文件，同时要排除 `.` 和 `..` 这两个特殊的隐藏文件，可以与方括号扩展结合使用，写成 `.[!.]*` (`.` 后面必须跟一个不是 `.` 的字符)。

方括号扩展还有两种变体：`[^...]` 和 `[!...]`。它们表示匹配不在方括号里面的字符，这两种写法是等价的。比如，`[^abc]` 或 `[!abc]` 表示匹配除了a、b、c以外的字符:

```bash
# 存在 aaa、bbb、aba 三个文件
$ ls ?[!a]?
aba bbb
```

注意，如果需要匹配 `[` 字符，可以放在方括号内，比如 `[[aeiou]`。如果需要匹配连字号 `-`，只能放在方括号内部的开头或结尾，比如 `[-aeiou]` 或 `[aeiou-]`。


## [start-end] 扩展

方括号扩展有一个简写形式 `[start-end]`，表示匹配一个连续的范围。比如，`[a-c]` 等同于 `[abc]`，`[0-9]` 匹配 `[0123456789]`:

```bash
# 存在文件 a.txt、b.txt 和 c.txt
$ ls [a-c].txt
a.txt
b.txt
c.txt

# 存在文件 report1.txt、report2.txt 和 report3.txt
$ ls report[0-9].txt
report1.txt
report2.txt
report3.txt
```

这种简写形式有一个否定形式 `[!start-end]`，表示匹配不属于这个范围的字符。比如，`[!a-zA-Z]` 表示匹配非英文字母的字符:

```bash
$ echo report[!1–3].txt
report4.txt report5.txt
```

## 大括号扩展

大括号扩展 `{...}` 表示分别扩展成大括号里面的所有值，各个值之间使用逗号分隔。比如，`{1,2,3}` 扩展成 `1` `2` `3`。

```bash
$ echo {1,2,3}
1 2 3

$ echo d{a,e,i,u,o}g
dag deg dig dug dog

$ echo Front-{A,B,C}-Back
Front-A-Back Front-B-Back Front-C-Back
```

另一个需要注意的地方是，大括号内部的逗号前后不能有空格。否则，大括号扩展会失效:

```bash
$ echo {1 , 2}
{1 , 2}
```

逗号前面可以没有值，表示扩展的第一项为空:

```bash
$ cp a.log{,.bak}

# 等同于
# cp a.log a.log.bak
```

大括号可以嵌套:

```bash
$ echo {j{p,pe}g,png}
jpg jpeg png

$ echo a{A{1,2},B{3,4}}b
aA1b aA2b aB3b aB4b
```

大括号也可以与其他模式联用，并且总是先于其他模式进行扩展:

```bash
$ echo /bin/{cat,b*}
/bin/cat /bin/b2sum /bin/base32 /bin/base64 ... ...

# 基本等同于
$ echo /bin/cat;echo /bin/b*
```

大括号可以用于多字符的模式，方括号不行（只能匹配单字符）:

```bash
$ echo {cat,dog}
cat dog
```

由于大括号扩展 `{...}` 不是文件名扩展，所以它总是会扩展的。这与方括号扩展 `[...]` 完全不同，如果匹配的文件不存在，方括号就不会扩展。这一点要注意区分:

```bash
# 不存在 a.txt 和 b.txt
$ echo [ab].txt
[ab].txt

$ echo {a,b}.txt
a.txt b.txt
```


## {start..end} 扩展

大括号扩展有一个简写形式 `{start..end}`，表示扩展成一个连续序列。比如，`{a..z}` 可以扩展成26个小写英文字母:

```bash
$ echo {a..c}
a b c

$ echo d{a..d}g
dag dbg dcg ddg

$ echo {1..4}
1 2 3 4

$ echo Number_{1..5}
Number_1 Number_2 Number_3 Number_4 Number_5
```

注意，如果遇到无法理解的简写，大括号模式就会原样输出，不会扩展:

```bash
$ echo {a1..3c}
{a1..3c}
```

这种简写形式可以嵌套使用，形成复杂的扩展:

```bash
$ echo .{mp{3..4},m4{a,b,p,v}}
.mp3 .mp4 .m4a .m4b .m4p .m4v
```

大括号扩展的常见用途为新建一系列目录:

```bash
# 新建36个子目录，每个子目录的名字都是”年份-月份“
$ mkdir {2007..2009}-{01..12}
```

这个写法的另一个常见用途，是直接用于 `for` 循环:

```bash
for i in {1..4}
do
  echo $i
done
```

这种简写形式还可以使用第二个双点 `{start..end..step}`，用来指定扩展的步长:

```bash
$ echo {0..8..2}
0 2 4 6 8
```

个简写形式连用，会有循环处理的效果:

```bash
$ echo {a..c}{1..3}
a1 a2 a3 b1 b2 b3 c1 c2 c3
```


## 变量扩展

Bash 将美元符号 `$` 开头的词元视为变量，将其扩展成变量值:

```bash
$ echo $SHELL
/bin/bash
```

变量名除了放在美元符号后面，也可以放在 `${}` 里面:

```bash
$ echo ${SHELL}
/bin/bash
```

`${!string*}` 或 `${!string@}` 返回所有匹配给定字符串 `string` 的变量名:

```bash
# 所有以 S 开头的变量名
$ echo ${!S*}
SECONDS SHELL SHELLOPTS SHLVL SSH_AGENT_PID SSH_AUTH_SOCK
```


## 子命令扩展

`$(...)` 可以扩展成另一个命令的运行结果，该命令的所有输出都会作为返回值:

```bash
$ echo $(date)
Tue Jan 28 00:01:13 CST 2020
```

还有另一种较老的语法，子命令放在反引号之中，也可以扩展成命令的运行结果:

```bash
$ echo `date`
Tue Jan 28 00:01:13 CST 2020
```


## 算术扩展

`$((...))` 可以扩展成整数运算的结果:

```bash
$ echo $((2 + 2))
4
```


## 字符类

`[[:class:]]` 表示一个字符类，扩展成某一类特定字符之中的一个。常用的字符类如下:

- `[[:alnum:]]` : 匹配任意英文字母与数字
- `[[:alpha:]]` : 匹配任意英文字母
- `[[:blank:]]` : 空格和 Tab 键。
- `[[:cntrl:]]` : ASCII 码 0-31 的不可打印字符。
- `[[:digit:]]` : 匹配任意数字 0-9。
- `[[:graph:]]` : A-Z、a-z、0-9 和标点符号。
- `[[:lower:]]` : 匹配任意小写字母 a-z。
- `[[:print:]]` : ASCII 码 32-127 的可打印字符。
- `[[:punct:]]` : 标点符号（除了 A-Z、a-z、0-9 的可打印字符）。
- `[[:space:]]` : 空格、Tab、LF（10）、VT（11）、FF（12）、CR（13）。
- `[[:upper:]]` : 匹配任意大写字母 A-Z。
- `[[:xdigit:]]` : 16进制字符（A-F、a-f、0-9）。

```bash
# 输出所有大写字母开头的文件名
$ echo [[:upper:]]*
```

字符类的第一个方括号后面，可以加上感叹号 `!`，表示否定。比如，`[![:digit:]]` 匹配所有非数字:

```bash
# 输出所有不以数字开头的文件名
$ echo [![:digit:]]*
```

字符类也属于文件名扩展，如果没有匹配的文件名，字符类就会原样输出:

```bash
# 不存在以大写字母开头的文件
$ echo [[:upper:]]*
[[:upper:]]*
```


## 量词语法

量词语法用来控制模式匹配的次数, 常用的语法有下面几个:

`?(pattern-list)` :匹配零个或一个模式。
`*(pattern-list)` :匹配零个或多个模式。
`+(pattern-list)` :匹配一个或多个模式。
`@(pattern-list)` :只匹配一个模式。
`!(pattern-list)` :匹配给定模式以外的任何内容。

```bash
# ?(.) 匹配零个或一个 .
$ ls abc?(.)txt
abctxt abc.txt

# ?(def) 匹配零个或一个 def
$ ls abc?(def)
abc abcdef

# +(.txt|.php) 匹配文件有一个.txt或 .php 后缀名
$ ls abc+(.txt|.php)
abc.php abc.txt

# +(.txt) 匹配文件有一个或多个 .txt 后缀名
$ ls abc+(.txt)
abc.txt abc.txt.txt

# !(b) 表示匹配单个字母b以外的任意内容，所以除了 ab.txt 以外，其他文件名都能匹配
$ ls a!(b).txt
a.txt abb.txt ac.txt
```

量词语法也属于文件名扩展，如果不存在可匹配的文件，就会原样输出:

```bash
# 没有 abc 开头的文件名
$ ls abc?(def)
ls: 无法访问'abc?(def)': 没有那个文件或目录
```







# 变量

使用以下语法声明变量，变量在整个脚本内可见，注意 `=` 两边不要有空格，否则会出错：

```bash
VAR="global variable"
echo $VAR
```

在函数内可以使用 `local` 关键字定义局部变量，局部变量只在函数内可见:

```bash
function sayHello {
  VAR="Aloha~~~"
  echo $VAR
}
sayHello
```

# 脚本参数

可以使用以下特殊变量来访问脚本参数:

|  | 说明|
|:--|:--|
| $0 | 当前脚本的文件名 |
| $n | 传递给脚本或函数的参数。n 是一个数字，表示第几个参数。例如，第一个参数是$1，第二个参数是$2。 |
| $# | 传递给脚本或函数的参数个数。 |
| $* | 传递给脚本或函数的所有参数。 |
| $@ | 传递给脚本或函数的所有参数。被双引号(" ")包含时，与 $* 稍有不同，下面将会讲到。 |
| $? | 上个命令的退出状态，或函数的返回值。 |
| $$ | 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。 |

脚本内容:

```bash
#!/bin/bash
echo "File Name: $0"
echo "First Parameter : $1"
echo "First Parameter : $2"
echo "Quoted Values: $@"
echo "Quoted Values: $*"
echo "Total Number of Parameters : $#"
```

运行结果:

```s
$./test.sh Zara Ali
File Name : ./test.sh
First Parameter : Zara
Second Parameter : Ali
Quoted Values: Zara Ali
Quoted Values: Zara Ali
Total Number of Parameters : 2
```


# 调用其他 shell 命令

在 bash 脚本中可以直接执行 shell 命令:

```bash
uname -o
```

执行结果:

```
GNU/Linux
```


# 获取用户输入

使用 `read` 关键字，可以用交互式的方式获取用户的输入:

```bash
#!/bin/bash

echo -e "Enter your email: \c"
read email

echo -e "Enter your password: \c"
read password

echo -e "\nYour email is $email, your password is $password"
```

执行结果:

```
Enter your email: dongyu_1991@outlook.com
Enter your password: 12345678

Your email is dongyu_1991@outlook.com, your password is 12345678
```