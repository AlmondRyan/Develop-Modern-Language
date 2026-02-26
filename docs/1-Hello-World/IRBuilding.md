# IR 构建

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/IR)

上一节我们已经设计了我们的 IR，那么你可能会想，我们理论上有了一个"形"，怎么让他成真呢？这节我们就来看看如何让他成真。

## Value 和 Type

> 文件：`Value.h/.cpp`, `Type.h/.cpp`

任何 IR 系统都需要回答两个最基本的问题：程序中的"东西"是什么？它是什么类型的？Value 和 Type 就是回答这两个问题的基石。

### Value

在我们的 SSA IR 中，**所有的东西都是值**，无论是指令呀、常量呀、函数都是值。

```Cpp
class Value {
public:
    virtual ~Value() = default;
    
    Type *getType() const { return type; }
    std::string getName() const { return name; }
    void setName(const std::string &n) { name = n; }
    
    virtual std::string toString() const;

protected:
    Value(Type *ty, const std::string &name = "") : type(ty), name(name) {}
    
    Type *type;
    std::string name;
};
```

在这里，Value 是一个基类，将构造函数设为 `protected` 意味着你不能直接实例化 Value 对象，必须继承他，这特别合理 —— 在 IR 中，所有"东西"都是某种特定的值。创建指令的时候需要继承、创建常量的时候需要继承……

### Type

```Cpp
enum class TypeID {
    Void,
    Integer32,
    String
};

class Type {
public:
    Type(TypeID id) : id(id) {}
    TypeID getID() const { return id; }
    
    static Type *getVoidTy();
    static Type *getInt32Ty();
    static Type *getStringTy();
    
    std::string toString() const;

private:
    TypeID id;
};
```

这里，`Type` 类设计非常简单，但是有几个非常重要的设计决策：

- 单例，单例，单例：`getVoidTy()`/`getInt32Ty()`/`getStringTy()` 应该返回唯一的类型实例，防止大量创建重复的类型对象
- 枚举：用枚举标识类型，高效又安全（我知道语义分析的时候用字符串有点鲁莽但是后期一定会改的，还是那句话，先跑起来再说）

类型系统是 IR 的骨架，后续的优化和代码生成都依赖于精确的类型信息。

## Instruction

有了值和类型，接下来要解决“程序到底在做什么”。指令系统就是用来描述程序行为的。

### OpCode

```Cpp
enum class OpCode {
    // 模块级元素
    Module = 0x01,
    External = 0x02,
    Constant = 0x03,
    
    // 内存操作
    Load = 0x04,
    
    // 控制流
    Call = 0x05,
    Ret = 0x06,
    Entry = 0x07,
    
    // 算术运算
    Add = 0x10,
    Sub = 0x11,
    Mul = 0x12,
    Div = 0x13,
    
    // 比较运算
    CmpEQ = 0x20,
    CmpNE = 0x21,
    CmpLT = 0x22,
    CmpGT = 0x23,
};
```

> **现在的这个分配并非最终版本，会随着后期指令的增加编号也会改变**

我们给每一个枚举值都分配了一个唯一的十六进制编码，其中 `0x01-0x0F` 给模块级控制语句和控制流，`0x10-0x1F` 给算术运算，`0x20-0x2F` 给比较运算。

### Instruction 类

```Cpp
class Instruction : public Value {
public:
    Instruction(Type *ty, OpCode op, const std::string &name = "", 
                BasicBlock *parent = nullptr)
        : Value(ty, name), opcode(op), parent(parent) {}

    OpCode getOpCode() const { return opcode; }
    BasicBlock *getParent() const { return parent; }
    
    const std::vector<Value *> &getOperands() const { return operands; }
    void addOperand(Value *v) { operands.push_back(v); }

private:
    OpCode opcode;
    BasicBlock *parent;            // 指令所属的基本块
    std::vector<Value *> operands; // 操作数列表
};
```

这个设计是这样的：

1. 继承自 `Value`：就像上文所说，指令也是一个值，意味着指令可以作为其他指令的操作数，正是 SSA 的核心 —— 指令的结果可以被后续指令使用
2. parent 指针：指向指令所属的基本块，这种关联方式（基本块包含指令，指令知道自己属于哪个基本块）在遍历 IR、修改 IR 的时候很有用
3. operands 向量：存储指令的操作数，这里存储 `Value*` 而非拷贝可以体现 IR 的引用语义

## Function

```Cpp
class Function : public Value {
public:
    Function(const std::string &name, Type *retType, Module *parent, 
             std::vector<Type *> argTypes = {});

    Module *getParent() const { return parent; }
    Type *getReturnType() const { return getType(); }
    
    const std::vector<std::unique_ptr<BasicBlock>> &getBasicBlocks() const { return basicBlocks; }
    const std::vector<Type *> &getArgTypes() const { return argTypes; }

    BasicBlock *addBasicBlock(std::unique_ptr<BasicBlock> bb);
    bool isDeclaration() const { return basicBlocks.empty(); }

    std::string toString() const override { return "@" + getName(); }

private:
    Module *parent;
    std::vector<std::unique_ptr<BasicBlock>> basicBlocks;
    std::vector<Type *> argTypes;
};
```

Function 的关键设计：

1. 区分声明和定义：`isDeclaration()` 通过检查 `basicBlocks` 是否为空来判断。如果一个函数没有基本块，那它就是一个外部声明（比如库函数），不需要生成代码体。
2. 参数类型信息：`argTypes` 保存了函数的参数类型。注意这里存的是 `Type*` 而不是完整的参数名——在 IR 层面，我们主要关心类型信息，参数名主要用于调试。
3. 返回类型：函数的返回类型实际上存储在继承的 `type` 字段中，`getReturnType()` 只是一个更语义化的包装。

