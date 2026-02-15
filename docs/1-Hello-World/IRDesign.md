# IR 设计

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/IR)

欢迎来到特别重要的一节！（其实每一节都特别重要，都是强耦合的）

在这节中，我们将探讨一个事—— IR 的设计。先提前说一下，我们设计的是简单的类汇编、类三地址码的 IR，本节不讨论 IR 生成（`IRBuilder`）等内容，只把重点放在 IR 的设计。

这一节定义非常多，废话不多说，开始吧。

## Hello World

在讲述实现之前，我们先看看最终的 IR 是什么样子。好比是做菜之前先看效果图，防止中途放弃：

```asm
section .cdata
    string @str1 = "Hello World!"

section .code
_entry:
    define @main() -> i32 {
        loadc @str1
        syscall 0
        halt
    }
```

是不是特别像汇编和 Rust 的结合体？没错，这就是我们的设计目标：**既要接近高级语言的结构（函数和类型），也要接近底层的指令形式**。

整体分为两个 Section，分别是 `.cdata`（Constant Data）和 `.code`。`.cdata` 存储整个 Translation Unit 的常量（不包括在 16 位有符号整数范围内的整数，这些作为立即数直接插入指令部分），而 `.code` 就是你想的 —— 主要代码部分。

### 常量数据段

> 我们这里暂时不考虑多文件编译的 IR 生成

还是那句话，**这里不包括 16 位有符号整数范围内的整数**，因为这些会作为立即数插入到指令流中。这其实是有设计取舍的，因为如果把所有常量插入常量数据段，会导致数据段内容爆炸（尤其是一个 Translation Unit 有大量的常量的话），我们选择忽略占比比较大的整数字面量（如果是大整数或超出 16 位 int 的值，我们依旧把他们当作常量插入数据段）。

每个数据都由如下内容组成：
```
<type> @<name> = <value>
```

比如我们声明一个 32 位整数：
```asm
section .cdata
    i32 @l = 1145141919
```

### 代码段

代码段非常好理解，就是存储主要逻辑的段。代码段内存储的代码由标签（Tag，比如 `_entry:`（前面有下划线，后面有冒号））、函数定义、主要操作指令组成。

指令是类似于三地址码的指令，其实更像是汇编，比如：

```asm
syscall 0
```

代表系统调用编号为 0 的函数（这里是 `__builtin_print()`）。

## 核心设计

我们的 IR 设计深受 LLVM IR 的启发，整个系统的基石是两个类，分别是 `Value` 和 `Type`，称作 Value-Type 系统。

### Type

在 IR 中，一切皆有类型。不仅仅是变量，还有函数、基本块全部都有类型（基本块类型通常是 `Void`），也就是：

```Cpp
enum class TypeID {
    Void,
    Integer32,
    String
};
```

我们目前只有三种类型，即：

- `Void`：空类型，用于没有返回值的指令或函数
- `Integer32`：32 位整数类型，即 `i32`
- `String`：字符串

`Type` 类采用单例模式（也就是我们的 `ErrorHandler` 同样的设计）设计，通过 `Type::getInt32Ty()` 等静态方法获取类型。

这样不止节省内存，还方便比较。比较两个类型是否相同，仅需比较指针地址是否相等即可。

```Cpp
class Type {
public:
    Type(TypeID id) : id(id) {}
    TypeID getID() const {
        return id;
    }

    static Type *getVoidTy();
    static Type *getInt32Ty();
    static Type *getStringTy();

    std::string toString() const;
private:
    TypeID id;
}
```

### Value

`Value` 是 IR 中所有能被“使用”的东西的基类。它有两个关键属性：

- `Type* type`: 这个值的类型。
- `std::string name`: 这个值的名字（可选，主要为了调试好看）。

```Cpp
class Value {
public:
    virtual ~Value() = default;

    // type Getter...
    // name Getter & Setter...
protected:
    Value(Type *type, const std::string &name = "") : type(type), name(name) {}
    Type *type;
    std::string name;
}
```

此时引出了一个重要的继承体系：

- `Constant`（常量）：比如数字 `114514` 或字符串 `Hello`
- `Instruction`（指令）：指令本身也是一个值，这意味着一条指令的结果可以被另一条指令使用（在我们的三地址码设计中，体现不明显），比如 `loadc` 和 `syscall` 指令
- `BasicBlock`（基本块）：**代码** 的容器
- `Function`（函数）：**基本块** 的容器

## 基本结构

IR 不是一堆散乱的指令，他是高度结构化的。

### 模块

模块是最顶层的容器，你可以把它理解为一个源文件编译后的产物。包含：

