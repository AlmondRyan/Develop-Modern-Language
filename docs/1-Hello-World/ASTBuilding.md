# AST 的构建

有了 AST 节点可不算完成，我们需要一个方法来构建起这棵抽象语法树，那么就是这篇文章的主题：__AST构建__（AST Building）。

## 工作原理

众所周知，ANTLR 解析之后会返回一棵 Parse Tree，幸运的是，ANTLR 也提供了 Visitor 用来访问这棵 Parse Tree。

我们的目标就是通过这个 Visitor，访问 Parse Tree 后生成我们的 AST。

还是从我们输入的源文件下手：

```
public int main() {
    __builtin_print("Hello World");
}
```

ANTLR 解析之后会变成这个样子：

```
program
├── functionDefinition
│   ├── public
│   ├── typeSpecifier
│   │   ├── INT
│   ├── main
│   ├── (
│   ├── )
│   ├── block
│   │   ├── {
│   │   ├── statement
│   │   │   ├── expression (FunctionCall)
│   │   │   │   ├── __builtin_print
│   │   │   │   ├── (
│   │   │   │   ├── argumentList
│   │   │   │   │   ├── expression
│   │   │   │   │       └── "Hello World"
│   │   │   │   └── )
│   │   │   └── ;
│   │   └── }
└── EOF
```

我们的目标就是遍历这颗极其详尽的树，分析他、生成我们的 AST。

对于下文出现的“为什么不继承 `RyntraBaseVisitor`、为什么自己写 Visitor” 问题，先来做一个统一的说明：

如果你跟着一些教程写的话，你声明 `visit` 函数的时候，声明应该是 `std::any` 类型（对于新版 ANTLR，旧版是 `antlrcpp::Any` 类型），那么为什么我们这里采取自己写的 `std::shared_ptr` 版本呢？

首先你需要先理解，`BaseVisitor` 只是一个工具类，提供了默认的 Visitor 空实现，但有一个非常明显的坏处：他用的是 `any` 类型！

如果你从没用过现代 C++，我来简单的解释一下 `std::any` 类型：和 Java 一样，用来存储任意类型的单个值。为了取用它，我们需要标准库提供的 `std::any_cast` 来获取对应类型的值。

注意到问题了吗？`std::any_cast` 有一个坏处，就是如果两个类型不兼容，会抛出 `std::bad_any_cast` 异常，并且静态分析器没有办法直接检测类型不匹配这样会导致程序异常的不稳定（只要你写错）。

所以优点有几个：

- 类型安全
- 性能更好（直接返回类型，不用先转换为 `any` 再转换为对应类型）
- 接口清晰、易于理解
- 易于测试

并且，如果你继承了 `antlr::RyntraBaseVisitor` 后修改类型，会导致虚函数隐藏，调用之后会走基类的 `std::any` 版本，导致行为错误。

所以我们对于大项目来说，不继承我们的 `RyntraBaseVisitor` 是有理由且有很多优点的。

****

## 辅助函数

我们先从辅助函数入手，因为这些辅助函数贯穿了整个类：

```Cpp
SourceLocation getLoc(antlr4::ParserRuleContext *context) {
    return SourceLocation(context->getStart()->getLine(), 
                          context->getStart()->getCharPositionInLine());
}

SourceLocation getLoc(antlr4::tree::TerminalNode *node) {
    return SourceLocation(node->getSymbol()->getLine(), 
                          node->getSymbol()->getCharPositionInLine());
}

template <typename Tp, typename... Args>
std::shared_ptr<Tp> createNode(antlr4::ParserRuleContext *ctx, Args &&...args) {
    auto node = std::make_shared<Tp>(std::forward<Args>(args)...);
    node->setLocation(getLoc(ctx));
    return node;
}

template <typename Tp, typename... Args>
std::shared_ptr<Tp> createNode(antlr4::tree::TerminalNode *node, Args &&...args) {
    auto astNode = std::make_shared<Tp>(std::forward<Args>(args)...);
    astNode->setLocation(getLoc(node));
    return astNode;
}
```

对于 `getLoc` 函数来说，作用特别简单：获取当前上下文的起始位置（行和列），并存到 `SourceLocation` 中。

我们这里用到了两个，分别是 `ParserRuleContext` 和 `TerminalNode`，他们之间的区别：

- `ParserRuleContext` 会从非终结符上下文（比如表达式上下文、语句上下文）获取起始 Token 的行列信息。
- `TerminalNode` 会从终结符上下文（比如词法符号，比如标识符、字面量、关键字）获取他的 Token 行列信息

当然，`createNode` 模板函数也用了两个，他们的作用都是创建一个 AST 节点，根据不同的传入上下文调用不同的 `getLoc()` 函数填充行号列号信息。

