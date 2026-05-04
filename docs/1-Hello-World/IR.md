# IR

> 源代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/IR)

接下来看一个非常重中之重的部分 —— IR（Intermediate Representation，中间表示）。

## SSA

在了解我们的 IR 之前，先来看一个重要的概念：SSA（Static Single Assignment，静态单赋值）。

在传统的编程语言中，变量可以多次赋值，比如这样：

```javascript
let a = 10;
a += 20;
a = 114514;
```

在这种代码中，`x` 代表不同的值，取决于程序执行到哪一行，这对数据流分析产生了很大的麻烦。

于是我们引入了 SSA，在 SSA 中：

- 每个变量只能被定义（赋值）一次
- 如果需要多次赋值，我们会把变量重命名为多个不同的版本

如果上面的代码转到 SSA 中将会是这样：

```
a1 = 10;
a2 = a1 + 20;
a3 = 114514;
```

看到了吗？现在每个变量名都唯一对应一个具体的值，这使得定义-使用链（Use-Def Chain）变得特别直接，因为任何地方的变量使用都指向唯一的定义点。

## Ryntra SSA IR

知道了什么是 SSA 之后，我们来通过一个例子认识一下我们的 IR。

> **注意：我们的 IR 现在还是处于一个比较原型的阶段，暂时的设计目标是可以正确表达我们的 Hello World 程序，而没有采取直接设计好全部，否则会耽误很长时间**

（如果你跟着我的 GitHub 仓库写的代码，实际上并不会得到这么好看的输出，因为我为了教程好看手动调了嘿嘿）

```rir
module HelloWorld {
    external func @__builtin_print(string) -> void
    @unnamed0 = global constant string "Hello World\\n"
    func @main() -> i32 {
    entry:
        %1 = loadc @unnamed0
        call @__builtin_print(%1)
        ret i32 0
    }
}
```

首先你会看到一个东西叫做 Module（模块），如果你了解过 LLVM IR，应该对这个概念不陌生，和 LLVM IR 类似，都表示一个编译单元（一般情况下是一个源文件），是我们的 IR 中的最高级别的容器。

然后是 `external` 语法，这个用于声明该模块之外的元素，比如 `external func` 声明外部函数，`external constant` 声明外部常量。对于函数来说，我们只需要知道函数的签名即可。

`->` 语法标注返回值类型，对于所有的内置函数来说都需要标记为外部函数。

然后是重头戏 —— 常量。

注意，这里我们是在模块级别定义数据，而不是在函数内部。`@unnamed0` 是一个全局常量，存储了我们的字符串数据。

接下来是函数，函数通过 `func` 声明，函数名等标识符（不包括 SSA 值）通过 `@` 标记，依旧用 `->` 声明返回值类型，这里是 `i32` 也就是整数类型（相当于 C++ 的 `int` / `int32_t`）。

函数由基本块组成，每个基本块都由名称和指令构成，而函数必须有一个入口块叫做 `entry`，函数将从这里开始执行。

让我们看看函数体内部发生了什么：

1. `%1 = loadc @unnamed0`: 这里用到了 SSA 值（以 `%` 开头）。因为我们的 IR 基于 SSA，数据必须加载到"虚拟寄存器"中才能运算。`loadc` 指令的作用就是将全局常量 `@unnamed0` 加载到局部变量 `%1` 中。
2. `call`: 调用外部函数。
3. `ret`: 返回整型 0。

这段 IR 在底层的执行逻辑，大致对应如下的 VM 字节码：

```bytecode
load_const 0   ; 加载 "Hello World" 字符串
bcall 0        ; 调用 __builtin_print() 函数，从栈顶弹出元素
load_const 1   ; 加载常量 0（返回值）
ret            ; 返回，从栈顶弹出元素
```

## 实现

我们把 IR 的实现分为几个部分，包括：

1. 基础数据结构：
    - Type（类型）
    - Value（值）
    - Constant（常量）
    - Instruction（指令）
    - BasicBlock（基本块）
    - Function（函数）
    - Module（模块）
2. 构建工具：IRBuilder
3. 生成器：IRGenerator

让我们从基本数据结构下手，讲一下我们的 IR 实现。

### 类型 Type

类型是 IR 的基石，所有的 Value 和 Instruction 都需要类型。

