# VM

> 源代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/VM)

恭喜你！来到了第一章的最后一部分 —— VM。有了 VM，我们的语言才能真正的跑起来。

让我们抱着激动的心情，开始吧！

## 栈机 VM

先来简单认识一下栈机 VM：

> 栈式虚拟机（Stack-based Virtual Machine, Stack VM）是一种以"栈"这种数据结构为核心来执行计算任务的抽象计算模型。它的设计哲学是用软件模拟的栈来管理所有数据，从而简化指令集和虚拟机的实现。

栈式虚拟机的核心是操作数栈（Operand Stack），遵循 LIFO 的原则：

1. 数据入栈（Push）：当需要处理数据时（例如一个数字或一个变量），会先执行指令将其"压入"操作数栈的顶部。
2. 运算执行：当需要执行运算时（例如加法），运算指令（如 `ADD`）会隐式地从栈顶弹出所需数量的操作数（例如两个），完成计算后，再将结果"压回"栈顶。
3. 数据出栈（Pop）：计算结果可以保留在栈顶供后续指令使用，或者被弹出并存入其他位置（如局部变量表）。

我知道你们看这些枯燥乏味的定义会感到没意思，我们举个例子，比如计算 $114 + 514$：

1. 首先 `PUSH 114` 入栈，现在栈里有 `[114]`
2. 然后 `PUSH 514` 入栈，现在栈里有 `[114, 514]`
3. 最后调用 `ADD`，栈弹出两个操作数 `114, 514` 加一起，然后推回去，现在栈里有 `[628]`

操作流程就是这么个流程，我们实战一下。

## 实现

### 架构核心总览

先来看一下整体的架构。我们目前有四个核心部分：

1. VMValue：运行时值的表示，所有在虚拟机里跑来跑去的数据都是这玩意儿
2. Bytecode：字节码的指令定义，以及每个"函数"的字节码数据结构
3. BytecodeGenerator：把 IR Module 翻译成字节码
4. VirtualMachine：真正执行字节码的引擎

链路非常简单：IR Module $\rightarrow$ BytecodeGenerator $\rightarrow$ Bytecode $\rightarrow$ VirtualMachine $\rightarrow$ 跑起来

你要做的就是让前面的 IR Generator 输出一个 IR Module，然后塞给 `BytecodeGenerator`，它吐出字节码，再喂给 `VirtualMachine`，然后啪的一下 —— 跑起来了，就这么神奇。

### VMValue & Bytecode

先看 VMValue，它是虚拟机中所有运行时值的统一表示：

```Cpp
class VMValue {
public:
    enum class Type { Void, Int32, String, FunctionPtr };
    using ValueData = std::variant<std::monostate, int32_t, std::string, void*>;

    VMValue() : type_(Type::Void), data_(std::monostate{}) {}
    explicit VMValue(int32_t val) : type_(Type::Int32), data_(val) {}
    explicit VMValue(const std::string& val) : type_(Type::String), data_(val) {}

    Type getType() const;
    int32_t asInt32() const;
    const std::string& asString() const;
    bool isVoid() const;
    bool isInt32() const;
    bool isString() const;
};
```

和 IR 的 `Constant` 类似，这里又用了 `std::variant`，老演员了。`VMValue` 可以表示四种运行时值：

- `Void`：空值，啥也没有，比如没有返回值的函数的结果
- `Int32`：32 位整数
- `String`：字符串
- `FunctionPtr`：函数指针，不过现在还没怎么用到，留作扩展

然后是字节码的指令定义：

> 这里算术四则运算实现了但是没有用到，下一章会用到

```Cpp
enum class OpCode : uint8_t {
    LoadConst,  // 加载常量到栈顶
    Call,       // 调用用户自定义函数
    BCall,      // 调用内置函数
    Return,     // 从函数返回
    Add, Sub, Mul, Div,  // 算术四则运算
    Pop,        // 弹出栈顶值
    Halt        // 停止执行
};

struct Instruction {
    OpCode opcode;
    int32_t operand;  // 指令参数（常量池索引、函数索引等）
};

class BytecodeFunction {
public:
    std::string name;
    std::vector<Instruction> instructions;
    bool isExternal;
    int32_t paramCount;
};
```

每个 `Instruction` 是一个 opcode + operand 的组合。操作数 `operand` 是个多面手：

- 对于 `LoadConst`，operand 是常量池中的索引
- 对于 `Call`，operand 是函数列表中的索引
- 对于 `BCall`，operand 是内置函数表中的索引
- 算术指令完全不需要 operand，操作数直接从栈上取

