# AST的构建和遍历

在上一节中，我们定义了节点的类型（`ASTNodes.h`），现在我们需要一种神奇的方式，来把 ANTLR 生成的解析树（Parse Tree）转换为我们的 AST。

这种方式叫做 AST Building，通常通过遍历解析树实现。非常幸运，ANTLR 4 提供了一种 `Visitor` 模式的接口，我们可以用它来访问解析树的每一个节点，然后对应的转换为我们的 AST 节点。

了解了基础的转换方式后，让我们来开始吧。

## AST 构建

> AST 构建部分我们把它写在 `ASTBuilder.h` 和 `ASTBuilder.cpp` 中

在我们的架构中，`ASTBuilder` 是用来构建 AST 的工具。

### 核心思想

ANTLR 提供了一个叫做 `XBaseVisitor` 的接口，其中 `X` 是你的 `grammar` 名称，在这里就是 Ryntra。

他的每一个 `visit` 方法（举个例子，比如 `visitStringLiteral(context)` 方法），接受一个 ANTLR 的 Context 对象，并返回我们自定义的 `IASTNode` 智能指针。

如果你跟着我做到这里，你会发现一个问题：为什么他生成的是返回 `std::any` 的方法呢？我先直接把解决方法告诉你：令他返回对应的 `std::shared_ptr<Node>` 类型，比如：

- StringLiteral 等 Literal 类型：返回 `std::shared_ptr<IASTNode>`
- Statement 这种大类，返回 `std::shared_ptr<StatementNode>`
- ...

接下来我来说为什么：

1. 为什么 ANTLR 要用 `std::any`（早期版本使用 ANTLR 自己实现的 `any` 类型）

目的有如下几个：

- 保证通用性、灵活性：ANTLR 不知道你要构建什么，你是要构建 AST，还是计算结果，还是生成 IR，没有办法预测，所以选择 `std::any` 保证灵活性，你可以自己改变
- 多语言支持：ANTLR 的 Runtime 提供了多种语言的绑定，比如 Java 用 `Object`，Python 用 `Any`，C++ 为了保证一致，也使用 `std::any`
- 支持多种返回类型：每个节点都可能返回多种类型。比如 Literal 节点返回字面量值，Expression 这类的返回一种自定义结构

其实说来说去，还是为了保证一致性和通用性。

2. 为什么我们不用 `std::any`？

- 类型确定：我们的目的是什么？构建 AST！类型必须是返回 AST 节点的
- 长期维护：我们需要类型安全，方便 IDE 解析，`std::any` 会包装多种类型并且需要 `std::any_cast<>` 转换为指定类型，如果类型不一致没有办法通过静态分析发现错误。
- 性能敏感：`std::any` 有自己的开销哦！

### 工作流程

#### 创建节点

我们为了方便创建节点，而不是写一大堆冗余、死长的样板代码，写了一个小的辅助函数 `createNode<>()`，实现如下：

```Cpp
template <typename Tp, typename... Args>
std::shared_ptr<Tp> createNode(antlr4::ParserRuleContext *ctx, Args &&... args) {
    auto node = std::make_shared<Tp>(std::forward<Args>(args)...);
    node->setLocation(getLoc(ctx));
    return node;
}
```

此时问题来了，`getLoc()` 是干什么的？别忘了，我们写的是编译器，错误位置是必须要知道的，恰巧，ANTLR 提供了接口获取源码位置，我们小小的包装一下：

```Cpp
SourceLocation getLoc(antlr4::ParserRuleContext *ctx) {
    return SourceLocation(ctx->getStart()->getLine(), ctx->getStart()->getCharPositionInLine());
}
```

`SourceLocation` 就是一个结构体，包装了 `line` 和 `column` 字段，其实理论上应该包装 `file` 字段的，但是我们后期有自己的解决方法，所以不用担心。

#### 访问程序入口

我们从这里开始遍历。`ProgramContext` 包括了我们写的 `program` 规则。实现如下：

```Cpp
std::shared_ptr<ProgramNode> ASTBuilder::visitProgram(antlr::RyntraParser::ProgramContext *context) {
    // 解析包名，先不用管
    // 解析导入，先不用管

    std::vector<std::shared_ptr<FunctionDefinitionNode>> functions;
    for (auto *functionContext : context->functionDefinition()) {
        auto function = visitFunctionDefinition(functionContext);
        functions.push_back(std::move(function));
    }

    return createNode<ProgramNode>(context, std::move(functions));
}
```

其实如果你学过树结构，你可能会看懂一点。我们来解释一下：

- 我们先定义了函数列表（因为我们规定整个程序由 functions 组成），然后遍历每一个 `functionDefinition()`
- 获得一个 `functionContext` 之后，利用访问者模式 `visit` 遍历 FunctionDefinition
- OK，获得了函数需要的东西之后，把它放在函数列表中
- 创建节点

#### 访问函数定义

这不就来了吗！如果你上文没看懂，现在你就看懂了。

先看代码：

```Cpp
std::shared_ptr<FunctionDefinitionNode> ASTBuilder::visitFunctionDefinition(antlr::RyntraParser::FunctionDefinitionContext *context) {
    std::string returnType = context->returnType()->getText();
    std::string functionName = context->IDENTIFIER()->getText();
    
    std::vector<std::shared_ptr<ParameterNode>> parameters;
    if (context->parameterList()) {
        parameters = visitParameterList(context->parameterList());
    }

    std::shared_ptr<BlockNode> body = visitBlock(context->block());

    return createNode<FunctionDefinitionNode>(context, returnType, functionName, std::move(parameters), std::move(body));
}
```

回头看一眼 `functionDefinition` 是怎么定义的：

```antlr
functionDefinition
    : returnType IDENTIFIER LPAREN parameterList? RPAREN block
    ;
```

是不是非常明确？我们 `FunctionDefinitionNode` 需要四个字段，分别是：

- 返回类型
- 函数名称
- 参数列表
- 函数体

我们都可以通过遍历节点来获得（除了 `returnType` 和 `functionName` 字段，他们是纯文本）。

#### 访问语句和表达式

对于语句和表达式，因为有多种子类型，通常需要进行类型判断和分发。这里我们只用到了 `functionCall()` 语句，对于其他的可以先不用管。

```Cpp
std::shared_ptr<StatementNode> ASTBuilder::visitStatement(antlr::RyntraParser::StatementContext *context) {
    if (context->functionCall()) {
        // 如果是函数调用语句
        auto funcCall = visitFunctionCall(context->functionCall());
        return createNode<FunctionCallStatementNode>(context, std::move(funcCall));
    }
}
```

#### 访问函数调用语句

对于函数调用，我们的语法规则是这样的