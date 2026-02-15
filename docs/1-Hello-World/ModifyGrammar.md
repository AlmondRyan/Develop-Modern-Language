# 修改语法

> 本文的有关代码在 [这里](https://github.com/AlmondRyan/Ryntra-Project/blob/master/ANTLR/Ryntra.g4)

欢迎来到第一部分。在接下来的内容中，你将会多次看到修改语法这一部分。因为我们每一章都会添加新的语法，并且 ANTLR 的 `.g4` 文件描述了我们的语法，修改它是必须的。

## 理解现有的语法

如果你跟着我的环境配置部分一步步走到现在，你应有这样的语法文件：

```antlr
grammar Ryntra;

// Keywords
INT: 'int';

// Symbols & Operators
SEMICOLON: ';';
LPAREN: '(';
RPAREN: ')';
LBRACE: '{';
RBRACE: '}';

// Lexical Objects
IDENTIFIER: [a-zA-Z_][a-zA-Z_0-9]*;
WS: [ \t\r\n]+ -> skip;

// Parser Rules
program
    : function+ EOF
    ;

functionDefinition
    : typeSpecifier IDENTIFIER LPAREN RPAREN block
    ;

typeSpecifier
    : INT
    ;

block
    : LBRACE RBRACE
    ;
```

还记得我在环境设置里说过在这里讲解语法文件吗？现在就是时候了！

ANTLR 的语法文件是一种基于 EBNF（Extended Backus-Naur Form，扩展巴克斯·诺尔范式）的一种语法描述文件，它保留了 EBNF 的一些特性：

- `*` 表示零次或多次重复
- `+` 表示一次或多次重复
- `?` 表示可选
- `|` 表示选择
- `()` 表示分组

当然也添加了一些自己的特性：

- `grammar` / `lexer grammar` / `parser grammar` 标注
- 全大写表示词法规则，小写开头表示语法规则
- 自定义动作
- 局部变量、返回值、内置函数等

理解了一些基础背景知识后，让我们来看一下这个语法文件：

### 词法规则

```antlr
// Keywords
INT: 'int';

// Symbols & Operators
SEMICOLON: ';';
LPAREN: '(';
RPAREN: ')';
LBRACE: '{';
RBRACE: '}';

// Lexical Objects
IDENTIFIER: [a-zA-Z_][a-zA-Z_0-9]*;
WS: [ \t\r\n]+ -> skip;
```

还记得我说过全大写表示词法规则（Lexer Rules）吗？这里就是词法规则！

当然，ANTLR 的语法文件也是有优先级的，**越在上面优先级越高**，所以重点就是 **关键字一定要在 `IDENTIFIER` 规则的上面防止被识别为标识符**。

这里我们定义了：

- 关键字 `int`
- 符号和运算符（虽然这里没有运算符）
    - 小括号：`()`
    - 大括号：`{}`
    - 分号：`;`
- 语法对象：
    - 标识符（正则表示）
    - 空格、Tab、回车和换行的处理方式：`skip`（完全不出现在 Parse Tree 中） [^1]

### 语法规则

```antlr
program
    : function+ EOF
    ;

functionDefinition
    : typeSpecifier IDENTIFIER LPAREN RPAREN block
    ;

typeSpecifier
    : INT
    ;

block
    : LBRACE RBRACE
    ;
```

这里我们描述了四个语法对象：

- 主程序：由一个或多个函数组成，最后是 `EOF`（End of File）
- 函数：由类型、标识符、左右小括号、函数体组成
- 类型：由 `INT` 关键字组成
- 函数体：由左右大括号组成（这里留空）

也就是如果你输入：

```
int main() {

}
```

他将会这样解析：

- 主程序
    - 函数
        - 类型：int
        - 标识符：main
        - 左括号：(
        - 右括号：)
        - 函数体：
            - 左大括号：{
            - 右大括号：}
    - EOF

这样就解释了我们生成的解析树：

```
(program (function (typeSpecifier int) main ( ) (block { })) <EOF>)
```

其中每一个括号代表一个层级。

****

## 修改语法

好了，现在我们已经了解了现在的语法，那么接下来把他进行修改就如鱼得水了。如果你还是没看懂——不要慌，跟着我们一起做。

首先先来理解我们要添加的语法：

```Ryntra
public int main() {
    __builtin_print("Hello World!");
}
```

其实他就是在函数体中添加了一个内置函数调用（解析的时候直接解析为函数调用节点）。函数调用的格式一般情况下为：

```
identifier ( argument list );
```

并且，参数列表由 `,` 隔开。在这个例子中，我们只有一个参数（字符串字面量），所以我们要添加的词法规则也就只有：

- 逗号 `,`
- 字符串字面量

```antlr
// Symbols & Operators
COMMA: ',';

// Lexical Objects
STRING_LITERAL: '"' (~["\\\r\n] | '\\' .)* '"';
```

其中 STRING_LITERAL 是一个简化的字符串字面量定义，它能够匹配包含转义字符的字符串。`~["\\\r\n]` 表示除了双引号、反斜杠和换行符之外的任何字符，`'\\' .` 表示反斜杠后跟任意一个字符（支持各种转义序列）。

看到我们要添加的语法前面每一个函数声明都需要一个 `public` 了吗？我们这里暂且是把它当作占位符来处理，因为访问修饰符需要在后面实现类的时候进行实现，我们现在先把他当作占位符来实现。

所以我们需要一个 `public` 关键字：

```antlr
// Keywords
PUBLIC: 'public';
```

接下来来看主程序入口，我们这里定义每个程序都是由一个或多个函数定义组成的，所以我们修改主程序入口 `program` 语法规则为：

```antlr
program
    : functionDefinition+ EOF
    ;
```

非常好理解，其实就是整个程序由一个或多个的 `functionDefinition` 组成。

接下来，我们需要深入 `functionDefinition` 规则的处理。其实很好修改，只需要在前面加上一个 `public` 关键字即可，修改如下：

```antlr
functionDefinition
    : PUBLIC typeSpecifier IDENTIFIER LPAREN RPAREN block
    ;
```

非常简洁暴力。

对于接下来的语法规则来说，我们需要理解一个概念 —— Expression 和 Statement 的区别：

- Expression（表达式）的定义如下：
    - 有值（计算后产生值，可以被 evaluate）
    - 可以放在赋值表达式右侧
    - 可以嵌套在其他表达式中
    - 可以成为表达式语句
- Statement（语句）的定义如下：
    - 无值（只执行一个动作）
    - 构成程序的基本动作单元
    - 不能嵌套在表达式中

**Statement 可以包含 Expression**，这是一个重要的包含关系，但反过来不行。

理解了这些之后，我们就可以继续了。首先，由于语句是基本动作单元，每一个语句块（block）都是由一个一个的语句组成的，所以：

```antlr
block
    : LBRACE statement* RBRACE
    ;
```

也就不言自通了。其中 `*` 代表可能有零个或多个，这样就支持了 `public int main() { }` 这种空函数体。

接下来我们定义 `statement`。在我们的设计中，函数调用既可以有返回值（作为表达式），也可以作为独立的语句执行。最简单的方式是将所有表达式都可以作为语句：

```antlr
statement
    : expression SEMICOLON
    ;
```

这样，任何表达式加上分号就变成了一个语句。

接下来我们定义 `expression`。我们的表达式目前支持两种类型：函数调用和字符串字面量。为了更好地生成解析树，我们使用标签来区分不同的表达式类型：

```antlr
expression
    : IDENTIFIER LPAREN argumentList? RPAREN # FunctionCall
    | STRING_LITERAL                         # StringLiteral
    ;
```

这里我们直接定义了两种表达式：

- 函数调用表达式：由标识符、左括号、可选的参数列表和右括号组成
- 字符串字面量表达式：直接匹配字符串字面量

这种设计有几个优点：

1. 清晰性：函数调用直接被定义为一种表达式类型
2. 扩展性：后续可以轻松添加更多类型的表达式（如整数字面量、变量引用等）
3. 一致性：符合"函数调用既是表达式又可作为语句"的设计理念

那么为什么函数调用也是表达式呢？我们来理解一下：

表达式的定义是：有值、可以嵌套、可以成为表达式语句、可以放在赋值语句右侧。函数调用恰恰满足这些定义：

```Cpp
// 1. 有值
int result = calculate(1, 2);

// 2. 可以嵌套
int result = calculate(calculate(3, 4), 2);

// 3. 可以成为表达式语句
calculate(1, 2); // 这是一个合法的语句！

// 4. 可以作为其他函数调用的参数
print(calculate(1, 2));
```

在我们的设计中，即使是没有返回值的函数（如 `__builtin_print`），也可以作为表达式使用，只是它的值可能是特殊的 `void` 类型。这简化了初期的实现，同时为后续的类型系统留下了扩展空间。

最后是参数列表的设计，我们的语言中参数列表每个都是一个表达式，所以我们的设计是这样的：

```antlr
argumentList
    : expression (COMMA expression)*
    ;
```

这个规则表示参数列表由一个或多个表达式组成，表达式之间用逗号分隔。其中 `?` 表示整个参数列表是可选的（允许零个参数）。

这样，我们重新生成 ANTLR Source，既可正常解析：

```Ryntra
public int main() {
    __builtin_print("Hello World!");
}
```

现在我们对后面的 AST 设计已经铺下了第一块砖！