---
layout: post
title:  "《软件调试》 学习 21 调试符号"
date:   2017-06-18 00:46:30 +0800
categories: debugging
---

* TOC
{:toc}

在软件调试中，调试符号 (Debug Symbols) 是将被调试程序的二进制信息与源程序信息联系起来的桥梁。很多重要的调试功能都必须要有调试符号才能够工作，比如源代码调试、栈回溯、按名称显示变量等。

从软件编译的角度看，调试符号是编译器在编译过程中，为支持调试而收录的调试信息。这些调试信息所描述的目标包括：变量、类型、函数、标号、源代码行等。

调试符号是在编译过程中逐步收集和提炼出来的，最后由链接器或专门工具保存到调试符号文件中。Visual Studio 编译器默认将调试符号保存到单独的文件中，即 PDB 文件。PDB 是 Program Database 的缩写，意思是用来描述源程序的数据库。

微软没有暴露 PDB 的结构，而是提供了两种方式来访问符号文件中的符号。一种是 DbgHelp 函数库，另一种是 DIA SDK(Debug Interface Access)。


## 名称修饰

一个完整的函数声明包括 返回值类型、调用协议名称、函数名称、参数信息等若干个部分。为了把这些信息都记录在一个字符串中以便标识和组织一个函数，VC 编译器使用了名称修饰 (Name Decoration) 技术。其宗旨就是将函数的本来名称、调用协议、返回值等信息按照一定规则编排成一个新的名字，称为修饰名称 (Decorated Name)。

例如下面分别是 TestTry 函数的原型和它的修饰名称：

```
int TestTry(HWND hWnd, int n);
?TestTry@@YAHPAUHWND__@H@Z
```

与修饰前的名称相比，修饰后的名称不再包含空格和括号这些不便于存储的分隔符，而是将多个部分合并成单一的连贯整体，因此名称修饰有时候也被称为名字碾平 (Name Mangling) 或名称粉碎。

观察编译生成的汇编文件可以看到修饰后的名称，你可以在 VS 中通过 Properities --> C/C++ --> OutputFiles --> Assembler Output 来告知编译器输出汇编文件。

另外，使用 DbgHelp 中的 UnDecorateSymbolName 函数可以把一个修饰过的符号名翻译称为本来的名字，不过这个函数不能从修饰名中解析出函数原型中的其他信息，如参数和返回值等。

### C 和 C++

首先 C 和 C++ 编译器使用的名称修饰规则是不同的，这意味着一个同样的函数，在 C 和 C++ 编译器中会修饰为不同的名称，例如 `void __cdecl test(void)` 函数，在 C 中会被修饰为 `_test`, 而按照 C++ 规范编译产生的修饰名称是 `?test@@ZAXXZ`。

因为链接器链接目标文件 (.obj 文件) 时是使用修饰名称来链接的，如果调用方和被调用方使用的编译规范不同，链接时就会出现以下错误：

```
error LNK2001: unresolved external symbol "void __cdecl test(void)" (?test@@ZAXXZ)
```

VC 编译器默认按照文件扩展名来选择对应的编译器，`.c` 后缀的文件使用 C 编译器，`.cpp` 或 `.cxx` 后缀的文件使用 C++ 编译器。不过你也可以使用如下的编译开关来手动选择编译器：
- /Tc 后面跟文件名，则强制该文件使用 C 编译器；
- /Tp 后面跟文件名，则强制改文件使用 C++ 编译器；
- /TC 指定所有文件都使用 C 编译器；
- /TP 指定所有文件都使用 C++ 编译器；

在 C++ 中， `extern "C"` 关键字生命使用 C 的名称修饰规则。

### C 的名称修饰规则

第一，对于使用 C 调用协议 `__cdecl` 的函数，在函数名称前加一个下划线；

第二，对于使用快速调用协议 `__fastcall` 的函数，在函数名称的前后各加一个 @ 符号，后面跟参数的长度，不考虑返回值。例如 `extern "C" int __fastcall Test(int n)` 修饰后的名称为 `@Test@4`;