```Cpp
class Type {
public:
    enum class Kind {
        Void,
        Int32,
        String,
        Function
    };

    Type(Kind kind) : kind_(kind) {}
    virtual ~Type() = default;

    Kind getKind() const { return kind_; }

    virtual std::string toString() const = 0;
    virtual bool isEqual(const Type *other) const = 0;

    bool isVoid() const { return kind_ == Kind::Void; }
    // 其余的 isXX() 函数

    static std::shared_ptr<Type> getVoidType();
    // 其余的 getXXType() 函数

private:
    Kind kind_;
};
```

我们把类型分为四种：`void`, `i32`, `string`, `func`。

然后是重点：`getXXType()` 静态函数，他们的作用很简单，提供全局共享的基本类型实例（工厂模式）。

为什么要这么写？很简单，给你举个例子你就懂了。假设我们在后面创建一个新的 IR 函数的时候，会写：

> 注意，后面的 `addFunction()` 函数签名不长这样，只是举个例子

```Cpp
builder.addFunction("foo", { Type::getInt32Type() }, { Type::getVoidType() });
```

如果我们凭直觉写出这些：

```Cpp
auto i32Type = std::make_shared<Type>(Type::Kind::Int32);
auto voidType = std::make_shared<Type>(Type::Kind::Void);
builder.addFunction("foo", i32Type, voidType);
```

如果有多个函数节点，循环的创建了几十个几百个这样的 `i32Type` 和 `voidType` 对象，内存占用是不可接受的。

于是我们采取全局单例模式，整个程序生命周期中，`i32Type` 和 `voidType` 对象只存在一份，显著的降低了内存占用。

```Cpp
class VoidType : public Type {
public:
    VoidType() : Type(Kind::Void) {}

    std::string toString() const override {
        return "void";
    }

    bool isEqual(const Type *other) const override {
        return other->isVoid();
    }
};

class Int32Type : public Type {
public:
    Int32Type() : Type(Kind::Int32) {}
    std::string toString() const override { /* Implementation */ }
    bool isEqual(const Type *other) const override { /* Implementation */ }
};

class StringType : public Type {
public:
    StringType() : Type(Kind::String) {}
    std::string toString() const override  { /* Implementation */ }
    bool isEqual(const Type *other) const override  { /* Implementation */ }
};

// 这个类的具体实现请去提供的 GitHub 仓库看，太长了不在这里写
class FunctionType : public Type {
public:
    FunctionType(std::shared_ptr<Type> returnType,
                    std::vector<std::shared_ptr<Type>> paramTypes)
        : Type(Kind::Function),
            returnType_(returnType),
            paramTypes_(paramTypes) {}

    std::shared_ptr<Type> getReturnType() const;
    const std::vector<std::shared_ptr<Type>> &getParamTypes() const;

    std::string toString() const override;
    bool isEqual(const Type *other) const override;
private:
    std::shared_ptr<Type> returnType_;
    std::vector<std::shared_ptr<Type>> paramTypes_;
};
```

接下来来看这些类型的实例。

1. 简单类型：没有内部数据，`int` 就是 `int`，`void` 就是 `void`，不需要额外信息
2. 复杂类型（比如现在的函数）：是独一无二的。`FunctionType` 的存在是为了组合其他类型。它把"返回值类型"和"参数类型列表"封装在了一起，形成了一个完整的函数签名。

这些不只是为了实现 `getXXType()` 这么简单，还实现了多态：

```Cpp
void printSignature(std::shared_ptr<Type> type) {
    std::cout << type->toString() << std::endl; 
}
```

这里如果传入 `Int32Type()`，会正确输出 `i32`；传入 `VoidType()` 正确输出 `void`，防止写一大堆 `if-else` 链导致我们累死。

```Cpp
inline std::shared_ptr<Type> Type::getVoidType() {
    static std::shared_ptr<Type> voidType = std::make_shared<VoidType>();
    return voidType;
}

inline std::shared_ptr<Type> Type::getInt32Type()  { /* ... */ }
inline std::shared_ptr<Type> Type::getStringType() { /* ... */ }
```

这些是具体的 `getXXType()` 实现。

### 值 Value

```Cpp
class Value {
public:
    Value(std::shared_ptr<Type> type, const std::string &name = "")
        : type_(type), name_(name) {}

    virtual ~Value() = default;

    std::shared_ptr<Type> getType() const { return type_; }
    const std::string &getName() const { return name_; }
    void setName(const std::string &name) { name_ = name; }

    virtual std::string toString() const = 0;

    virtual std::string getReferenceName() const {
        return name_.empty() ? "" : "@" + name_;
    }

    virtual bool isLocal() const { return false; }

protected:
    std::shared_ptr<Type> type_;
    std::string name_;
};
```