`BytecodeFunction` 是一个编译后的函数表示。注意它和 IR 中的 `Function` 的对应关系 —— 每个 IR 函数最终会被翻译成一个 `BytecodeFunction`。`isExternal` 标记外部函数（不需要生成指令，但需要保留索引以便 `Call` 指令跳转），`paramCount` 记录参数个数，方便调用时从栈上正确取参数。

### Bytecode Generator

有了字节码的数据结构，接下来就是怎么把 IR 转换过来了。

```Cpp
class BytecodeGenerator {
public:
    BytecodeGenerator();
    std::vector<std::shared_ptr<BytecodeFunction>> generate(
        const std::shared_ptr<IR::Module>& module);
    const std::vector<VMValue>& getConstantPool() const;
};
```

输入是一个 IR Module，输出是一组 `BytecodeFunction` 和一个常量池。和 IRGenerator 类似，这里也用了两遍扫描：

1. 第一遍：遍历所有的 IR 函数，为每个函数创建一个 `BytecodeFunction` 并记录索引。外部函数也只留一个占位，方便被调用时索引正确
2. 第二遍：遍历所有非外部函数的 IR 基本块，逐条生成字节码指令

```Cpp
std::vector<std::shared_ptr<BytecodeFunction>> BytecodeGenerator::generate(
    const std::shared_ptr<IR::Module>& module)
{
    // 第一遍：注册所有函数，建立名字到索引的映射
    int32_t idx = 0;
    for (const auto& func : module->getFunctions()) {
        functionIndices_[func->getName()] = idx++;
        functions_.push_back(std::make_shared<BytecodeFunction>(
            func->getName(), func->isExternal(), func->getParameters().size()));
    }
    // 第二遍：为非外部函数生成字节码
    for (size_t i = 0; i < module->getFunctions().size(); ++i) {
        if (!module->getFunctions()[i]->isExternal()) {
            currentFunction_ = functions_[i];
            for (const auto& block : irFunc->getBasicBlocks()) {
                generateBasicBlock(block);
            }
        }
    }
    return functions_;
}
```

这里有个细节 —— `paramCount` 是从 IR 函数的参数列表长度推断出来的，这样在执行 `Call` 指令时，VM 就知道要从栈上弹多少个参数了。

接下来是逐条指令的翻译，以 `LoadConstant` 为例：

```Cpp
case IR::Instruction::Opcode::LoadConstant: {
    auto constant = std::dynamic_pointer_cast<IR::Constant>(operands[0]);
    if (constant) {
        VMValue val;
        if (constant->getType()->isInt32())
            val = VMValue(std::get<int32_t>(constant->getValue()));
        else if (constant->getType()->isString())
            val = VMValue(std::get<std::string>(constant->getValue()));
        int32_t poolIdx = addConstant(val);
        currentFunction_->addInstruction(OpCode::LoadConst, poolIdx);
    }
    break;
}
```

看到没？IR 中的 `loadc` 指令在这里被翻译成了 `LoadConst` 字节码。IR 常量被转换成 `VMValue` 塞进常量池，然后把常量池索引作为指令操作数。

再看看 `Call` 和 `BCall` 的区分：

```Cpp
case IR::Instruction::Opcode::Call: {
    if (callee->getName().rfind("__builtin_", 0) == 0) {
        int32_t builtinIdx = getBuiltinIndex(name);
        currentFunction_->addInstruction(OpCode::BCall, builtinIdx);
    } else {
        int32_t funcIdx = getFunctionIndex(name);
        currentFunction_->addInstruction(OpCode::Call, funcIdx);
    }
    break;
}
```

这里做了一个巧妙的分流：如果函数名以 `__builtin_` 开头，就生成 `BCall` 指令调用内置函数；否则生成 `Call` 指令调用普通用户函数。内置函数和普通函数在虚拟机内部的执行路径是完全不同的。

`Return` 的翻译也有点东西：

```Cpp
case IR::Instruction::Opcode::Return: {
    auto imm = std::dynamic_pointer_cast<IR::ImmediateValue>(operands[0]);
    if (imm) {
        // 如果是立即数，比如 ret i32 0
        // 先把它推到栈上，然后 Return 从栈顶取值
        VMValue val;
        if (imm->getType()->isInt32())
            val = VMValue(std::stoi(/* 从字符串中提取数字 */));
        int32_t poolIdx = addConstant(val);
        currentFunction_->addInstruction(OpCode::LoadConst, poolIdx);
    }
    currentFunction_->addInstruction(OpCode::Return);
    break;
}
```

