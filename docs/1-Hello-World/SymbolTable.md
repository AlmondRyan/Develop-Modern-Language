# 符号表的设计

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/Semantic)

我们终于结束了 AST 的旅程！当然不要太开心，接下来每次加入新功能都会重新修改 AST —— 但是没有这么麻烦了！

今天让我们来看一下符号表（Symbol Table）的设计。符号表是我们接下来添加变量系统的根本，没有它，就没有变量、函数等等。

## 工作原理

符号表（Symbol Table）是核心数据结构之一，用于在编译过程中存储和检索标识符（Identifier）的信息。信息包括变量名、函数名、类型、作用域等。

一般情况下，符号表用于管理语义分析阶段的符号信息。

我们采取经典的作用域栈（Scope Stack）设计模式，这种可以自然地处理嵌套作用域。我相信你脑袋里肯定出现的是这种登山阶梯：

```
{
    {
        {
            {
                //...
            }
        }
    }
}
```

我没有说你说的不对，但是其实嵌套作用域包括了一个更广泛的定义，比如在全局作用域（是的，全局也是一个作用域）定义函数、函数中定义变量、`if` 中定义变量、函数中定义 lambda 等等，都算作嵌套作用域。

接下来让我们来深入作用域栈这个模型，其实特别好理解。

### 作用域栈模型

作用域栈结构使用一个栈来模拟，我们这里选择 `std::vector`，当然你可以选择 `std::stack`，这个栈保存当前活跃的所有作用域。

什么叫活跃作用域？就是当前被处理的作用域，或者说，最近被压入栈的作用域。

每当编译器在语义分析的时候遇到一个标识符，会从内部向外部（栈顶向栈底）查找这个标识符的声明，直到找到或查找到栈底（找不到声明）。

举个例子，现在我们有这样的一个代码：

```java
public void process() {
    // process for something...
}

public void foo() {
    int v = 10;
    if (v == 10) {
        int z = 20;
        // 此时活跃作用域是 if 块作用域
    }
    // 活跃作用域是 foo 的作用域
}

// 活跃作用域是全局作用域
```

其中，`process()` 和 `foo()` 都是位于全局作用域的符号，而 `foo()` 中的 `v` 位于 `foo` 的局部作用域，`z` 位于 `if` 块的块级作用域。

因为当前处理在 `if` 块内部，所以活跃作用域就是 `if` 块作用域，当遇到一个大括号的时候，弹出一级作用域，当再遇到一个大括号的时候，再弹出一级作用域，以此类推...

看懂了吗？就是离你现在处理位置最近的一个大括号就是当前的活跃作用域。[^1]

对于作用域栈来说，最简单的理解就是遇到左大括号压栈，右大括号出栈。

## 类结构设计

讲完理论知识之后，让我们聚焦到代码上。对于整个符号表设计，我们需要三个核心类：符号表（`SymbolTable`）、符号（`Symbol`）、函数符号（`FunctionSymbol`）[^2]。

### 符号基类

```Cpp
class Symbol {
public:
    virtual ~Symbol() = default;
    explicit Symbol(std::string name) : name(std::move(name)) {}
    const std::string& getName() const { return name; }
private:
    std::string name;
};
```

作为所有符号的抽象基类，所有的符号都会派生自这个类，包括 `FunctionSymbol` 也是派生自 `Symbol`。

你会发现这里只有名字，没有类型，也没有位置信息。这确实是一个比较简化的设计——毕竟这只是个基类嘛。至于位置信息（SourceLocation），目前的实现是在 `define` 的时候单独传入用于报错，虽然有点偷懒，但也够用了。

### 函数符号

```Cpp
class FunctionSymbol : public Symbol {
public:
    FunctionSymbol(std::string name, std::string returnType, std::vector<std::string> paramTypes)
        : Symbol(std::move(name)), returnType(std::move(returnType)), paramTypes(std::move(paramTypes)) {}
    
    const std::string& getReturnType() const { return returnType; }
    const std::vector<std::string>& getParamTypes() const { return paramTypes; }

private:
    std::string returnType;
    std::vector<std::string> paramTypes;
};
```

函数符号，顾名思义，表示函数的符号，在构造函数中初始化了基类 `Symbol`，并初始化了返回类型和参数的类型列表。

这里我们暂时用 `std::string` 来表示类型（比如 "int", "void"），这在后期肯定是不够用的（毕竟还要支持自定义类型），但为了不让步子迈得太大，我们先这样凑合着用。