`Value` 是所有 IR 节点的基类，不管是常量、指令、还是函数，说到底都是"值"。这个基类提供了几个所有值都共通的属性：

- 类型：每个值都有类型，不管是 `i32`、`string` 还是函数类型
- 名称：每个值都有自己的名字，全局值（常量、函数）用 `@` 前缀，后面会看到局部 SSA 值用 `%` 前缀
- 序列化：`toString()` 是纯虚函数，子类根据自己的特性实现不同的输出

特别注意 `getReferenceName()` 函数，全局值返回 `@name`，而局部值（后面要说的 `Instruction`）会重写为 `%name`，这个区分直接对应了我们 IR 文本格式中的命名约定。

### 常量 Constant

聊完了 Value，接下来看它的第一个子类 —— Constant。

```Cpp
class Constant : public Value {
public:
    using ValueType = std::variant<int32_t, std::string>;

    Constant(std::shared_ptr<Type> type, ValueType value, const std::string &name = "")
        : Value(type, name), value_(value) {}

    const ValueType &getValue() const { return value_; }

    std::string toString() const override {
        std::string result = name_.empty() ? "" : "@" + name_ + " = ";
        result += "global constant " + type_->toString() + " ";

        if (std::holds_alternative<int32_t>(value_)) {
            result += std::to_string(std::get<int32_t>(value_));
        } else if (std::holds_alternative<std::string>(value_)) {
            std::string strValue = std::get<std::string>(value_);
            result += "\"" + escapeString(strValue) + "\"";
        }

        return result;
    }

    static std::shared_ptr<Constant> createInt32Constant(int32_t value, const std::string &name = "");
    static std::shared_ptr<Constant> createStringConstant(const std::string &value, const std::string &name = "");

private:
    static std::string escapeString(const std::string &str) { /* ... */ }

    ValueType value_;
};
```

Constant 代表模块级别的全局常量，使用 `@` 前缀引用。

这里有个很有意思的设计 —— 我们用 `std::variant<int32_t, std::string>` 作为常量的值类型。`variant` 是 C++17 引入的"类型安全的联合体"，它能存储 `int32_t` 和 `string` 两种类型之一，但比传统的 `union` 安全得多，因为你不可能误读里面的数据。

`toString()` 的输出格式形如 `@name = global constant i32 42` 或 `@name = global constant string "Hello"`，对应了我们 IR 文本格式中模块级别的常量声明。

另外注意 `escapeString()` 函数，它负责处理字符串中的转义字符：`\n`、`\t`、`\"` 等等，确保输出的字符串在 IR 文本格式中是合法且可读的。

工厂方法 `createInt32Constant()` 和 `createStringConstant()` 简化了常量的创建过程，调用者不需要手动指定类型，传入值就行了。

### 指令 Instruction

指令是 IR 的核心，它继承自 Value，但和 Constant 不同 —— Instruction 是局部值，使用 `%` 前缀。

```Cpp
class Instruction : public Value {
public:
    enum class Opcode {
        LoadConstant,
        Call,
        Return,
        Add,
        Sub,
        Mul,
        Div
    };

    Instruction(Opcode opcode, std::shared_ptr<Type> type,
                const std::vector<std::shared_ptr<Value>> &operands = {},
                const std::string &name = "")
        : Value(type, name), opcode_(opcode), operands_(operands) {}

    Opcode getOpcode() const { return opcode_; }
    const std::vector<std::shared_ptr<Value>> &getOperands() const { return operands_; }

    // SSA instructions are local values — reference with %
    std::string getReferenceName() const override {
        return name_.empty() ? "" : "%" + name_;
    }

    bool isLocal() const override { return true; }
    std::string toString() const override { /* ... */ }
};
```

指令有哪些？目前我们定义了这些操作码：

| Opcode | 含义 |
|--------|------|
| `LoadConstant` | 加载全局常量到局部 SSA 值 |
| `Call` | 函数调用 |
| `Return` | 函数返回 |
| `Add` / `Sub` / `Mul` / `Div` | 算术运算 |