第三，对于使用标准调用协议 `__stdcall` 的函数，在函数名称前加一个下划线，名称后面加一个 @ 符号，后面跟参数的长度，不考虑返回值。例如 `extern "C" int __stdcall Test(int n, int m)` 修饰后的名称为 `_Test@8`;

### C++ 的名称修饰规则

因为 C++ 要支持类和命名空间等特征，其修饰名称中必须考虑类名和命名空间信息，所以它的规则要复杂一些。

C++ 标准没有定义统一的名称修饰规则，因此不同编译器使用的规则是不一样的，即使是同一种编译器，不同版本间可能也会有差异。

以 VC++ 编译器为例，C++ 的名称修饰规则为：
1. 问号前缀；
2. 函数名称，构造函数和析构函数具有特别的名称，分别是 ?0 和 ?1, 运算符重载也具有特别的名称，例如 new, delete, =, +, ++ 的名称分别为 ?2, ?3, ?4, ?H, ?E;
3. 如果名称不是 ?0 这样的特殊函数名，那么加一个分隔符 @
4. 如果是类的方法，那么从所属类开始依次加上类名和父类名，每个类名后面跟一个 @ 符号。所有类名都加好之后，再加一个 @ 符号，接着加上字符 Q 或 S(静态方法)。如果不是类的方法，那么直接加上 @；
5. 调用协议的代码，C 调用协议的代码为 A，`__fastcall` 的代码为 I, `__stdcall` 的代码为 G。对于类方法，调用协议前面会加一个字符 A，this 调用协议的代码为 E；
6. 返回值编码，例如字符 H 表示整数类型的返回值；
7. 参数列表编码，以 @ 结束，细节从略；
8. 后缀 Z；

可以看出 C++ 的修饰名都是以 ? 开头，以 Z 结尾的，这可以作为与 C 修饰名区分的方法。


## 调试信息的存储格式

### COFF 格式

COFF 是 Common Object File Format 的缩写，是一种广为流传的二进制文件格式，用来存储可执行文件、目标文件、库文件等。Windows 的 PE 格式也是源于 COFF 格式的。

COFF 格式的调试信息可以和目标文件或映像文件保存在一起，也可以单独存放，取决于编译器的设置。

Windows 中使用 .dbg 文件来存放 COFF 格式的调试信息。

### CodeView(CV) 格式

CodeView 是与 MSC 编译器一起使用的调试器，它使用的调试符号格式被称为 CV 格式。其详细定义未公开。

与 COFF 格式类似，它可以和目标文件或映像文件保存在一起，也可以单独存放。

### PDB 格式

PDB 格式是 Visual C++ 1.0 引入的一种新的存储调试信息的格式。

PDB 格式的调试信息必须单独存放在一个文件中，通常以 .pdb 为后缀。PDB 格式的调试符号文件支持 Edit and Continue (EnC) 等高级调试功能，这是 CV 和 COFF 格式所不支持的。

根据包含符号的完整程度，PDB 格式的调试符号文件又分为私有 (private) 和 公有(public) 两种。私有 PDB 是由编译器和链接器产生的，公共 PDB 则是基于私有 PDB 剔除了数据类型、函数原型和源代码行等与源码关系密切的私有信息之后生成的。

公共 PDB 的典型用途是公开给客户或合作伙伴使用。例如微软符号服务器中的大多数符号文件都是公有 PDB。

公共 PDB 不支持源代码级调试、观察局部变量和按类型显示变量等功能。

### DWARF 格式

DWARF 是一种公开的调试信息格式规范，是由 DWARF 组织定义的。该格式主要用在 Unix 和 Linux 系统中。


## 目标文件中的调试信息

简单来说，目标文件 (Object File) 就是编译器用来存放目标代码 (Object Code) 的文件。目标文件既是编译过程的输出结果，又是链接过程的输入材料。

