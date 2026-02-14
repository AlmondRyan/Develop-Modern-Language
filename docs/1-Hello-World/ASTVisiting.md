# AST 的访问

我们前面成功的设计了 AST 的节点、构建了属于我们自己的 AST，接下来就是设计一个东西来访问我们的 AST。

## 非环形访问者模式

我们采取 Acyclic Visitor（非环形访问者）模式。此时你可能会问：什么是 Acyclic Visitor？那么好，我们现在来详细解释一下。

### 经典的 Visitor

经典的 Visitor 用于在稳定数据结构上定义新的操作，而不修改数据结构。核心的结构是：

- `Visitor` 接口：为每个具体的元素类声明一个 `visit()` 方法
- `ConcreteVisitor`：实现 `Visitor` 的接口，定义针对每个元素的具体操作
- `Element` 接口：声明 `accept(Visitor)` 方法
- `ConcreteElement`：实现 `accept()` 方法，在 `accept()` 中调用 `visitor.visit()`

这样做的目的很简单，将操作和实现分离，这样就可以不修改元素类增加新的操作。

当然，坏处也十分明显，首先就是循环依赖（Circular Dependency）问题：

```mermaid
flowchart LR
    A(Visitor) -->|需要知道所有Concrete Element 的类型| B(ConcreteElement)
    B -->|需要知道Visitor接口| A
```

Visitor 明确需要知道 Concrete Element 的类型这样才能给它生成 `visit()` 方法，而 Concrete Element 又需要 Visitor 的接口才能实现对应的行为... 死循环诶，这您受得了吗。

这种双向依赖导致：

- 添加新的 Element 特别困难，因为你添加一个 Element 会导致所有 Visitor 接口变动，如果不改你将获得 "类为 abstract" 的提示并且编译报错，并且违反了开闭原则（OCP）
- 耦合过强，Element 和 Visitor 之间紧紧绑定在一起，改动任何一个都会影响另一个

此时，Acyclic Visitor 模式就来解决这个问题。

### Acyclic Visitor 模式的核心思想

他来干什么？解决经典 Visitor 模式的循环依赖问题。

他怎么实现？

- 为每个 ConcreteElement 定义自己的 Visitor 子接口，这个接口只包含一个针对这个类型的 `visit()` 方法
- Visitor 基类的接口仅作为标记用，没有任何具体方法
- 每个 Visitor 需要实现多个 Element Visitor 接口，从而实现针对不同 Element 的操作
- Element 的 `accept()` 方法接受 Visitor 基接口，方法内部用 `dynamic_cast<>` 检查传入的 Visitor 是否实现对应的 `visit()` 接口，如果实现了，就调用；没实现，就报错

此时，Element 不再依赖包含所有 `visit()` 方法的 Visitor，此时只依赖 Visitor 标记接口和他自己对应的子接口，Visitor 接口不再耦合所有的 Element 类型。直接达成目标。

## 基类代码实现

说了这么多，我们直接来看实战。

首先，我们需要一个公共祖先 `IVisitor`，他没有任何方法，仅包括一个虚析构函数用于实现多态（Polymorphism）：

```Cpp
class IVisitor {
public:
    virtual ~IVisitor() = default;
}
```

接下来，为每个节点类型生成一个属于自己的访问者接口，注意：**没有继承 `IVisitor`**。

```Cpp
template <typename T>
class Visitor {
public:
    virtual ~Visitor() = default;
    virtual void visit(T &node) = 0;
}
```

那么此时你可能会问，为什么不继承 `IVisitor`？这是一个非常好的问题：

1. 避免菱形继承、虚继承的开销：如果 `Visitor<Tp>` 继承 `IVisitor`，具体访问者多重继承多个 `Visitor<Tp>`，必然形成菱形继承（`IVisitor` 出现多次），C++ 标准要求这样使用虚继承解决二义性和内存布局混乱，但是虚继承并非零开销。如果不继承，规避了这些所有的问题
2. 接口职责清晰：`IVisitor` 作为标记接口，`Visitor<Tp>` 作为操作接口，两种功能正交，符合单一职责原则

!!! warning "注意"

    任何希望被 `accept()` 接收的访问者必须公开继承 **IVisitor**，否则无法将实例绑定到 `IVisitor &`，导致编译错误。

## `accept()` 的实现

我们在这里举个例子，以 `StringLiteralNode` 的 `accept()` 举例子：

```Cpp
void StringLiteralNode::accept(IVisitor &visitor) {
    if (auto *v = dynamic_cast<Visitor<StringLiteralNode> *>(&visitor)) {
        v->visit(*this);
    } else {
        throw std::runtime_error("Visitor doesn't support StringLiteralNode");
    }
}
```

这里看似很难理解，其实特别好理解：

- 把传入的 `visitor` 交叉转型为 `Visitor<StringLiteralNode> *` [^1]
- 如果转型成功（该访问者支持处理 `StringLiteralNode`），就调用他的 `visit()` 方法
- 转型失败抛出异常

其他的 `accept()` 类似，比如 `ProgramNode`：

```Cpp
void ProgramNode::accept(IVisitor &visitor) {
    if (auto *v = dynamic_cast<Visitor<ProgramNode> *>(&visitor)) {
        v->visit(*this);
    } else {
        throw std::runtime_error("Visitor doesn't support ProgramNode");
    }
}
```

## 真实样例

讲了这么多理论知识，相信你现在已经跃跃欲试了，但是现在还不着急着手制作语义分析，我们先来实现一个简单的 `PrintVisitor`，接受 `IdentifierNode` 和 `StringLiteralNode`：

```Cpp
class PrintVisitor : public IVisitor,
                     public Visitor<IdentifierNode>,
                     public Visitor<StringLiteralNode> {
public:
    void visit(IdentifierNode &node) override {
        std::cout << "Identifier Node: " << node.getName() << std::endl;
    }

    void visit(StringLiteralNode &node) override {
        std::cout << "String Literal Node: " << node.getValue() << std::endl;
    }
}
```

调用也非常简单：

```Cpp
StringLiteralNode node("HELLO WORLD");
PrintVisitor visitor;
node.accept(visitor);
```

我们只需要实现对应节点的 `visit()` 即可，非常方便。

****

下一章，让我们深入符号表的创建和使用。

[^1]: 为什么可以从 `IVisitor*` 转换到 `Visitor<StringLiteralNode> *`？需要有一个条件满足：完整对象必须同时派生自 `IVisitor` 和 `Visitor<StringLiteralNode>`，这称作 Cross-Cast