指令的名字（SSA 值名）通过 `getReferenceName()` 返回 `%name`，和全局的 `@name` 区分开来，这直接体现了 SSA 中局部值和全局值的区别。

来看一下 `toString()` 的实现细节，这里有几个值得一提的点：

1. 返回值处理：`ret` 指令永远不会产生 SSA 值；`call` 只有当返回类型非 `void` 时才会产生 SSA 值。这一点非常重要 —— 我们不会给无返回值的调用分配虚拟寄存器。
2. 操作数输出：每条指令的操作数通过 `getReferenceName()` 引用，这意味着一份 `Value*` 可以被多条指令引用，但输出时会自动带上 `@` 或 `%` 前缀。
3. 算术指令：`add`、`sub`、`mul`、`div` 都采用相同的操作数模式 —— 两个操作数，一个结果。

输出示例：

```
%1 = loadc @unnamed0
call @__builtin_print(%1)
ret i32 0
%2 = add %3, %4
```

### 立即值 ImmediateValue

```Cpp
class ImmediateValue : public Value {
public:
    ImmediateValue(std::shared_ptr<Type> type, const std::string &literalValue)
        : Value(type, ""), literalValue_(literalValue) {}

    std::string toString() const override {
        return type_->toString() + " " + literalValue_;
    }

    std::string getReferenceName() const override {
        return toString();
    }
};
```

这个东西很简单但很有用：它代表了一个"立即数"，即直接在指令中编码的字面值，比如 `i32 0`、`i32 42` 这种。它没有名字（`name_` 为空），`getReferenceName()` 直接返回 `toString()` 的值，所以在指令输出中你会看到 `ret i32 0` 而不是 `ret %some_value`。

这玩意在哪用？在 `IRBuilder::createReturnInt32()` 里，我们创建了一个 `ImmediateValue` 来代表返回值 `0`：

```Cpp
std::shared_ptr<Instruction> IRBuilder::createReturnInt32(const std::string &name, int32_t value) {
    auto immediate = std::make_shared<ImmediateValue>(
        Type::getInt32Type(),
        std::to_string(value));
    return createReturn(name, immediate);
}
```

这样 `ret i32 0` 就出来了。

### 基本块 BasicBlock

有了指令，我们就可以把它们组织成基本块了。

```Cpp
class BasicBlock {
public:
    BasicBlock(const std::string &name) : name_(name) {}

    const std::string &getName() const { return name_; }
    void addInstruction(std::shared_ptr<Instruction> instruction) {
        instructions_.push_back(instruction);
    }
    const std::vector<std::shared_ptr<Instruction>> &getInstructions() const {
        return instructions_;
    }

    std::string toString() const {
        std::string result = name_ + ":\n";
        for (const auto &inst : instructions_) {
            result += "    " + inst->toString() + "\n";
        }
        return result;
    }

private:
    std::string name_;
    std::vector<std::shared_ptr<Instruction>> instructions_;
};
```

基本块的概念在编译原理中非常重要，它是一个"直的代码片段" —— 只有一个入口和一个出口，块内的指令会顺序执行，不会跳进跳出。

在我们现在的实现中，BasicBlock 非常简单：

- 名字：比如 `entry`，用于在 IR 文本中标记块的开始
- 指令列表：用 `vector` 存放当前块中的所有指令
- 序列化：输出格式是 `name_:\n` 后面跟着一堆缩进的指令

这里注意 `BasicBlock` 并没有继承 `Value`，因为它不是 SSA 值 —— 你不能把整个基本块当成操作数传给指令。基本块只是指令的容器。

### 函数 Function

函数是比基本块更高一级的组织单位。

```Cpp
class Function : public Value {
public:
    struct Parameter {
        std::string name;
        std::shared_ptr<Type> type;
    };

    Function(const std::string &name,
             std::shared_ptr<Type> returnType,
             const std::vector<Parameter> &parameters = {},
             bool isExternal = false)
        : Value(std::make_shared<FunctionType>(returnType, extractParamTypes(parameters)), name),
          returnType_(returnType),
          parameters_(parameters),
          isExternal_(isExternal) {}

    std::shared_ptr<Type> getReturnType() const;
    const std::vector<Parameter> &getParameters() const;
    bool isExternal() const;

    void addBasicBlock(std::shared_ptr<BasicBlock> block);
    const std::vector<std::shared_ptr<BasicBlock>> &getBasicBlocks() const;
    std::shared_ptr<BasicBlock> getEntryBlock() const;

    std::string toString() const override { /* ... */ }
};
```