为什么需要参数类型列表？这对于语义分析中的函数重载解析和类型检查至关重要。

### 符号表管理器

```cpp
class SymbolTable {
public:
    SymbolTable();
    
    void enterScope();
    void exitScope();
    
    void define(std::shared_ptr<Symbol> symbol, SourceLocation location);
    std::shared_ptr<Symbol> resolve(const std::string& name);

private:
    std::vector<std::unordered_map<std::string, std::shared_ptr<Symbol>>> scopes;
};
```

`scopes` 是一个 `std::vector`，充当栈的角色，这也就是我们说的作用域栈。栈中的每个元素是一个 `std::unordered_map`，也就是每一级作用域都会维护一个属于自己的符号表。

这里使用了 `std::shared_ptr` 来管理符号的生命周期，这意味着符号的所有权是共享的，即使作用域销毁了，如果你在别处还持有这个符号的指针，它也不会消失（虽然在这个简单的编译器里不太可能发生这种情况）。

## 关键操作

### 作用域管理

作用域管理非常简单，只有两个关键操作，分别是 `enterScope()` 和 `exitScope()`，从名字就可以看出来分别是压栈和弹出栈。

操作是这样的：

- 压栈（`enterScope()`）：压入一个新的哈希表
- 弹栈（`exitScope()`）：从向量末尾移出一个哈希表。注意，如果在全局作用域还尝试弹出（虽然理论上解析器不会这么干），我们需要报错，防止程序崩溃。随着哈希表的销毁，引用计数跟着减少，当减到零的时候自动销毁。

### 符号定义

```Cpp
void SymbolTable::define(std::shared_ptr<Symbol> symbol, SourceLocation location) {
    if (scopes.empty()) {
        ErrorHandler::getInstance().makeError("SymbolTable: No scope to define symbol", location);
        return;
    }
    auto& currentScope = scopes.back();
    if (currentScope.find(symbol->getName()) != currentScope.end()) {
        ErrorHandler::getInstance().makeError("Symbol '" + symbol->getName() + "' is already defined...", location); // 重复定义检查
        return;
    }
    currentScope[symbol->getName()] = std::move(symbol);
}
```

原理也特别简单，就始终在当前作用域（`scopes.back()`）中插入符号。

这里包含了一个很重要的检查：**重复定义检查**。如果在当前作用域中找到了同样的符号，就报错。

但是需要注意，**这里只检查当前作用域**。如果外层作用域有一个叫 `x` 的变量，你在内层作用域再定义一个 `x`，这是允许的，这叫**遮蔽（Shadowing）**。

### 符号解析

```Cpp
std::shared_ptr<Symbol> SymbolTable::resolve(const std::string& name) {
    for (auto it = scopes.rbegin(); it != scopes.rend(); ++it) {
        auto found = it->find(name);
        if (found != it->end()) {
            return found->second;
        }
    }
    return nullptr;
}
```

这里我们用到了链式查找，使用反向迭代器从栈顶（最内层）向栈底（全局）查找，返回遇到的第一个匹配项。如果找遍了所有的作用域都没找到，那就只能两手一摊，返回 `nullptr` 了。

## 内置符号注册

还记得我们输出用的是 `__builtin_print()` 吗？我们当时说的是这个是内置函数，既然是内置函数我们就应该直接注册：

```Cpp
SymbolTable::SymbolTable() {
    enterScope(); // 初始化全局作用域

    // 这是一个临时的占位符，真正的内置函数机制可能会更复杂
    define(std::make_shared<FunctionSymbol>("__builtin_print", "void", std::vector<std::string>{"string"}),
        SourceLocation(0, 0));
}
```

我们在构造函数里先调用 `enterScope()` 进入全局作用域，然后调用 `define` 创建一个新的函数符号，分别是名称、返回类型、参数类型。

因为是内置的，它没有具体的源代码位置，所以我们传入 `SourceLocation(0, 0)` 作为占位符。

****

我们已经成功地设计了符号表，它为我们后期的变量以及类之类的打了基础。虽然现在的 `Symbol` 类看起来有点单薄（连个类型字段都没有），但别担心，面包会有的，牛奶也会有的。接下来让我们深入语义分析。

[^1]: 用大括号来说实际上并不准确，例如 Python 用缩进表示，Shell 用 `if`/`fi` 等表示作用域，但是这样是最好理解的
[^2]: 暂且的符号表只有这些类，到了后期还会有 `ClassSymbol` 之类的