并且，注意到了吗？对于终结符上下文创建节点的时候，我们没有直接传入 `ctx` 上下文（甚至说根本没有这个参数），而是传入了节点和所需的参数。这很重要，在下文中会反复出现。

## 遍历入口

这棵 Parse Tree 的入口就是 `program` 规则，那么非常容易想到，我们应该从 `visitProgram()` 方法入手。先看一下声明：

```Cpp
class ASTBuilder {
public:
    std::shared_ptr<ProgramNode> visitProgram(antlr::RyntraParser::ProgramContext *ctx);
    // ...
}
```

ANTLR 会给每一个 Parser Rule 生成对应的 Context（注意大小写），这里我们用到了 `ProgramContext`。

```Cpp
std::shared_ptr<ProgramNode> ASTBuilder::visitProgram(antlr::RyntraParser::ProgramContext *ctx) {
    std::vector<std::shared_ptr<FunctionDefinitionNode>> functions;
    for (auto *funcCtx : ctx->functionDefinition()) {
        functions.push_back(visitFunctionDefinition(funcCtx));
    }
    return createNode<ProgramNode>(ctx, std::move(functions));
}
```

还是那句话，我们的 `program` 都是由一个或多个函数定义组成，也就是 `program` 这个根上的子节点。

ANTLR 依旧很善良，给我们提供了一个便捷访问子节点的方法，也就是 `functionDefinition()` 这个方法。方法名称会根据你不同的命名而不同，但是目标终究是一致的：访问子节点。

当他获得到了子节点之后，我们调用 `visitFunctionDefinition()` 方法，递归的访问子节点获取信息，并把它放到 `functions` 数组中。然后调用 `createNode` 模板函数创建节点。

****

## 遍历函数定义

上一节说了，我们会调用 `visitFunctionDefinition`，那么他的逻辑呢？这不就来了：

```Cpp
// 声明
std::shared_ptr<FunctionDefinitionNode> visitFunctionDefinition(antlr::RyntraParser::FunctionDefinitionContext *ctx);

// 实现
std::shared_ptr<FunctionDefinitionNode> ASTBuilder::visitFunctionDefinition(antlr::RyntraParser::FunctionDefinitionContext *ctx) {
    auto type = visitTypeSpecifier(ctx->typeSpecifier());
    auto nameNode = createNode<IdentifierNode>(ctx->IDENTIFIER(), ctx->IDENTIFIER()->getText());
    auto body = visitBlock(ctx->block());
    return createNode<FunctionDefinitionNode>(ctx, std::move(type), std::move(nameNode), std::move(body));
}
```

还是一模一样的定义，我就不多说了，如果没看懂的话看遍历入口这一节。

他的实现非常有趣，这里涉及到了访问语法规则和词法规则：

- 访问词法规则：通过语法规则引用的 `TerminalNode`，调用 `getText()`，`getSymbol()` 等方法；当然你可以遍历 Token 流，这是下下策甚至基本上不需要使用
- 访问语法规则：通过 `ParserRuleContext` 访问对应的子节点

还记得上面的那棵 Parse Tree 吗？`functionDefinition` 的子节点有 `public`/`typeSpecifier`/`Identifier`/`()`/`block` 这些，对于词法规则你可以使用全大写的方法来遍历，比如：

```Cpp
ctx->PUBLIC();
ctx->IDENTIFIER();
```

而对于语法规则，还是用驼峰方式访问函数：

```Cpp
ctx->typeSpecifier();
```

讲完了访问语法规则和词法规则，我们来看一下如何填充这个节点。

对于 `FunctionDefinitionNode`，我们需要 Type, Name, Body 三个参数（依旧没有 Parameters）。

所以，我们可以通过访问 `typeSpecifier` 语法规则、`IDENTIFIER` 词法规则，`block` 语法规则获得所需要的信息，然后调用 `createNode<>()` 即可填充信息。

****

## 遍历类型声明

类型声明是由一个一个的词法规则组成的，所以我们可以完全可以通过 `ctx->TYPE()` 访问，但是我们不这么干：

```Cpp
std::shared_ptr<TypeSpecifierNode> visitTypeSpecifier(antlr::RyntraParser::TypeSpecifierContext *ctx);

std::shared_ptr<TypeSpecifierNode> ASTBuilder::visitTypeSpecifier(antlr::RyntraParser::TypeSpecifierContext *ctx) {
    return createNode<TypeSpecifierNode>(ctx, ctx->getText());
}
```

我们这里直接调用了 ParserRuleContext 的 `getText()` 函数，这个函数会将所有的 `TerminalNode`（包括子节点的 `TerminalNode`）的文字拼接起来，这里这么写的原因很简单：因为以后 `typeSpecifier` 只是一个选择器和 Identifier，没有必要对每一个选项都判空然后获取文字，这也是方便的点之一。

