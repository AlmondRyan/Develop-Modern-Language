# AST 节点设计

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/tree/master/Compiler/AST)

欢迎来到 AST 设计部分！

在上一节中，我们已经成功修改了语法文件，让它能够解析我们的 Hello World 程序。

但是，仅仅能够解析是不够的，我们需要将解析的结果转换成编译器能够更容易处理的数据结构—— **抽象语法树（Abstract Syntax Tree, AST）**。

> 以下本文都在 `ASTNodes.h` 中进行工作

> 下文中，所有的 Getter 注释默认都是对所有私有成员的 Getter，`accept()` 的实现样例见 `ProgramNode`，`toString()` 可自行实现，推荐 S-表达式方法

## 引入

抽象语法树（AST）是一种结构化的中间表示（Intermediate Representation），它应该忠实反映我们语言的语法结构。后续的语义分析、代码生成全部基于我们的抽象语法树来进行。

让我们从我们标准的 Hello World 程序出发：
```
public int main() {
    __builtin_print("Hello World");
}
```

可以推导出我们的 AST **暂且** 至少需要表达以下结构：

- 整个程序由一个或多个函数定义组成
- 每个函数包含：
    - 访问控制修饰符（现在是占位符，不处理）
    - 返回类型（这里的 `int`）
    - 标识符（这里的 `main`）
    - 函数体
- 函数体是一系列语句（Statement）
- 其中一条语句是表达式语句（Expression Statement）
- 这个表达式是一个函数调用（Function Call）
- 调用的参数是一个字符串字面量（String Literal）

基于这个层次结构，我们自顶向下构建我们的 AST 节点。

## 基类设计

所有的 AST 节点共用一个公共接口类：`IASTNode`。从名字也可以看出他是一个接口。

```Cpp
class IASTNode : public std::enable_shared_from_this<IASTNode> {
public:
    // 虚析构函数
    virtual ~IASTNode() = default;

    // 实现 Visitor 模式的接口
    virtual void accept(IVisitor &visitor) = 0;

    // 调试用
    virtual std::string toString() const = 0;

    // 获取节点的行号列号信息
    SourceLocation getLocation() const { return location; }

    // 设置节点的行号列号信息
    void setLocation(const SourceLocation newLocation) { 
        location = newLocation; 
    }
private:
    SourceLocation location;
}
```

其中 `SourceLocation` 的定义如下：

```Cpp
struct SourceLocation {
    int line;
    int column;

    SourceLocation() = default;
    SourceLocation(const int _line, const int _column) : 
            line(_line), column(_column) {}
};
```

!!! note "为什么使用 `std::enable_shared_from_this<T>`"

    因为我们统一使用 `std::shared_ptr<IASTNode>` 管理 AST 节点生命周期。此基类允许节点安全地通过 `shared_from_this()` 获取自身的 `shared_ptr`，避免悬空指针或重复析构。

****

## 主程序节点设计

这棵树终归需要有一个根节点，那么就是这个主程序节点，代表整个翻译单元（Translation Unit）。

```Cpp
class ProgramNode : public IASTNode {
public:
    ProgramNode(std::vector<std::shared_ptr<FunctionDefinitionNode>> funcs)
            : functions(std::move(funcs)) {}
    
    // Getter，获取节点内|存的信息（这么断句，不是内存）
    const std::vector<std::shared_ptr<FunctionDefinitionNode>> &getFunctions() const {
        return functions;
    }

    void accept(IVisitor &visitor) override;
    std::string toString() const override;
private:
    std::vector<std::shared_ptr<FunctionDefinitionNode>> functions;
}
```

根据我们的语法规则，整个程序会由多个函数组成（暂且），所以我们定义整个程序就是一个存储了 `FunctionDefinitionNode` 的数组。

`accept()` 作为 Visitor 模式的接口，我们在下一章 ASTVisiting 中会详细说明这一点的，这里你只需要直到有这么个东西存在即可。

`toString()` 是用来调试用的接口，推荐使用 S-表达式方法，在这里我来举个例子：

```Cpp
std::string ProgramNode::toString() const {
    std::stringstream ss;
    ss << "(Program";
    for (const auto &func : functions) {
        ss << " " << func->toString();
    }
    ss << ")";
    return ss.str();
}
```

****

## 函数定义节点设计

> 注意：虽然这里我们没用到参数，但是参数在函数定义里也是必不可少的，在这里先用 TODO 注释写上，以后会来实现的