- 全局常量 (`constantObjects`)：对应 `.cdata` 段。
- 函数列表 (`functions`)：对应 `.code` 段。

```Cpp
class Module {
public:
    // 构造函数
    Module(const std::string& name);
    
    // 给模块中添加函数
    void addFunction(std::unique_ptr<Function> func);
    // 给模块中添加常量（用于字符串）
    void addConstantObject(std::unique_ptr<ConstantObject> global);
    
    // 获取函数对象
    Function* getFunction(const std::string& name);
    // 获取常量对象
    ConstantObject* getConstantObject(const std::string& name);

    // 帮助函数：获取下一个 string 类型常量 ID
    size_t getNextStringConstantId() { return stringConstantCounter++; }
    // 添加常量（用于整数）
    void addConstant(std::unique_ptr<Constant> c) { constants.push_back(std::move(c)); }

    std::string print() const;

private:
    std::string name;
    std::vector<std::unique_ptr<Function>> functions;
    std::vector<std::unique_ptr<ConstantObject>> constantObjects;
    std::vector<std::unique_ptr<Constant>> constants;
    size_t stringConstantCounter = 0;
};
```

### 函数

函数包含了一系列的**基本块**（Basic Block）。

```Cpp
class Function : public Value {
public:
    Function(const std::string& name, Type* retType, Module* parent);
    
    // 获取模块
    Module* getParent() const { return parent; }
    // 获取基本块们
    const std::vector<std::unique_ptr<BasicBlock>>& getBasicBlocks() const { return basicBlocks; }
    
    // 添加基本块
    BasicBlock* addBasicBlock(std::unique_ptr<BasicBlock> bb);
    
    // 调试用
    std::string toString() const override;

private:
    Module* parent;
    std::vector<std::unique_ptr<BasicBlock>> basicBlocks;
};
```

函数的定义看起来像这样：`define @main() -> i32 { ... }`。它明确了名字、参数（目前还没做）和返回类型。

### 基本块

基本块是指令的容器。它的特点是：**单入口，单出口**。程序流只能从块头进入，从块尾离开（跳转或返回）。

```Cpp
class BasicBlock : public Value {
public:
    BasicBlock(const std::string& name, Function* parent);
    
    // 获取父函数
    Function* getParent() const { return parent; }
    // 获取其中包裹的指令
    const std::list<std::unique_ptr<Instruction>>& getInstructions() const { return instructions; }
    // 添加指令
    Instruction* addInstruction(std::unique_ptr<Instruction> inst);
    // 调试用
    std::string toString() const override;

private:
    Function* parent;
    std::list<std::unique_ptr<Instruction>> instructions;
};
```

在 Hello World 中，函数中的基本块是没有明显标签的（`_entry` 是程序入口标签，而非基本块标签），但实际上每一个基本块都是有一个标签的，在 `if...else...` 等控制流语句中，标签是跳转的目标。

## 指令

指令是真正干活的地方。我们的指令集目前非常精简（只有三个，简直是精简指令集中的精简版）：

```cpp
enum class OpCode {
    LoadC,   // 加载常量
    Syscall, // 系统调用
    Halt     // 停机
};
```

### `loadc` 加载常量

`loadc @str1`

这条指令用于把一个全局常量（如字符串）加载到栈中（在我们的抽象 IR 中，它更像是一个声明：“我要用这个常量了”）（我们的 VM 是标准的栈机）。

### `syscall` 调用内置函数

`syscall 0`

这是我们与虚拟机交互的方式之一，其中 `0` 代表打印字符串（即 `__builtin_print` 方法）

### `halt` 停止运行

停机指令。程序运行到这里就结束了。简单直接。

## 内存管理

你可能注意到了代码里大量的 `std::unique_ptr` 和 `std::move`。

```Cpp
void Module::addFunction(std::unique_ptr<Function> func) {
    functions.push_back(std::move(func));
}
```

这是为了明确**所有权**：

- `Module` 拥有 `Function`。
- `Function` 拥有 `BasicBlock`。
- `BasicBlock` 拥有 `Instruction`。

这种层级关系保证了当我们销毁一个 Module 时，它里面所有的东西都会被自动、干净地回收，不会发生内存泄漏。

## 小结

我们的 IR 设计虽然简单，但麻雀虽小，五脏俱全：

1. 分层结构：Module -> Function -> BasicBlock -> Instruction。
2. 类型安全：基于 `Type` 系统的强类型检查。
3. 统一抽象：一切皆 `Value`，无论是常量还是指令。

有了这个坚实的基础，下一节我们将探讨如何使用 `IRBuilder` 优雅地把 AST 变成这堆 IR 代码。