Function 有几个值得注意的地方：

1. 继承自 Value：函数本身也是一个值，它的类型是 `FunctionType`，封装了返回类型和参数类型列表。这意味着你可以把函数当作参数传递给 `call` 指令。
2. Parameter 结构体：用一个内部结构体描述函数参数，每个参数有名字和类型。不过看了代码你会发现，在当前的实现中参数名并没有在 IR 文本中输出，只输出了类型，这是为了简化。
3. `isExternal`：标记是否是外部函数。外部函数只有签名没有函数体（基本块），输出时前面会带上 `external` 关键字。
4. 基本块列表：函数体由一组基本块构成，第一个基本块被视为入口块（`getEntryBlock()`），整个函数从这里开始执行。

来看序列化的逻辑：

```Cpp
std::string toString() const override {
    std::string result;
    if (isExternal_) {
        result = "external ";
    }
    result += "func @" + name_ + "(";
    for (size_t i = 0; i < parameters_.size(); ++i) {
        if (i > 0) result += ", ";
        result += parameters_[i].type->toString();
    }
    result += ") -> " + returnType_->toString();
    if (!isExternal_ && !basicBlocks_.empty()) {
        result += " {\n";
        for (const auto &block : basicBlocks_) {
            result += block->toString();
        }
        result += "}";
    }
    return result;
}
```

外部函数输出 `external func @name(...) -> retType`，内部函数输出完整的带函数体的版本。这样我们就得到了类似这样的输出：

```
external func @__builtin_print(string) -> void

func @main() -> i32 {
entry:
    %1 = loadc @unnamed0
    call @__builtin_print(%1)
    ret i32 0
}
```

### 模块 Module

最后是最高级的容器 —— Module。

```Cpp
class Module {
public:
    Module(const std::string &name) : name_(name) {}

    void addFunction(std::shared_ptr<Function> function);
    void addConstant(std::shared_ptr<Constant> constant);

    const std::vector<std::shared_ptr<Function>> &getFunctions() const;
    const std::vector<std::shared_ptr<Constant>> &getConstants() const;

    std::shared_ptr<Function> getFunction(const std::string &name) const;
    std::shared_ptr<Constant> getConstant(const std::string &name) const;

    std::string toString() const { /* ... */ }
};
```

Module 是整个 IR 的根节点，它包含了函数和常量的集合，并且通过名字映射表提供快速的查找。

这里来看最核心的 `toString()` 方法，它定义了整个 IR 文本格式的输出结构和顺序：

```Cpp
std::string toString() const {
    std::string result = "module " + name_ + " {\n\n";

    // 1. External function declarations first
    for (const auto &function : functions_) {
        if (function->isExternal()) {
            result += "    " + function->toString() + "\n";
        }
    }

    // 2. Global constants
    for (const auto &constant : constants_) {
        result += "    " + constant->toString() + "\n";
    }

    // 3. Defined functions
    for (const auto &function : functions_) {
        if (!function->isExternal())
            result += "    " + function->toString() + "\n\n";
    }

    result += "}\n";
    return result;
}
```

输出顺序是：

1. 先输出所有的 `external` 声明（外部函数）
2. 再输出所有的全局常量声明
3. 最后输出函数定义（带函数体的内部函数）

这种"先声明后定义"的顺序，让 IR 文本格式的可读性更好 —— 你一眼就能看到这个模块依赖了哪些外部元素，有哪些全局数据，然后再看函数的具体实现。

### IR 构建 IRBuilder

有了上面的数据结构，现在需要一种方便的方式来构建 IR。我们不能让调用者手动创建 `Function`、`BasicBlock`、`Instruction` 然后拼到一起，那太容易出错了。

于是我们有了 `IRBuilder`。