因为我们的栈机 VM 所有操作都通过栈完成，所以返回值必须先入栈（`LoadConst`），然后 `Return` 指令再从栈顶弹出作为返回值。这也解释了为什么上方 IR 示例中 `ret i32 0` 在字节码里对应的是两条指令：

```bytecode
load_const 1   ; 加载常量 0
ret            ; 弹出栈顶并返回
```

还有一个`addConstant()`辅助函数，负责把 VMValue 添加到常量池中并返回它的索引：

```Cpp
int32_t BytecodeGenerator::addConstant(const VMValue& value) {
    constantPool_.push_back(value);
    return static_cast<int32_t>(constantPool_.size() - 1);
}
```

`getBuiltinIndex()`则维护了一个内置函数表，决定了 `BCall` 指令的操作数编号：

> 这里通过字符串还是主打一个简洁

```Cpp
int32_t BytecodeGenerator::getBuiltinIndex(const std::string& name) {
    static const std::vector<std::string> builtinTable = {
        "__builtin_print",  // 0
    };
    // 查找并返回索引
}
```

目前只有一个 `__builtin_print`，注册在索引 0 的位置。这个索引必须和虚拟机内部的内置函数表一一对应，否则就会找不到出问题。

### Virtual Machine

好了，字节码有了，最后一步就是执行它。

```Cpp
class VirtualMachine {
public:
    VirtualMachine();
    void load(const std::vector<std::shared_ptr<BytecodeFunction>>& funcs,
              const std::vector<VMValue>& constantPool);
    VMValue execute(const std::string& entryPoint = "main");

private:
    VMValue executeFunction(BytecodeFunction* func, const std::vector<VMValue>& args);

    std::vector<VMValue> stack_;        // 操作数栈
    std::vector<VMValue> constantPool_; // 常量池
    std::vector<VMValue> functionList_; // 函数列表
    std::unordered_map<std::string, int32_t> functionMap_; // 名字到函数索引的映射
    std::vector<NativeFunction> builtins_; // 内置函数表
};
```

核心就是那个 `stack_` —— 操作数栈。所有数据都在栈上流动。

先看 `load()` 函数，它就是做准备的：

```Cpp
void VirtualMachine::load(const std::vector<std::shared_ptr<BytecodeFunction>>& funcs,
                          const std::vector<VMValue>& constantPool) {
    functionList_ = funcs;
    constantPool_ = constantPool;
    functionMap_.clear();
    for (const auto& f : funcs) {
        functionMap_[f->name] = f;
    }
}
```

接收字节码函数列表和常量池，建立名称到函数的映射。

然后 `execute()` 找到入口函数，开始执行：

```Cpp
VMValue VirtualMachine::execute(const std::string& entryPoint) {
    auto it = functionMap_.find(entryPoint);
    return executeFunction(it->second.get(), {});
}
```

核心的执行逻辑在 `executeFunction()` 里，它是一个巨大的 `switch` 循环：

```Cpp
VMValue VirtualMachine::executeFunction(BytecodeFunction* func,
                                         const std::vector<VMValue>& args) {
    size_t ip = 0;  // 指令指针，指向当前执行的指令
    while (ip < func->instructions.size()) {
        const auto& inst = func->instructions[ip];
        switch (inst.opcode) {
        case OpCode::LoadConst:
            // 从常量池取数据，压栈
            push(constantPool_[inst.operand]);
            break;

        case OpCode::Call: {
            // 根据参数个数从栈上弹出参数（注意倒序）
            size_t argCount = callee->paramCount;
            std::vector<VMValue> callArgs(argCount);
            for (int i = argCount - 1; i >= 0; --i)
                callArgs[i] = pop();
            // 递归执行被调用函数
            VMValue result = executeFunction(callee, callArgs);
            if (!result.isVoid()) push(result);
            break;
        }

        case OpCode::BCall:
            // 调用内置函数，从栈顶弹一个参数
            callArgs.push_back(pop());
            VMValue result = builtins_[inst.operand](callArgs);
            if (!result.isVoid()) push(result);
            break;

        case OpCode::Return:
            if (!stack_.empty()) return pop();
            return VMValue();  // void 返回

        case OpCode::Add: {
            auto b = pop(); auto a = pop();
            if (a.isInt32() && b.isInt32())
                push(VMValue(a.asInt32() + b.asInt32()));
            break;
        }
        // Sub、Mul、Div 同理...
        }
        ++ip;  // 指令指针前进
    }
    return VMValue();
}
```

来仔细说说几个关键指令的执行过程。

