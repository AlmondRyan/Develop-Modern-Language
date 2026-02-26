# VM (虚拟机) 和整个编译链路

> 本节的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/VM)

欢迎来到 Hello World 章节的最后一个部分 —— VM（Virtual Machine，虚拟机）！

因为我们的语言是基于 IR 转字节码然后在 VM 中执行（类似于 Python 的 CPython，JavaScript 的 V8），所以设计一个通用的 VM 尤为重要。

我们在这一章还是不涉及到任何关于 GC 的内容，只是简简单单的让他跑起来即可，GC 以后再说。

## VM 设计

我们的虚拟机采用了经典的栈机模型——所有的运算都在一个共享的栈上进行。想象一下手持计算器输入表达式时的感觉：输入数字，按下运算符，结果出现在栈顶——对，就是这个感觉。

整个虚拟机由四个核心组件构成，每个组件都有自己明确的职责：

### 栈

栈是虚拟机最繁忙的地方。当执行 `1 + 2` 时，`1` 和 `2` 会被压入栈，然后加法指令从栈中弹出这两个数，计算结果 `3`，最后把 `3` 再压回栈顶。就是这么简单。

```Cpp
class Stack {
    // 简化后的代码，完整版见代码仓库
    void push(RuntimeValue val) { data.push_back(std::move(val)); }
    RuntimeValue pop() { /* ... */ }
    
private:
    std::vector<RuntimeValue> data;  // 为什么用 vector 而不是 stack？
};
```

这里用 `std::vector` 而不是 `std::stack` 是有原因的：`std::stack` 只提供了最基本的栈操作，但我们还需要查看栈顶元素、清空栈等操作。更重要的是，`vector` 在内存中是连续存储的，对缓存更友好，这对频繁的压栈弹栈操作来说意味着更好的性能。

### 运行时值

虚拟机要处理的不只是整数，还有字符串、布尔值，未来还会有对象、数组等。如何安全地存储和操作这些不同类型的值？

```Cpp
struct RuntimeValue {
    std::variant<int32_t, std::string, std::monostate> value;
    // ...
};
```

这里故意用了结构体而不是类，因为 `RuntimeValue` 本质上就是个数据的"容器"，不需要复杂的封装。`std::variant` 就像一个"类型安全的 `union`"，让我们能优雅地处理不同类型的值。

### 帧

当函数调用发生时，需要记住很多信息：参数传进来了吗？局部变量存在哪？执行到哪条指令了？帧（Frame）就是为每个函数调用保存这些上下文的"记忆体"。

```Cpp
class Frame {
    // 每个帧保存着函数的局部变量、参数和当前执行位置
    std::map<std::string, RuntimeValue> locals;  // 局部变量表
    std::vector<RuntimeValue> args;              // 函数参数
    Instruction* currentInstruction;              // 当前执行到哪了
};
```

这样，当函数 A 调用函数 B 时，我们只需要创建一个新的帧压入调用栈，函数 B 执行完后再切回 A 的帧，一切都能无缝继续。

### VM

最后，我们需要一个总指挥来协调这些组件：

```Cpp
class VM {
    Stack stack;                          // 运行时栈
    std::vector<std::unique_ptr<Frame>> callStack;  // 调用栈
    std::map<std::string, IntrinsicHandler> intrinsics;  // 内置函数
    // ...
};
```

VM 负责指令的解析和执行、函数调用的调度、内置函数的管理等核心工作。

## VM 运行

现在，让我们把所有的拼图拼在一起，看看从调用 `run()` 到程序结束发生了什么：

- 准备阶段：VM 找到入口函数（比如 main），准备开始执行
- 函数调用：为入口函数创建一个新的帧，推入调用栈
- 指令循环：不断从当前帧取出下一条指令并执行，直到遇到 Ret 指令
- 值传递：指令间的数据通过栈来传递
- 返回处理：函数返回时，将返回值留在栈顶，恢复调用者的帧

在这个过程中，内置函数（如 `__builtin_print`）提供了一个特殊的通道，让虚拟机可以和外部世界交互：

