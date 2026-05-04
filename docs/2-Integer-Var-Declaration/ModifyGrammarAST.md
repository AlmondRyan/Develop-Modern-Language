# 修改语法和修改 AST

欢迎来到第二部分！第二部分是考验语言、编译器健壮性的第一大难关，在这里我们引入了变量这个东西。但不要慌，我们在上一章给变量留下了不少的接口，我们只需要一一插紧即可。

## 修改语法

首先！分析我们要加的新的语法：
```Ryntra
public int main() {
    int variable = 10;
    __builtin_print(variable);
}
```

我们看出来新的变量声明和 C/C++ 大差不差，都是 `<type> <name> [= <defaultValue>]` 结构。

这里引入了一个新的运算符——等号运算符，我们先在词法规则中添加等号这个符号：
```antlr
EQUAL: '=';
```

接下来，让我们来分析语法规则。

我们需要一个单独的规则来表示函数声明。我们上面已经给了一个基本的结构，我们现在就可以照着写了。

```antlr
variableDeclaration
    : typeSpecifier IDENTIFIER (EQUAL expression)?
```

这个词法规则表示了一个变量声明，应有一个类型、后面跟一个标识符、跟一个可有可无的初始化语句。

`variableDeclaration` 是一个语句，我们可以把它插入到语句中：

```antlr
statement
    : variableDeclaration SEMICOLON
    | expression SEMICOLON
    | returnStatement
    ;
```

细心的你可能注意到了，我们用 `__builtin_print()` 打印了一个变量，但是我们现在的 expression 只允许：
```antlr
expression
    : IDENTIFIER LPAREN argumentList? RPAREN # FunctionCall
    | IDENTIFIER                             # VariableReference
    | STRING_LITERAL                         # StringLiteral
    ;
```

这几种东西。我们的变量名无处可去！不要慌，我们只需要把他加上即可：
```antlr
expression
    : IDENTIFIER LPAREN argumentList? RPAREN # FunctionCall
    | IDENTIFIER                             # VariableReference
    | STRING_LITERAL                         # StringLiteral
    | INTEGER_LITERAL                        # IntegerLiteral
    ;
```

此时重新生成 ANTLR Source，他将可以正确解析：
```
public int main() {
    int a = 10;
    __builtin_print(a);
}
```

为：

```
(program (functionDefinition public (typeSpecifier int) main ( ) (block { (statement (variableDeclaration (typeSpecifier
 int) a = (expression 10)) ;) (statement (expression __builtin_print ( (argumentList (expression a)) )) ;) (statement (r
eturnStatement return (expression 0) ;)) })) <EOF>)
```

这个美妙的解析树。

## 修改 AST

### 添加新节点

OK，我们有了语法之后，下一步该干什么？修改 AST！

首先先来看 AST 节点列表，我们这里多了两个节点，分别是 `VariableDeclarationNode` 和 `VariableNode`。

- `VariableDeclarationNode` 负责表示变量声明，举个例子，`int a = 10;`
- `VariableNode` 表示变量使用，比如我们在 `__builtin_print()` 中调用 `a`，这个 `a` 就由 `VariableNode` 表示

```Cpp
class VariableDeclarationNode : public StatementNode {
public:
    VariableDeclarationNode(std::shared_ptr<TypeSpecifierNode> type, 
                            std::shared_ptr<IdentifierNode> name, 
                            std::shared_ptr<ExpressionNode> init)
        : type(std::move(type)), name(std::move(name)), initializer(std::move(init)) {}
    // Getters
    void accept(IVisitor &visitor) override;
    std::string toString() const override;

private:
    std::shared_ptr<TypeSpecifierNode> type;
    std::shared_ptr<IdentifierNode> name;
    std::shared_ptr<ExpressionNode> initializer;
};

class VariableNode : public ExpressionNode {
public:
    VariableNode(std::shared_ptr<IdentifierNode> name) : name(std::move(name)) {}
    std::shared_ptr<IdentifierNode> getName() const { return name; }
    void accept(IVisitor &visitor) override;
    std::string toString() const override;

private:
    std::shared_ptr<IdentifierNode> name;
};
```