LoadConst：最简单的一个。根据操作数从常量池取值，压到栈顶就行。

Call（用户函数调用）：这里有个重要的细节——参数是从栈上倒序弹出的。为什么是倒序？因为参数入栈的顺序是先第一个参数，再第二个参数……而栈是 LIFO 的，所以弹出的时候最后一个参数先出来。于是我们倒着装进 `callArgs` 数组，这样 `callArgs[0]` 就是第一个参数，符合直觉。然后递归执行被调用函数，如果有返回值就压回栈。

BCall（内置函数调用）：内置函数的调用路径更短。从栈顶弹出一个参数（目前 `__builtin_print` 就一个参数），调用注册在 `builtins_` 表中的 C++ 函数，如果有返回值就压回栈。

内置函数是怎么注册的？看看构造函数：

```Cpp
VirtualMachine::VirtualMachine() {
    builtins_ = {
        // 0: __builtin_print
        [](const std::vector<VMValue>& args) -> VMValue {
            if (!args.empty() && args[0].isString()) {
                const std::string& s = args[0].asString();
                for (char c : s) {
                    if (c == '\0') break;  // 遇到空字符停止
                    std::cout << c;
                }
            }
            return VMValue();
        },
    };
}
```

注意索引 0 对应 `__builtin_print`，和 `BytecodeGenerator::getBuiltinIndex()` 中的 `builtinTable` 顺序完全一致。这是个约定——两边都遵循同样的编号，否则就乱套了。

`__builtin_print` 的实现很简单：遍历字符串的每个字符输出到标准输出，遇到 `\0` 就停止。这是因为我们的字符串常量的 IR 文本中会带上 `\0` 结尾（可以回头看看 `Constant::toString()` 的转义逻辑），而输出的时候不应该把 `\0` 打出来。

Return：从栈顶弹出值作为返回值。如果栈为空，返回一个 `Void` 值。

算术指令：`Add`、`Sub`、`Mul`、`Div` 的处理模式一模一样——弹出两个操作数，计算结果，压回栈。栈机的好处就在这里体现出来了：指令本身完全不需要知道操作数在哪，反正都在栈顶。

### 使用示例

说了半天，这些东西怎么串起来？看到这里你可能有点晕了，我们来个完整的链路。

我们的 Hello World 程序从源代码一路走到被执行，经过了：

1. 源代码 $\rightarrow$ AST（解析器）
2. AST $\rightarrow$ 类型检查后的 AST（语义分析）
3. 类型检查后的 AST $\rightarrow$ IR Module（IRGenerator）
4. IR Module $\rightarrow$ Bytecode（BytecodeGenerator）
5. Bytecode $\rightarrow$ 执行（VirtualMachine）

前面几步在之前的教程里已经讲过了，假设我从裤兜里掏出一个 `module`，
然后我们把 `module` 丢给 `BytecodeGenerator`：

```Cpp
BytecodeGenerator generator;
auto bytecodeFuncs = generator.generate(module);
const auto& constantPool = generator.getConstantPool();
```

`generate()` 会两遍扫描，最终产生这样的字节码函数列表：

| 索引 | 函数名 | 指令 |
|------|--------|------|
| 0 | `__builtin_print` | _（外部，空）_ |
| 1 | `main` | `load_const 0` → `bcall 0` → `load_const 1` → `return` |

而常量池长这样：

| 索引 | 值 |
|------|-----|
| 0 | `"Hello World\0"` |
| 1 | `0`（整型） |

然后塞给虚拟机：

```Cpp
VirtualMachine vm;
vm.load(bytecodeFuncs, constantPool);
vm.execute("main");
```

虚拟机开始执行 `main` 函数，指令指针 `ip` 从 0 开始：

1. `load_const 0`：从常量池索引 0 取出字符串 `"Hello World\0"`，压栈。栈：`["Hello World\0"]`
2. `bcall 0`：从栈顶弹出 `"Hello World\0"`，调用 `builtins_[0]`（即 `__builtin_print`），输出 `Hello World`。栈：`[]`
3. `load_const 1`：从常量池索引 1 取出整型 `0`，压栈。栈：`[0]`
4. `return`：从栈顶弹出 `0`，作为返回值返回

控制台输出：

```
Hello World
```

但是现在有一个毛病，如果你写输出：
```ryntra
public int main() {
    __builtin_print("a\n");
    __builtin_print("b");
}
```

你会喜提 `a\\nb`，只是因为转义字符还没有 Handle，这可是下一章的事情，不要慌。

完结撒花！

****

诶诶诶这教程还没完结呢！只是第一章完结了！