****

## 遍历字符串字面量

```Cpp
std::shared_ptr<StringLiteralNode> visitStringLiteral(antlr::RyntraParser::StringLiteralContext *ctx);

std::shared_ptr<StringLiteralNode> ASTBuilder::visitStringLiteral(Ryntra::antlr::RyntraParser::StringLiteralContext *ctx) {
    std::string raw = ctx->STRING_LITERAL()->getText();
    std::string val = raw.substr(1, raw.length() - 2);
    // TODO: handle escape sequences if needed
    return createNode<StringLiteralNode>(ctx, val);
}
```

这里面访问了词法规则获取到了原文字，但是我们的词法规则写了他是包括引号的，我们当然不想要引号，所以我们利用 `substr` 方法去掉引号，再创建节点。

看到这里面的 TODO 注释了吗？因为我们这里暂且不需要处理转义字符（但是下一步就需要了），所以为了保证简洁，先留一个 TODO。

****

## 遍历函数体

```Cpp
// Declaration...

std::shared_ptr<BlockNode> ASTBuilder::visitBlock(antlr::RyntraParser::BlockContext *ctx) {
    std::vector<std::shared_ptr<StatementNode>> statements;
    for (auto *stmtCtx : ctx->statement()) {
        statements.push_back(visitStatement(stmtCtx));
    }
    return createNode<BlockNode>(ctx, std::move(statements));
}
```

> 在这里忽略了声明，不过你需要写！

由于函数体是由一个一个语句所组成的，并且语句是一个语法规则，我们通过 `visit` 函数来访问语法规则获得信息并放到 `vector` 中。

****

## 遍历语句

```Cpp
// Declaration...

std::shared_ptr<StatementNode> ASTBuilder::visitStatement(Ryntra::antlr::RyntraParser::StatementContext *ctx) {
    auto expr = visitExpression(ctx->expression());
    return createNode<ExpressionStatementNode>(ctx, std::move(expr));
}
```

暂且来说，我们只有 `ExpressionStatement` 这一种，所以我们直接访问其中包裹的 `expression()` 即可，然后创建节点。

****

## 遍历表达式

```Cpp
// Declaration...

std::shared_ptr<ExpressionNode> ASTBuilder::visitExpression(Ryntra::antlr::RyntraParser::ExpressionContext *ctx) {
    if (auto *callCtx = dynamic_cast<Ryntra::antlr::RyntraParser::FunctionCallContext *>(ctx)) {
        return visitFunctionCall(callCtx);
    }
    if (auto *strCtx = dynamic_cast<Ryntra::antlr::RyntraParser::StringLiteralContext *>(ctx)) {
        return visitStringLiteral(strCtx);
    }
    return nullptr;
}
```

这里和其他的不同，原因很简单：语法规则写的 expression 是一个选择器，我们需要通过 `dynamic_cast` 获取当前的类型，如果上下文不是 `nullptr`，那么证明匹配上了，就访问对应的节点。

这里是一个标准的 Dispatch（分派）原则。

****

## 遍历函数调用

```Cpp
// Declaration...

std::shared_ptr<FunctionCallNode> ASTBuilder::visitFunctionCall(Ryntra::antlr::RyntraParser::FunctionCallContext *ctx) {
    auto nameNode = createNode<IdentifierNode>(ctx->IDENTIFIER(), ctx->IDENTIFIER()->getText());
    std::vector<std::shared_ptr<ExpressionNode>> args;
    if (ctx->argumentList()) {
        args = visitArgumentList(ctx->argumentList());
    }
    return createNode<FunctionCallNode>(ctx, std::move(nameNode), std::move(args));
}
```

函数调用由 Identifier 和 Argument List 组成。Identifier 还是老生常谈的词法规则，而 Argument List 是一个语法规则，当然是可空的。

对于可空的规则，我们需要判断指针是否是 `nullptr` 然后再进行处理。

****

## 遍历参数列表

```Cpp
// Declaration...

std::vector<std::shared_ptr<ExpressionNode>> ASTBuilder::visitArgumentList(Ryntra::antlr::RyntraParser::ArgumentListContext *ctx) {
    std::vector<std::shared_ptr<ExpressionNode>> args;
    for (auto *exprCtx : ctx->expression()) {
        args.push_back(visitExpression(exprCtx));
    }
    return args;
}
```

参数列表和遍历 program 等都是一致的思路，由于参数列表是由一个或多个表达式组成的（不可能是空的），所以我们直接访问就可以了。

****

## 小结

AST 最难的一部分已经过去了！接下来就是 Acyclic Visitor 原则的使用，不要慌，我们会解决的！