```Cpp
class FunctionDefinitionNode : public IASTNode {
public:
    FunctionDefinitionNode(std::shared_ptr<TypeSpecifierNode> type, 
                           std::shared_ptr<IdentifierNode> name,
                           std::shared_ptr<BlockNode> body)
        : returnType(std::move(type)), name(std::move(name)), body(std::move(body)) {}
    
    std::shared_ptr<TypeSpecifierNode> getReturnType() const {
        return returnType;
    }

    std::shared_ptr<IdentifierNode> getName() const { 
        return name; 
    }

    std::shared_ptr<BlockNode> getBody() const {
        return body; 
    }

    // ...... accept and toString ......
private:
    std::shared_ptr<TypeSpecifierNode> returnType;
    std::shared_ptr<IdentifierNode> name;
    std::shared_ptr<BlockNode> body;
    // TODO: std::vector<ParameterNode> parameter;
}
```

根据语法，我们的函数定义结构应该是：

```
public TypeSpecifier Identifier () {
    Body
}
```

- 我已经重复说过多遍 `public` 作为占位符实现
- TypeSpecifier 即为返回值类型
- Identifier 是函数的名字
- Body 是函数体

我们的 AST 设计也和其一一对应，这也满足我们的设计目标：忠实的表示我们的语法。

## 类型说明符节点设计

```Cpp
class TypeSpecifierNode : public IASTNode {
public:
    TypeSpecifierNode(const std::string &name) :
            name(name) {}
    
    const std::string &getName() const {
        return name;
    }

    // ... accept and toString ...
private:
    std::string name;
}
```

这里我们没有复杂的类型，暂时只用 `std::string` 存储类型，但这不是长久之计，后期会重构为更通用的东西。

## 表达式和语句的基类

```Cpp
class ExpressionNode : public IASTNode {};
class StatementNode : public IASTNode {};
```

- 所有语句继承 `StatementNode`
- 所有表达式继承 `ExpressionNode`

这样一是达成了分类的目标，二是使得 API 明确这里需要传入的信息，提升类型安全性。

## 字符串字面量节点设计

```Cpp
class StringLiteralNode : public ExpressionNode {
public:
    StringLiteralNode(const std::string &value) : value(value) {}

    const std::string &getValue() const { 
        return value; 
    }
    
    // ... accept and toString ...
private:
    std::string value;
};
```

## 标识符节点设计

```Cpp
class IdentifierNode : public ExpressionNode {
public:
    IdentifierNode(const std::string &name) : name(name) {}

    const std::string &getName() const { 
        return name; 
    }
    
    // ... accept & toString ...
private:
    std::string name;
};
```

## 函数调用节点设计

```Cpp
class FunctionCallNode : public ExpressionNode {
public:
    FunctionCallNode(std::shared_ptr<IdentifierNode> name, 
                     std::vector<std::shared_ptr<ExpressionNode>> args)
        : functionName(std::move(name)), arguments(std::move(args)) {}

    std::shared_ptr<IdentifierNode> getFunctionName() const { 
        return functionName; 
    }

    const std::vector<std::shared_ptr<ExpressionNode>> &getArguments() const { 
        return arguments; 
    }

    // ... accept & toString ...
private:
    std::shared_ptr<IdentifierNode>              functionName;
    std::vector<std::shared_ptr<ExpressionNode>> arguments;
};
```

函数调用是表达式（Expression），因为可能产生值。

虽然我们现在只有 `__builtin_print()` 这种内置函数调用，而且它只返回 `void`，但是这并不耽误我们把它看作表达式。

## 表达式语句节点设计

```Cpp
class ExpressionStatementNode : public StatementNode {
public:
    ExpressionStatementNode(std::shared_ptr<ExpressionNode> expr) : 
                            expression(std::move(expr)) {}

    std::shared_ptr<ExpressionNode> getExpression() const { 
        return expression; 
    }

    // ... accept & toString ...
private:
    std::shared_ptr<ExpressionNode> expression;
};
```

表达式语句很好理解，就是把表达式降级为语句，忽略返回值，保留副作用。

## 语句块节点设计

```Cpp
class BlockNode : public IASTNode {
public:
    BlockNode(std::vector<std::shared_ptr<StatementNode>> stmts) : 
              statements(std::move(stmts)) {}
    
    const std::vector<std::shared_ptr<StatementNode>> &getStatements() const { 
        return statements; 
    }

    // ... accept & toString ...
private:
    std::vector<std::shared_ptr<StatementNode>> statements;
};
```

语句块就是由语句定义而成的块。可能包含零条（空块）或多条语句。

## 小结

至此，我们已经给我们的 Hello World 程序设计了一个完整的、类型安全的、可扩展的 AST 节点体系。

每个节点：

- 继承自合适的基类
- 提供了 `accept()` 实现 Visitor 模式
- 实现了 `toString()` 方便打印 AST 用来调试
- 持有源码位置信息，方便打印错误信息
- 使用 `std::shared_ptr<>` 管理子节点，避免深拷贝的开销

接下来，我们将进入 AST 的构建（Building）部分，将 ANTLR 的 Parse Tree 转换为我们的 AST，方便接下来的语法操作。