从文件角度来讲，编译器的任务是将源文件中的代码翻译成目标代码，然后存储到文件中；而链接器的任务是将多个目标文件中的目标代码链接成一个可以被操作系统加载和执行的映像文件 (Image File)。

VC 编译器的目标文件使用的是 COFF 格式，整个文件分成多个数据块，下图列出了一个典型目标文件的数据布局：

![]( {{site.url}}/asset/software-debugging-symbol-obj-file-layout.png )

可以看到该目标文件最前面是一个 `IMAGE_FILE_HEADER` 结构，之后是 5 个 `IMAGE_SECTION_HEADER` 结构，分别描述了该文件中 5 个节的概要信息，分别为：

- .drectve, 以空格分隔的链接选项，如 `-defaultlib:LIBCD -defaultlib:DNAMES`;
- .debug$S, 调试符号信息 (Symbol Information)；
- .bss, 自由格式的未初始化数据；
- .text, 目标代码；
- .debug$T, 调试类型信息 (Type Information)；

在头信息之后是节的数据，每个节最多有 3 种数据：原始数据 (raw data), 重定位信息 (relocation), 行号信息 (line number)。

从上图中可以看到， .drectve 节只有原始数据，.debug$S 有原始数据和重定位信息， .bss 节没有任何数据， .text 节有 3 种数据， .debug$T 只有原始数据。

在节信息之后是调试符号表和字符串表。

### IMAGE_FILE_HEADER 结构

以下是 `IMAGE_FILE_HEADER` 结构的定义：

```c
typedef struct __IMAGE_FILE_HEADER
{
    WORD Machine;                   // CPU 架构
    WORD NumberOfSections;          // 节数量
    DWORD TimeDateStamp;            // 时间戳
    DWORD PointerToSymbolTable;     // 符号表的偏移，如果没有符号表，则为 0
    DWORD NumberOfSymbols;          // 符号表中的符号数量
    WORD SizeOfOptionalHeader;      // 可选头结构的大小
    WORD Characteristics;           // 特征
}
```

### IMAGE_SECTION_HEADER 结构

`IMAGE_FILE_HEADER` 之后是连续多个 `IMAGE_SECTION_HEADER` 结构，其个数由 `IMAGE_FILE_HEADER` 的 NumberOfSections 字段指定。

每个 `IMAGE_SECTION_HEADER` 结构用来描述一个 节 (section) 的概要信息，包括名称、长度、数据的偏移。

本节开头的目标文件结构图中的右侧给出了 `IMAGE_SECTION_HEADER` 结构的定义。

### 节的重定位信息和行号信息

每个节有 3 种数据，原始数据、重定位信息、行号信息。

原始数据因节的不同而有不同的格式。

重定位信息用来描述链接和加载映像文件时应该如何修改节数据。每一条重定位信息是一个 `IMAGE_RELOCATION` 结构：

```c
typedef struct __IMAGE_RELOCATION
{
    UINT32 VirtualAddress;      // 需要重定位项目的 RVA 地址，也就是该数据相对于文件开头的偏移地址
    UINT32 SymbolTableIndex;    // 相关符号结构在符号表中的索引，通过它可以找到需要重定位的地址
    UNIT16 Type;                // 重定位类型
}IMAGE_RELOCATION;
```

行号信息用来描述源代码与目标代码的对应关系，每条源代码行信息是一个 `IMAGE_LINENUMBER` 结构：

```c
typedef struct __IMAGE_LINENUMBER
{
    union 
    {
        UINT32 SymbolTableIndex;    // 当 LineNumber 为 0 时，该字段有效，对应该函数名称的符号索引
        UINT32 VirtualAddress;      // 当 LineNumber 不为 0 时，该字段有效，表示目标代码相对于函数起始地址的偏移
    }
    UINT16 LineNumber;              // 相对于函数起始行号的偏移
}IMAGE_LINENUMBER;
```

