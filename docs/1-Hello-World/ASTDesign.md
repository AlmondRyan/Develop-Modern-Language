# AST 节点设计

欢迎来到 AST 设计部分！

在上一节中，我们已经成功修改了语法文件，让它能够解析我们的 Hello World 程序。

但是，仅仅能够解析是不够的，我们需要将解析的结果转换成编译器能够更容易处理的数据结构—— **抽象语法树（Abstract Syntax Tree, AST）**。

## 基础节点设计

> 以下本文都在 `ASTNodes.h` 中进行工作

首先，我们需要一个所有节点的基类 `IASTNode`。所有的 AST 节点都将继承自它。

它定义了所有节点必须具备的能力，比如接受访问者（Visitor 模式）和生成字符串表示（用于调试）。

```Cpp
class IASTVisitor; //< 这是前向声明，用于实现 Visitor Mode

class IASTNode : public std::enable_shared_from_this<IASTNode> {
public:
    virtual ~IASTNode() = default;
    virtual std::string toString() const = 0;
    virtual void accept(IASTVisitor *visitor) = 0;

    SourceLocation getLocation() const { return location; }
    void setLocation(SourceLocation loc) { location = loc; }
private:
    SourceLocation location;
};
```

由于我们整个 AST 节点都是利用了 `std::shared_ptr`，所以 `std::enable_shared_from_this` 是很有用的。

用通俗的语言解释一下 `std::enable_shared_from_this`：如果一个类已经被 `std::shared_ptr<>` 管理，我们可以通过提供的 `shared_from_this()` 函数安全的获取指向自身的 `shared_ptr`。记得 **不要在构造函数、析构函数中调用 `shared_from_this()` 因为控制块未建立/销毁**。

那么我们为什么要用到这个呢？当然就是 Visitor 模式需要了！如果你很懵，不要慌，我们下一节介绍 Visitor 模式的时候会详细说明。

## 声明节点

我们的程序是由一系列声明组成的（目前只有函数声明）。

### 程序节点

这是整棵树的根节点。它包含了程序中的所有内容，目前主要就是函数定义列表。

```Cpp
class ProgramNode : public IASTNode {
public:
    ProgramNode(std::vector<std::shared_ptr<FunctionDefinitionNode>> funcs)
        : functions(std::move(funcs)) {}

    std::string toString() const override;
    void accept(IASTVisitor *visitor) override;

    std::vector<std::shared_ptr<FunctionDefinitionNode>> getFunctions() const {
        return functions;
    }

private:
    std::vector<std::shared_ptr<FunctionDefinitionNode>> functions;
};
```

### 函数定义节点

表示一个函数的定义，包含返回值类型、函数名、参数列表和函数体。

```cpp
class FunctionDefinitionNode : public IASTNode {
public:
    FunctionDefinitionNode(const std::string &retType,
                           const std::string &name,
                           std::vector<std::shared_ptr<ParameterNode>> params,
                           std::shared_ptr<BlockNode> b)
        : returnType(retType), functionName(name),
          parameters(std::move(params)), body(std::move(b)) {}

    std::string toString() const override;
    void accept(IASTVisitor *visitor) override;

    std::string getReturnType() const { return returnType; }
    std::string getFunctionName() const { return functionName; }
    std::vector<std::shared_ptr<ParameterNode>> getParameters() const { return parameters; }
    std::shared_ptr<BlockNode> getBody() const { return body; }

private:
    std::string                                 returnType;
    std::string                                 functionName;
    std::vector<std::shared_ptr<ParameterNode>> parameters;
    std::shared_ptr<BlockNode>                  body;
};
```

虽然 `main` 函数没有参数，但为了通用性，我们也需要定义参数节点 `ParameterNode`。

```cpp
class ParameterNode : public IASTNode {
public:
    ParameterNode(const std::string &t, const std::string &n)
        : type(t), name(n) {}
    
    // ... toString, accept, getters ...
private:
    std::string type;
    std::string name;
};
```

## 语句节点

函数体是由语句组成的。我们需要一个语句的基类 `StatementNode`。

```cpp
class StatementNode : public IASTNode {
    // 目前作为一个空基类，用于类型区分
};
```

### 代码块节点 (BlockNode)

代码块 `{ ... }` 本身也是一种特殊的结构，它包含了一组语句。

```cpp
class BlockNode : public IASTNode {
public:
    BlockNode(std::vector<std::shared_ptr<StatementNode>> stmts) 
        : statements(std::move(stmts)) {}

    void addStatement(std::shared_ptr<StatementNode> stmt) {
        statements.push_back(std::move(stmt));
    }

    std::vector<std::shared_ptr<StatementNode>> getStatements() const {
        return statements;
    }

    // ... toString, accept ...

private:
    std::vector<std::shared_ptr<StatementNode>> statements;
};
```

### 函数调用语句节点

在我们的设计中，`__builtin_print("...");` 是一个函数调用语句，就像上一节所说，因为 `__builtin_print` 返回 `void` 所以我们区分于真正返回值的节点，把他定义为函数调用语句。

```Cpp
class FunctionCallStatementNode : public StatementNode {
public:
    FunctionCallStatementNode(std::shared_ptr<FunctionCallNode> fcall) 
        : functionCall(std::move(fcall)) {}

    std::shared_ptr<FunctionCallNode> getFunctionCall() const { return functionCall; }

    // ... toString, accept ...
private:
    std::shared_ptr<FunctionCallNode> functionCall;
};
```

## 表达式节点

表达式是有值的节点。

### 函数调用节点

虽然 `__builtin_print` 被用作语句，但从结构上讲，它是一个函数调用。我们将函数调用的核心逻辑封装在 `FunctionCallNode` 中，这样它既可以被 `FunctionCallStatementNode` 包含，也可以作为表达式的一部分（例如 `int a = func();`）。

```cpp
class FunctionCallNode : public IASTNode {
public:
    FunctionCallNode(const std::string &n, std::vector<std::shared_ptr<IASTNode>> arg)
        : functionName(n), arguments(arg) {}

    std::string getFunctionName() const { return functionName; }
    std::vector<std::shared_ptr<IASTNode>> getArguments() const { return arguments; }

    // ... toString, accept ...
private:
    std::string                            functionName;
    std::vector<std::shared_ptr<IASTNode>> arguments;
};
```

### 字符串字面量节点

最后，我们需要表示 `"Hello World!"` 这个字符串。它是字面量（Literal）的一种。

```cpp
class LiteralNode : public IASTNode {};

class StringLiteralNode : public LiteralNode {
public:
    StringLiteralNode(std::string val) : value(val) {}

    std::string getValue() const { return value; }

    // ... toString, accept ...
private:
    std::string value;
};
```

看着这些类定义，你可能会想：“只有声明没有实现吗？”。是的，为了保持代码结构的清晰，我们将声明放在 `.h` 文件中，而将 `toString` 等方法的实现放在 `.cpp` 文件中。当然，更重要的是 `accept` 方法的实现，这关系到我们如何在 AST 上进行操作（比如生成代码）。

在下一节中，我们将介绍 **访问者模式（Visitor Pattern）**，并实现 AST 的构建器。
