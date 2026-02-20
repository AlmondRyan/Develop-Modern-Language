# IR 设计

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/IR)

欢迎来到特别重要的一节！（其实每一节都特别重要，都是强耦合的）

在这节中，我们将探讨一个事—— IR 的设计。先提前说一下，我们设计的是：

- 高层IR：SSA（Static Single Assignment）样式
- 底层IR：类汇编、类三地址码的IR

本节不讨论 IR 生成（`IRBuilder`）等内容，只把重点放在 IR 的设计。

## 高层 IR（SSA IR）

### Hello World 例子

```
declare __builtin_print(string) -> void

global @str1 : string = "Hello World"

define @main() -> i32 {
entry:
    %0 = call __builtin_print(@str1)
    ret 0
}
```

整个 IR 分为三个部分：

- `declare` 部分：用于声明外部函数，也就是给多文件编译留一个口子，比如用到了其他地方的函数可以在这里 `declare` 声明，就像是 C++ 的前向声明
- `global` 部分：用于声明全局常量，如果数字是介于 16 位 `int` 的情况，我们把它直接优化为立即数而非常量，如果超出了 16 位 int，默认放到常量池
- `define` 部分：用于声明函数（实现在当前 Translation Unit 内），默认要求每个函数必须存在 `entry:` 标签作为函数入口，程序入口查找 `define @main`

`@` 开头的是标识符（如果是有名的变量/常量，则会用名称；无名常量会用类型缩写+id），而 `%` 开头的是寄存器。

让我们来一个一个看：

#### `declare` 部分

`declare` 部分 **只能** 声明外部函数和内置函数，我给你举个例子：

```
// Algorithm.rynt
declare(package) Algorithm; // 这里以后会有的，嗯对会有的
public int add(int a, int b) {
    return a + b;
}

// main.rynt
declare(package) Main;
import Algorithm;
public int main() {
    int result = add(3, 4);
}
```

这里，`main.rynt` 使用了 `add` 函数，但是 `add` 函数的定义并不在这里，所以我们要声明外部函数来告诉后端的字节码生成调用这个函数。生成的 IR 是这个样子的：

```
declare add(i32, i32) -> i32

define @main() -> i32 {
entry:
    %0 = call add(3, 4)
    ret 0
}
```

#### `global` 部分

`global` 部分对应的就是常量池，这里面声明的所有东西都会进入 VM 的常量池，还是看上文，整数超出 16 位 `int` 范围和其他的都会进入常量池。

标准的格式是这样的：

```
global @name : type = value
```

举个例子，你定义了 `string a = "abc"`，IR 将会转存到：

```
global @a : string = "abc"
```

#### `define` 部分

`define` 部分，非常明显的是定义函数的地方。每个函数的定义是：

```
define @name(argumentList) -> returnType {
label:
    content
}
```

其中：

- `@name` 还是那个
- `argumentList` 指的是参数列表，但是没有名字只有类型
- `returnType` 是函数的返回类型
- `label` 是标签，用于区分基本块。每个函数必定会有一个 `entry` 标签，用于查找函数入口。
- `content` 是实际的代码

## 底层 IR（Assembly IR）

高层 IR 虽然很好，但是对于虚拟机来说还是太高级了（尤其是那一堆 SSA 形式的寄存器 `%0`, `%1`...）。为了让虚拟机跑得更顺畅，我们需要把它再“降级”一次，变成更接近机器指令的底层 IR。

你可以把它理解为汇编语言，或者 Java 的字节码。

### Hello World 例子

让我们看看刚才那个 Hello World 降级之后变成了什么样（参考 `HelloWorld.rir`）：

```
section .cdata
    string @str1 = "Hello World"

section .code
_entry:
    func @main() -> i32 {
        load_const @str1
        syscall 0
        ret 0
    }
```

这就非常有意思了！它变成了基于栈（Stack-Based）的指令集（虽然我们这里看起来像是直接操作寄存器或者栈顶，但其实更接近栈机）。

### 为什么要有这个 IR？

你可能会问：为什么不直接解释执行高层 IR？

1.  消除 SSA：高层 IR 里的 SSA 形式虽然对优化很友好，但执行起来很麻烦（还要搞 Phi 节点什么的）。底层 IR 把它变成了线性的指令序列。
2.  指令选择：高层 IR 的 `call __builtin_print` 被翻译成了更具体的 `syscall 0`。这意味着我们可以在这一层做针对特定平台的指令选择。
3.  内存布局：你看它分成了 `.cdata`（常量数据段）和 `.code`（代码段），这已经非常接近可执行文件的结构了。

### 结构解析

#### `section .cdata`

这里存放所有的静态数据，比如字符串常量、全局变量等。这和高层 IR 的 `global` 部分很像，但更纯粹——只存数据。

#### `section .code`

这里是真正的代码指令。

- `_entry:`：程序的入口标签。
- `func @main ...`：函数定义的开始。
- `load_const @str1`：把常量 `@str1` 加载到栈顶（或者寄存器）。
- `syscall 0`：执行系统调用。在这里，0 号系统调用对应 `print`。这就把高层的函数调用变成了底层的系统指令。
- `ret 0`：函数返回。

### 设计思路

从高层 IR 到底层 IR 的转换过程，其实就是编译器后端的主要工作：

1.  指令选择 (Instruction Selection)：比如把 `call __builtin_print` 映射到 `syscall 0`。
2.  寄存器分配 (Register Allocation)：虽然我们的 VM 是栈式的（或者说无限寄存器），对于栈机来说，就是把操作数压栈、弹栈。
3.  布局 (Layout)：确定代码和数据在内存中的位置。

这个设计虽然简单，但麻雀虽小五脏俱全，涵盖了编译器后端最核心的概念。