```Cpp
class IRBuilder {
public:
    IRBuilder();

    std::shared_ptr<Module> createModule(const std::string &name);
    std::shared_ptr<Function> createFunction(const std::string &name, std::shared_ptr<Type> returnType, const std::vector<Function::Parameter> &parameters = {}, bool isExternal = false);
    std::shared_ptr<BasicBlock> createBasicBlock(const std::string &name);
    std::shared_ptr<Constant> createGlobalConstant(std::shared_ptr<Type> type, Constant::ValueType value);
    std::shared_ptr<Instruction> createLoadConstant(const std::string &name, std::shared_ptr<Constant> constant);
    std::shared_ptr<Instruction> createCall(const std::string &name,std::shared_ptr<Function> function, const std::vector<std::shared_ptr<Value>> &args);
    std::shared_ptr<Instruction> createReturn(const std::string &name,std::shared_ptr<Value> value = nullptr);
    std::shared_ptr<Instruction> createReturnInt32(const std::string &name,int32_t value);
    std::shared_ptr<Instruction> createBinaryOp(Instruction::Opcode opcode,const std::string &name, std::shared_ptr<Value> lhs, std::shared_ptr<Value> rhs);

    void setInsertPoint(std::shared_ptr<BasicBlock> block);
    std::shared_ptr<BasicBlock> getInsertPoint() const;

    void addInstruction(std::shared_ptr<Instruction> instruction);
    std::shared_ptr<Module> getModule() const;
    std::string generateUniqueName(const std::string &base = "temp");

private:
    std::shared_ptr<Module> currentModule_;
    std::shared_ptr<BasicBlock> currentBlock_;
    int unnamedCounter_;
};
```

IRBuilder 维护了两个关键状态：

- `currentModule_`：当前正在构建的模块
- `currentBlock_`：当前插入点（正在填充指令的基本块）
- `unnamedCounter_`：为匿名实体生成唯一名字的计数器

来看几个核心方法的实现。

A. 创建函数：

```Cpp
std::shared_ptr<Function> IRBuilder::createFunction(const std::string &name, std::shared_ptr<Type> returnType, const std::vector<Function::Parameter> &parameters, bool isExternal) {
    auto function = std::make_shared<Function>(name, returnType, parameters, isExternal);
    currentModule_->addFunction(function);
    return function;
}
```

创建函数并自动注册到当前模块。

B. 创建全局常量（匿名版本）：

```Cpp
std::shared_ptr<Constant> IRBuilder::createGlobalConstant(std::shared_ptr<Type> type, Constant::ValueType value) {
    std::string name = "unnamed" + std::to_string(unnamedCounter_++);
    return createGlobalConstant(name, type, value);
}
```

如果调用者没有提供名字，自动生成 `unnamed0`、`unnamed1` 这样的名称。这就解释了我们的 IR 输出中为什么会有 `@unnamed0`。

C. 创建 `loadc` 指令：

```Cpp
std::shared_ptr<Instruction> IRBuilder::createLoadConstant(const std::string &name, std::shared_ptr<Constant> constant) {
    std::vector<std::shared_ptr<Value>> operands;
    operands.push_back(constant);
    auto instruction = std::make_shared<Instruction>(
        Instruction::Opcode::LoadConstant,
        constant->getType(),
        operands, name);
    if (currentBlock_) {
        currentBlock_->addInstruction(instruction);
    }
    return instruction;
}
```

关键点：创建指令后，如果当前有插入点（`currentBlock_`），指令会自动追加到当前基本块的指令列表中。这就是"构建器"模式的精髓 —— 你不需要手动管理指令和基本块的关系，只要设置好插入点，指令就会自动归位。

D. 创建 `call` 指令：

```Cpp
std::shared_ptr<Instruction> IRBuilder::createCall(const std::string &name, std::shared_ptr<Function> function, const std::vector<std::shared_ptr<Value>> &args) {
    if (!function) return nullptr;

    auto funcType = std::dynamic_pointer_cast<FunctionType>(function->getType());
    if (!funcType) return nullptr;

    std::vector<std::shared_ptr<Value>> operands;
    operands.push_back(function);
    operands.insert(operands.end(), args.begin(), args.end());

    auto instruction = std::make_shared<Instruction>(
        Instruction::Opcode::Call,
        funcType->getReturnType(),
        operands,
        name);

    if (currentBlock_) {
        currentBlock_->addInstruction(instruction);
    }

    return instruction;
}
```

注意 `operands` 的第一个元素是被调用的函数本身，后面的元素是参数。这样在 `Instruction::toString()` 中就可以统一用 `operands_[0]->getReferenceName()` 来输出函数名。

E. 创建算术指令：

