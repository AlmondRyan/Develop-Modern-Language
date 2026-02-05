# AST 节点设计

欢迎来到 AST 设计部分！

在上一节中，我们已经成功修改了语法文件，让它能够解析我们的 Hello World 程序。

但是，仅仅能够解析是不够的，我们需要将解析的结果转换成编译器能够更容易处理的数据结构—— **抽象语法树（Abstract Syntax Tree, AST）**。

## 基础节点设计

> 以下本文都在 `ASTNodes.h` 中进行工作

首先，我们需要一个基础语法节点，我们这里称作 `IASTNode`：

```Cpp
class IASTNode : public std::enable_shared_from_this<IASTNode> {
public:
    virtual ~IASTNode() = default;
    virtual void        accept(IASTVisitor &visitor) = 0;
    virtual std::string toString() const = 0;
};
```

这里定义了 `IASTNode` 类，从名字也可以看出，这是一个接口类（以 `I` 开头，即 Interface），接下来的所有节点都将基于这个接口类进行衍生。

!!! note "什么是 `std::enable_shared_from_this<T>`"

    因为我们所有的节点都是通过 `std::shared_ptr<>` 进行内存管理的，`std::enable_shared_from_this` 放在基类可以让其使用 `shared_from_this()` 安全的获取自身的 `shared_ptr`。

## 主程序节点设计

我们无论如何都需要一个入口嘛。这个入口就是主程序节点，我们称作 `ProgramNode`。

```Cpp
class ProgramNode : public IASTNode {
// 这里声明的默认是 private 访问权限
    std::vector<std::shared_ptr<FunctionNode>> functions;
public:
    ProgramNode(std::vector<std::shared_ptr<FunctionNode>> funcs) 
                            : functions(std::move(funcs)) {}

    const std::vector<std::shared_ptr<FunctionNode>> &getFunctions() const { 
        return functions; 
    }

    void accept(IASTVisitor &visitor) override;
    std::string toString() const override;
};
```

由于我们语法设计的时候整个 Program 只接受 functions，