实现也非常的简单，在这里就不展示 `toString()` 的实现了，还是那句话：S-表达式方法解决你一切问题。

```Cpp
void VariableDeclarationNode::accept(IVisitor &visitor) {
    if (auto *v = dynamic_cast<Visitor<VariableDeclarationNode> *>(&visitor)) {
        v->visit(*this);
    }
}

void VariableNode::accept(IVisitor &visitor) {
    if (auto *v = dynamic_cast<Visitor<VariableNode> *>(&visitor)) {
        v->visit(*this);
    }
}
```

工作原理类似，根据节点类型是否可以 cast 到指定的节点类型来决定访问。

### 修改 Builder

ANTLR 给我们提供了两个新的 Visit 函数，分别叫做 `visitVariableDeclaration` 和 `visitVariableReference`，我们可以重写这两个函数（并把类型改成 `shared_ptr` 来实现 Building AST）。

我们先来看 `visitVariableReference`，因为他最简单。我们只需要两行代码：

```Cpp
std::shared_ptr<VariableNode> ASTBuilder::visitVariableReference(Ryntra::antlr::RyntraParser::VariableReferenceContext *ctx) {
    auto nameNode = createNode<IdentifierNode>(ctx->IDENTIFIER(), ctx->IDENTIFIER()->getText());
    return createNode<VariableNode>(ctx, std::move(nameNode));
}
```

第一步，我们创建 `IdentifierNode`，存储这个变量引用的名字。

第二步，我们创建 `VariableNode`，存储这个变量引用。

然后我们来看 `visitVariableDeclaration`：

```Cpp
std::shared_ptr<VariableDeclarationNode> ASTBuilder::visitVariableDeclaration(Ryntra::antlr::RyntraParser::VariableDeclarationContext *ctx) {
    auto type = visitTypeSpecifier(ctx->typeSpecifier());
    auto nameNode = createNode<IdentifierNode>(ctx->IDENTIFIER(), ctx->IDENTIFIER()->getText());
    std::shared_ptr<ExpressionNode> initializer = nullptr;
    if (ctx->EQUAL() && ctx->expression()) {
        initializer = visitExpression(ctx->expression());
    }
    return createNode<VariableDeclarationNode>(ctx, std::move(type), std::move(nameNode), std::move(initializer));
}
```

先来推导类型，然后获取名字，接下来看是否有初始化值，然后创建节点，就是这么简单。

其实你发现了吗？我们可以通俗的理解为：“如果你需要一个语法规则的值，就必须 `visit`，否则可以直接用。”

就像是你在安装一个柜子，说明书上赫然写着“安装门组件”（语法规则），你必须展开这个门组件（visit）将其组装好，然后安装在柜子上。

****

然后让我们运行程序吧！对于这个输入：

```
public int main() {
    int a = 10;
    __builtin_print(a);
    return 0;
}
```

你应该能看到这样的输出：

```
(Program (FunctionDefinition (TypeSpecifier int) (Identifier main) (Block (VariableDeclaration (TypeSpecifier int) (Iden
tifier a) (IntegerLiteral 10)) (ExpressionStatement (FunctionCall (Identifier __builtin_print) (Variable (Identifier a))
)) (Return (IntegerLiteral 0)))))
```

****

别忘了在 `NodesList.txt` 中添加新的两个 Node 哦，下一步将会用到。

```
TypeSpecifierNode
StringLiteralNode
IdentifierNode
FunctionCallNode
ExpressionStatementNode
BlockNode
FunctionDefinitionNode
ProgramNode
ReturnNode
IntegerLiteralNode
VariableNode
VariableDeclarationNode
```

下一步就是符号表和语义分析了。