## Module

Module 就是一个 Translation Unit。

```Cpp
class Module {
public:
    Module(const std::string &name);

    void addFunction(std::unique_ptr<Function> func);
    void addConstantObject(std::unique_ptr<ConstantObject> global);

    Function *getFunction(const std::string &name);
    ConstantObject *getConstantObject(const std::string &name);

    const std::vector<std::unique_ptr<Function>> &getFunctions() const { return functions; }
    const std::vector<std::unique_ptr<ConstantObject>> &getConstantObjects() const { return constantObjects; }

private:
    std::string name;
    std::vector<std::unique_ptr<Function>> functions;
    std::vector<std::unique_ptr<ConstantObject>> constantObjects;
    std::vector<std::unique_ptr<Constant>> constants;
};
```

Module 管理两类全局实体：

- 函数：包括用户定义的函数和外部声明的函数
- 全局常量：比如字符串字面量，它们在程序的整个生命周期都存在

注意这里也大量使用 `unique_ptr`，明确了所有权关系：Module 拥有其包含的所有函数和常量。

## IR Builder

有了以上的数据结构，理论上我们可以手动创建各种对象来构建 IR。但那样会非常繁琐——你需要手动创建指令、管理寄存器编号、处理类型转换等。构建器就是来解决这个问题的。

```Cpp
class IRBuilder {
public:
    IRBuilder(Module *m) : module(m), insertBlock(nullptr) {}

    void SetInsertPoint(BasicBlock *bb) { insertBlock = bb; }
    BasicBlock *GetInsertBlock() const { return insertBlock; }

    Function *CreateFunction(const std::string &name, Type *retType, 
                             std::vector<Type *> argTypes = {});
    BasicBlock *CreateBasicBlock(const std::string &name, Function *parent = nullptr);
    
    Instruction *CreateLoad(ConstantObject *global);
    Instruction *CreateCall(Function *func, const std::vector<Value *> &args);
    Instruction *CreateRet(Value *val);

private:
    std::string getNextRegisterName();
    
    Module *module;
    BasicBlock *insertBlock;
    size_t nextRegisterId = 0;
};
```

1. 插入点 (`insertBlock`)：记录当前指令要添加到哪个基本块。这类似于一个光标，你可以在不同基本块之间移动这个光标。
2. 自动寄存器命名：`getNextRegisterName()` 自动生成 `%1, %2, %3` 这样的 SSA 寄存器名。这解放了开发者，不需要手动跟踪编号。
3. 封装创建逻辑：每个 `CreateXXX` 方法都封装了指令的创建、类型设置、操作数添加、插入到基本块的全过程。

## HL-IR Builder

```Cpp
class HLIRBuilder : public Semantic::ITypedVisitor {
public:
    HLIRBuilder();
    std::unique_ptr<Module> takeModule() { return std::move(module); }

    // 遍历 AST 节点的 visit 方法
    void visit(Semantic::TypedProgramNode &node) override;
    void visit(Semantic::TypedFunctionDefinitionNode &node) override;
    void visit(Semantic::TypedBlockNode &node) override;
    void visit(Semantic::TypedExpressionStatementNode &node) override;
    void visit(Semantic::TypedReturnNode &node) override;
    void visit(Semantic::TypedStringLiteralNode &node) override;
    void visit(Semantic::TypedIntegerLiteralNode &node) override;
    void visit(Semantic::TypedIdentifierNode &node) override;
    void visit(Semantic::TypedFunctionCallNode &node) override;

private:
    Type *mapType(std::shared_ptr<Semantic::Type> type);
    
    std::unique_ptr<Module> module;
    std::unique_ptr<IRBuilder> builder;
    Value *lastValue = nullptr;  // 记录最后一次生成的 IR 值
};
```

1. 访问者模式：通过实现 `ITypedVisitor` 接口，HLIRBuilder 可以遍历 Typed AST 的每个节点。每访问一个节点，就生成对应的 IR。
2. 类型映射：`mapType` 负责将语义分析阶段的类型转换为 IR 层的类型。比如将语义层的 `IntType` 映射为 `Type::getInt32Ty()`。
3. 状态跟踪：`lastValue` 记录最近生成的 IR 值，这对于处理表达式嵌套很关键。比如 `1 + (2 * 3)`，需要把子表达式的结果传递给父表达式。

## 完整的构建流程

```Cpp
// 懒
using namespace Ryntra::Compiler;

// 创建模块，叫做 example
auto module = std::make_unique<Module>("example");

// 创建Builder，指向 module，也就是“我属于这个module！”
IRBuilder builder(module.get());

// 创建主函数，名称为 main，返回值为 int，无参数
Function *mainFunction = builder.CreateFunction("main", Type::getInt32Ty(), {});

// 创建基本块，名称为 entry，并设置当前插入点为 entry
BasicBlock *entryBlock = builder.CreateBasicBlock("entry", mainFunction);
builder.SetInsertPoint(entryBlock);

// 创建整数常量值，值为 69（没任何意义）
auto *const69 = new ConstantInt(69);

// 创建返回指令，返回这个整数常量
builder.CreateRet(const69);

// 打印
std::cout << module->print() << std::endl;
```

你将会得到：

```llvm
module example {
    func @main() -> i32 {
    entry:
        ret i32 69
    }
}
```

恭喜！太成功了！你成功创建了自己的第一个 IR！

## 小结

我们的 IR 构建系统现在已经初具规模，虽然目前只支持基础功能，但它的设计是开放且可扩展的。

下一节，终于来到了我们这个部分的最后了：VM，马上就能看到输出 Hello World 了！仅差临门一脚，别放弃。