```Cpp
std::shared_ptr<Instruction> IRBuilder::createBinaryOp(Instruction::Opcode opcode, const std::string &name, std::shared_ptr<Value> lhs, std::shared_ptr<Value> rhs) {
    if (!lhs->getType()->isEqual(rhs->getType().get())) {
        return nullptr;  // 类型不匹配，拒绝生成
    }
    std::vector<std::shared_ptr<Value>> operands = {lhs, rhs};
    auto instruction = std::make_shared<Instruction>(
        opcode, lhs->getType(), operands, name);
    // ...
}
```

这里有一个类型检查：左右操作数的类型必须一致，否则返回 `nullptr`。这算是一种在 IR 层面的防御性编程，防止生成不合法的指令。

F. 生成唯一名字：

```Cpp
std::string IRBuilder::generateUniqueName(const std::string &base) {
    return base + std::to_string(unnamedCounter_++);
}
```

这个函数被 IRGenerator 广泛使用，它为 SSA 值生成唯一的名称。每次调用都会递增计数器，确保生成的每个名字都是唯一的 —— 这正好符合 SSA 的要求，每个变量只能被定义一次。

### IR 生成 IRGenerator

好了，有了数据结构，有了构建工具，最后一步就是如何把 AST（抽象语法树）转换成 IR。

```Cpp
class IRGenerator : public Compiler::Semantic::ITypedVisitor {
public:
    IRGenerator();

    std::shared_ptr<Module> generate(Compiler::Semantic::TypedProgramNode &program, const std::string &moduleName = "module");

    void visit(Compiler::Semantic::TypedProgramNode &node) override;
    void visit(Compiler::Semantic::TypedFunctionDefinitionNode &node) override;
    void visit(Compiler::Semantic::TypedBlockNode &node) override;
    void visit(Compiler::Semantic::TypedExpressionStatementNode &node) override;
    void visit(Compiler::Semantic::TypedReturnNode &node) override;
    void visit(Compiler::Semantic::TypedStringLiteralNode &node) override;
    void visit(Compiler::Semantic::TypedIntegerLiteralNode &node) override;
    void visit(Compiler::Semantic::TypedIdentifierNode &node) override;
    void visit(Compiler::Semantic::TypedFunctionCallNode &node) override;

private:
    IRBuilder builder_;
    std::shared_ptr<Value> lastValue_;
    std::unordered_map<std::string, std::shared_ptr<Function>> functionMap_;
};
```

IRGenerator 继承自 `ITypedVisitor`（访问者模式），它的工作流程是：遍历 AST 的每个节点，每访问一个节点就生成对应的 IR 结构。

关键概念有两个：

1. `lastValue_`：上一个表达式节点生成的 IR 值。比如你访问了一个字符串字面量，`lastValue_` 就是加载这个字符串的 `loadc` 指令；访问了一个整数字面量，`lastValue_` 就是一个 `ImmediateValue`。后续的指令（如 `return`）会从 `lastValue_` 中取操作数。
2. `functionMap_`：函数名到 IR 函数的映射表，用于处理函数调用时的查找。注意这里的处理顺序很有意思 —— 先遍历一遍所有函数来"声明"它们，再遍历一遍来生成函数体，这叫做"两遍扫描"。

让我挑几个关键的方法说一下。

A. Program 节点的 visit：

```Cpp
void IRGenerator::visit(Sem::TypedProgramNode &node) {
    // 第一遍：先声明所有函数
    for (const auto &func : node.getFunctions()) {
        auto retType = toIRType(func->getReturnType());
        auto irFunc = builder_.createFunction(func->getName(), retType, {});
        functionMap_[func->getName()] = irFunc;
    }
    // 第二遍：再生成函数体
    for (const auto &func : node.getFunctions()) {
        func->accept(*this);
    }
}
```

第一遍遍历 AST 中的函数声明，为每个函数创建 IR 函数对象并注册到 `functionMap_`；第二遍遍历才生成函数体。为什么要这样？因为函数内部可能会调用后面定义的函数，如果只遍历一遍，遇到调用时被调用的函数还没创建，就会出现问题。

B. FunctionDefinition 节点的 visit：

```Cpp
void IRGenerator::visit(Sem::TypedFunctionDefinitionNode &node) {
    auto irFunc = functionMap_[node.getName()];
    auto entry = builder_.createBasicBlock("entry");
    irFunc->addBasicBlock(entry);
    builder_.setInsertPoint(entry);
    node.getBody()->accept(*this);
}
```