### 存储调试数据的节

PE 规范定义了 4 种用于存储调试数据的节，分别如下：

- .debug$F: 用于存储函数的帧数据，包含多个 `FPO_DATA` 结构；
- .debug$S: 用于存储 VC 编译器专用的符号信息；
- .debug$P: 用于存储 VC 编译器专用的预编译信息；
- .debug$T: 用于存储 VC 编译器专用的类型信息；

### 调试符号表

在目标文件中，通常有一个用于存储符号信息的符号表，因为它使用的是 COFF 格式，因此它经常被称为 COFF 符号表 (COFF Symbol Table). 这个表的每个表项都是一个固定长度的 `IMAGE_SYMBOL` 结构：

```c
typedef struct _IMAGE_SYMBOL
{
    union
    {
        BYTE ShortName[8];  // 不超过 8 个字节的符号名称
        struct 
        {
            DWORD Short;    // 如果不为 0, 则 ShortName 为符号名
            DWORD Long;     // 长符号名的偏移地址
        } Name;
        DWORD LongName[2];  // 以 DWORD 方式来访问这个联合体
    } N;
    DWORD Value;            // 取值，与类型有关
    SHORT SectionNumber;    // 节的编号
    WORD Type;              // 符号类型
    BYTE StorageClass;      // 存储方式
    BYTE NumberOfAuxSymbols;// 附属符号的数量
} IMAGE_SYMBOL, *PIMAGE_SYMBOL;
```

### COFF 字符串表

在目标文件的 COFF 调试符号表之后通常是 COFF 字符串表。

字符串表中存放的字符串大多是大于 8 字节的符号名，因为 `IMAGE_SYMBOL` 结构中只能存储长度不超过 8 字节的符号名。如果符号名长度大于 8, 就会被存储到字符串表中。


## PE 文件中的调试信息

PE 格式的全称是 Portable Executable, 这个名字反映了 NT3.1 的一个重要设计目标，那就是可移植性，因为设计 NT 时，期望可以很容易把它移植到其他 CPU 架构中运行。

### PE 文件布局

PE 文件由固定结构的文件头和不确定个数的若干个数据块构成，下图显示了 HiWorld.exe 程序 (Debug 版本) 的文件结构：

![]( {{site.url}}/asset/software-debugging-symbol-pe-file-layout.png )

文件最前面的是 `IMAGE_DOS_HEADER` 结构，这个结构是为了兼容老的 DOS 操作而设计的，因此价值不大。

`IMAGE_NT_HEADER` 是 PE 文件的真正开始，由 3 个部分组成，最前面是 4 个字节的 PE 文件签名，其内容固定为 `PE\0\0`, 签名之后是一个 `IMAGE_FILE_HEADER` 结构，这与目标文件的结构是相同的，接下来是一个 `IMAGE_OPTIONAL_HEADER` 结构。

在 `IMAGE_OPTIONAL_HEADER` 结构之后，是一系列 `IMAGE_SECTION_HEADER` 结构，用来描述 PE 文件中各个节 (section) 的情况，常见的节有以下几种：

- .text: 代码；
- .rdata: 只读的已经初始化的数据 (Read-only initialized data)；
- .data: 数据；
- .idata: 输入数据 (Import data)；
- .rsrc: 资源，比如 GUI 程序的菜单、图标、位图等，也可以存放 manifest 文件等；

Release 版本的结构与 Debug 版本类似，只是会将 .idata 节的内容合并到 .data 节中，以减小 EXE 文件的大小：

![]( {{site.url}}/asset/software-debugging-symbol-pe-file-layout-release.png )

### IMAGE_OPTIONAL_HEADER 结构

`IMAGE_OPTIONAL_HEADER` 是 PE 文件中非常重要的一个结构，其中包含了 PE 文件的许多属性，这些属性对于指导操作系统如何加载这个 PE 文件起着重要的作用：