```Cpp
registerIntrinsic("__builtin_print", [](const std::vector<RuntimeValue>& args) {
    for (const auto& arg : args) {
        std::cout << arg.toString();
    }
    return RuntimeValue();
});
```

这样，当我们在代码中调用 `__builtin_print("Hello")` 时，就会触发这个函数，在控制台输出 "Hello"。

把所有的部分组合起来，使用虚拟机只需要几行代码：

```Cpp
// 创建虚拟机
VM vm;

// 加载模块（从 IR 构建而来）
auto module = std::make_unique<Module>("test");

// 运行！
RuntimeValue result = vm.run(module.get(), "main");

// 看看结果
std::cout << "程序返回: " << result.toString() << std::endl;
```

## 编译流程

> 这一节代码在 `main.cpp` 中

太棒了，我们终于把最后一块拼图找到了。

那么接下来我们需要一张“图纸”把他拼起来，组成一个完整的链路才可以正确编译。

别忘了我们把整个链路包括在 `try...catch` 块中，因为可能抛异常哦。

### 源码

我们先来定义源码，默认的代码是这样的：

```Ryntra
public int main() {
    __builtin_print("hello world");
}
```

我们把它存在一个 `std::string` 中，用原始字符串字面量（Raw String Literal）存储，类似于：

```Cpp
std::string Source = R"(
public int main() {
    __builtin_print("hello world!2");
    return 0;
})";
```

### ANTLR 处理

ANTLR 的 Lexer 希望一个 `InputStream`，Parser 希望一个 `CommonTokenStream`，我们作为一个有爱的作者，必须满足他们：

```Cpp
antlr4::ANTLRInputStream input(Source);
Ryntra::antlr::RyntraLexer lexer(&input);
antlr4::CommonTokenStream tokens(&lexer);
tokens.fill();
Ryntra::antlr::RyntraParser parser(&tokens);
```

这样，因为我们的语法根叫做 `program()`，我们就可以通过：

```Cpp
auto tree = parser.program();
```

获取到解析树了。

### AST 构建

```Cpp
Ryntra::Compiler::ASTBuilder builder;
auto ast = builder.visitProgram(tree);
```

是不是相当简单？只是让 ASTBuilder 访问解析树的根节点而已。

### 语义分析

```Cpp
Ryntra::Compiler::Semantic::SemanticAnalyzer analyzer;
analyzer.analyze(ast);

Ryntra::Compiler::ErrorHandler::getInstance().print();
bool hasError = false;
for (const auto &error : Ryntra::Compiler::ErrorHandler::getInstance().getErrorObjects()) {
    if (error.type == Ryntra::Compiler::ET_ERROR) {
        hasError = true;
        break;
    }
}

if (hasError) {
    std::cout << "Semantic Analysis Failed." << std::endl;
} else {
    std::cout << "Semantic Analysis Passed." << std::endl;
}
```

我们调用语义分析器的 `analyze()` 函数，其中接受我们的 AST。然后我们需要检查报的错是全都是 Warning，还是全是 Error，还是混杂着的。如果全是 Warning 我们可以正常运行，如果但凡有一个 Error，就必须停止编译。

### IR 生成

```Cpp
Ryntra::Compiler::IR::HLIRBuilder irBuilder;
typedAST->accept(irBuilder);

auto module = irBuilder.takeModule();
```

我们用 HLIRBuilder，因为他会自动调用底层的 `IRBuilder`。

因为它实现了一个 Visitor 模式（经典版），所以调用 `accept()` 即可。

这里我们获取 Module，是为了 VM 解析。

### VM 运行

```Cpp
Ryntra::Compiler::VM::VM virtualMachine;
Ryntra::Compiler::VM::RuntimeValue result = virtualMachine.run(module.get(), "main");
```

直接调用 `run` 函数即可。

## 小结

我们的 Hello World 部分终于结束了！希望你能学点什么，然后下一节就是喜闻乐见的变量系统了，加油！