每个函数都创建一个名为 `entry` 的入口基本块，设置插入点，然后递归访问函数体。这就是为什么所有函数的第一个基本块都叫 `entry`。

C. Return 节点的 visit：

```Cpp
void IRGenerator::visit(Sem::TypedReturnNode &node) {
    node.getValue()->accept(*this);
    auto retVal = lastValue_;
    builder_.createReturn("", retVal);
    lastValue_ = nullptr;
}
```

先访问返回值表达式（结果存入 `lastValue_`），然后用 `lastValue_` 创建 `ret` 指令。注意 `createReturn("", retVal)` 传入了一个空字符串作为名字，因为 `ret` 指令不产生 SSA 值，不需要名字。最后把 `lastValue_` 置空，避免污染后续的表达式。

D. StringLiteral 节点的 visit：

```Cpp
void IRGenerator::visit(Sem::TypedStringLiteralNode &node) {
    auto constant = builder_.createGlobalConstant(
        Type::getStringType(),
        node.getValue());
    lastValue_ = builder_.createLoadConstant(
        builder_.generateUniqueName(""), constant);
}
```

访问字符串字面量时，首先创建一个全局常量（这就是 `@unnamed0` 的由来），然后生成一条 `loadc` 指令把它加载到局部 SSA 值中。`builder_.generateUniqueName("")` 生成像 `0`、`1`、`2` 这样的唯一名字。

E. FunctionCall 节点的 visit：

```Cpp
void IRGenerator::visit(Sem::TypedFunctionCallNode &node) {
    const std::string &calleeName = node.getFunctionName()->getName();

    // 先求值所有参数
    std::vector<std::shared_ptr<Value>> argValues;
    for (const auto &arg : node.getArguments()) {
        arg->accept(*this);
        if (lastValue_) argValues.push_back(lastValue_);
    }

    auto it = functionMap_.find(calleeName);
    if (it != functionMap_.end()) {
        callee = it->second;
    } else {
        // 没有在模块中找到 -> 声明为外部函数，参数类型从参数值推断
        auto retType = toIRType(node.getType());
        std::vector<Function::Parameter> params;
        for (size_t i = 0; i < argValues.size(); ++i) {
            params.emplace_back("p" + std::to_string(i), argValues[i]->getType());
        }
        callee = builder_.createFunction(calleeName, retType, params, true);
        functionMap_[calleeName] = callee;
    }

    // 如果返回类型是 void，不分配 SSA 名字
    bool isVoidCall = callee->getReturnType()->isVoid();
    std::string callName = isVoidCall ? "" : builder_.generateUniqueName("");
    auto callInst = builder_.createCall(callName, callee, argValues);
    lastValue_ = isVoidCall ? nullptr : callInst;
}
```

这是最复杂的一个 visit 方法。流程如下：

1. 先求值参数：因为参数表达式可能有副作用，必须按顺序求值
2. 查找被调用的函数：如果 `functionMap_` 里找到了，直接用；找不到就自动声明一个外部函数，参数类型从实际的参数值推断。这让我们在调用 `__builtin_print` 这样的内置函数时不需要预先声明
3. 决定是否分配 SSA 名字：如果函数返回 `void`，就不需要 SSA 值，传空字符串给 `createCall()`

还有一个小小的辅助方法 `toIRType()`，用于把语义分析阶段的类型映射到 IR 类型：

```Cpp
std::shared_ptr<Type> IRGenerator::toIRType(const std::shared_ptr<Sem::Type> &semType) {
    switch (semType->getKind()) {
    case Sem::TypeKind::VOID:    return Type::getVoidType();
    case Sem::TypeKind::PRIMITIVE: {
        const auto &name = static_cast<const Sem::PrimitiveType &>(*semType).getName();
        if (name == "i32" || name == "int")     return Type::getInt32Type();
        if (name == "string" || name == "str")  return Type::getStringType();
        return Type::getInt32Type();
    }
    case Sem::TypeKind::FUNCTION: { /* 递归转换参数和返回类型 */ }
    default: return Type::getVoidType();
    }
}
```

## 小结

这里，我们的 IR 就基本讲完了。下一章，让我们来看一下最重要的后端部分（也是第一章的最后一部分） —— Stack-Based VM（栈式虚拟机）。

**再次声明：我们现在的 IR 只是为了表示我们的 Hello World 程序，并没有完全做好，后面会开一章详细讲 IR 的优化。**

感谢你看到这里。