![]( {{site.url}}/asset/software-debugging-symbol-image-optional-header-struct.png )

其中，DataDirectory 字段是一个数组，用于定位导入表、导出表、资源表、异常表、调试信息表、签名表、TLS 和 延迟加载描述符表等其他数据表。它的每一个元素都是一个 `IMAGE_DATA_DIRECTORY` 结构：

```c
typedef struct _IMAGE_DATA_DIRECTORY
{
    DWORD VirtualAddress;   // 表的起始位置 RVA
    DWORD Size;             // 表的大小
}IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

下表列出了 DateDirectory 数组中 0~15 号元素所描述的目录表和它在 HiWord.exe (调试版本) 文件中的取值：

![]( {{site.url}}/asset/software-debugging-symbol-image-data-directory.png )

### 调试数据目录

PE 文件可以有一个可选的调试目录 (Debug Directory), 用来描述 PE 文件中所包含的调试信息的种类和位置。调试目录的位置可以是在 .debug 节中，也可以位于其它节中，或者不在任何节中。`IMAGE_OPTIONAL_HEADER` 中的 DateDirectory[6] 字段用来记录它的位置和长度。

调试目录可以包含多个固定长度的 `IMAGE_DEBUG_DIRECTORY` 结构，每个结构描述一个调试信息数据块：

```c
typedef struct _IMAGE_DEBUG_DIRECTORY
{
    DWORD Characteristics;  // 保留
    DWORD TimeDateStamp;    // 调试数据的产生时间
    DWORD MajorVersion;     // 调试数据格式的主版本号
    DWORD MinorVersion;     // 调试数据格式的次版本号
    DWORD Type;             // 调试信息的类型
    DWORD SizeOfData;       // 调试数据的长度
    DWORD AddressOfRawData; // 调试数据的内存地址
    DWORD PointerToRawData; // 调试数据在文件中的偏移，即 RVA
} IMAGE_DEBUG_DIRECTORY, *PIMAGE_DEBUG_DIRECTORY;
```

![]( {{site.url}}/asset/software-debugging-symbol-image-debug-directory-type.png )

### 调试数据

根据调试目录结构 `IMAGE_DEBUG_DIRECTORY` 中的信息可以找到各个调试数据块，其方法是将 PointerToRawData 字段所指定的文件偏移加上 PE 模块的基地址。

![]( {{site.url}}/asset/software-debugging-symbol-debug-data.png )

其中，前 4 个字节是 CodeView 数据的签名，RSDS 代表这个数据块是 `CV_INFO_PDB70` 格式的。RSDS 之后的 16 个字节是 GUID，而后的 `01 00 00 00` 是 Age 字段，剩下的便是字符串 PdbFileName. 

RSDS 或 NB10 类型的 CodeView 数据块是 PE 文件中最常见的调试数据，使用 VC 编译器构建的调试版本的各种 PE 文件默认都包含这个数据块。对于 Release 版本，需要在链接选项中设置 产生调试信息 (Generate Debug Info) 选项，才会包含有这个数据块。

现在的 VC 编译器都是用单独的 PDB 文件来存储调试信息，真正的调试信息都存储在 PDB 文件中。PE 文件中只保留了这个 RSDS 数据块来索引 PDB 文件就够了。

### 调试信息的产生过程

PE 文件中调试信息的产生过程，概括起来可以分为如下三个阶段：

- **收集阶段**：编译器在编译源文件的过程中收集调试信息，存放到目标文件中；
- **集成阶段**：链接器在链接目标文件的过程中，将分布在各个目标文件中的调试信息集成到 PE 文件中；
- **可选的调整压缩阶段**：如果调试信息的格式是 CodeView 格式，那么在链接的最后阶段执行一个名为 CVPACK.exe 的程序，将调试信息进行压缩和整理；

下图画出了产生调试信息的主要步骤：

![]( {{site.url}}/asset/software-debugging-symbol-generate-pe-debug-info.png )

