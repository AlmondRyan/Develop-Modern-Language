# 修改语法

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

function
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

ANTLR 的语法文件是一种基于 EBNF（扩展巴克斯·诺尔范式）的一种语法描述文件，它保留了 EBNF 的一些特性：

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
- 符号和运算符（这里没有运算符）：
    - 小括号：`(` `)`
    - 大括号：`{` `}`
    - 分号：`;`
- 语法对象：
    - 标识符（正则表示）
    - 空格、Tab、回车/换行的处理方式：`skip`（完全不生成在 Parse Tree 中）

### 语法规则

```antlr
program
    : function+ EOF
    ;

function
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

- 主程序：由一个或多个函数组成，最后是 EOF（End of File）
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
        - 类型：`int`
        - 标识符：`main`
        - 左括号：`(`
        - 右括号：`)`
        - 函数体：
            - 左大括号：`{`
            - 右大括号：`}`
    - EOF

这样就解释了我们生成的解析树：
```
(program (function (typeSpecifier int) main ( ) (block { })) <EOF>)
```

其中每一个括号代表一个层级。

## 修改现有的语法

好了，现在我们已经了解了现在的语法，那么接下来把他进行修改就如鱼得水了。如果你还是没看懂——不要慌，跟着我们一起做。

首先先来理解我们要添加的语法：
```
int main() {
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
STRING_LITERAL: '"' ( ~["\\\r\n] | '\\' ["\\bfnrt] )* '"';
```

其中 `STRING_LITERAL` 直接内置对转义字符的支持，方便我们后期处理 `\n` 等这样的转义字符。（注意这里我们支持了 `\b`, `\f`, `\n`, `\r`, `\t` 等常见转义）。

对于语法规则来说，我们需要理解一个概念—— Expression 和 Statement 的区别：

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

理解了这些之后，我们就可以来修改语法规则了。首先，由于语句是基本动作单元，每一个语句块（block）都是由一个一个的语句组成的，所以：

```antlr
block
    : LBRACE statement* RBRACE
    ;
```

也就不言自通了。

接下来我们定义 `statement`。在我们的设计中，函数调用既可以是语句（比如 `doSomething();`），也可以是表达式（比如 `int a = getSomething();`）。为了支持这两种情况，特别是考虑到 `void` 返回值的函数（它们不能作为表达式的一部分），我们明确地将函数调用作为一种语句类型：

```antlr
statement
    : functionCall SEMICOLON
    | expression SEMICOLON
    ;
```

当然，`expression` 本身作为一个语句（表达式语句）也是必须的。

接下来我们定义 `functionCall` 规则，把它单独提取出来是为了复用：

```antlr
functionCall
    : IDENTIFIER LPAREN argumentList? RPAREN
    ;
```

然后是 `expression`。既然函数调用也可以有返回值（非 `void`），那么它也应该是一个表达式。同时，我们的参数是字符串字面量，字面量当然也是表达式：

```antlr
expression
    : functionCall
    | STRING_LITERAL
    ;
```

（注：在完整的语法中 `expression` 会复杂得多，包含各种运算符优先级，这里我们为了实现 Hello World 先简化处理）。

那么问题来了，为什么函数调用既是语句又是表达式呢？这个需要这么理解：

我们上文说了，表达式的定义是：有值、可以嵌套、可以成为表达式语句、可以放在赋值语句右侧，函数调用恰恰满足这些定义，我们来举个例子：
```Cpp
int calculate(int n, int m) {
    return n + m;
}

int main() {
    using namespace std;
    // 1. 有值
    cout << calculate(1, 2) << endl;

    // 2. 可以嵌套
    calculate(calculate(3, 4), 2);

    // 3. 可以成为表达式语句
    calculate(1, 2); // 没错，这是语句！

    // 4. 可以放在赋值表达式右侧
    int result = calculate(1, 2);
}
```

所以函数调用也是一个表达式。但是我们需要注意一个事情，`void` 类型：

- 在 C++/C#/C/Java 中，`void` 作为一个特殊的类型处理，不可以赋值
- 在 Rust/Python/TypeScript 中，`void` 可以被赋值，Rust 里作为 `()` 返回，Python 里作为 `None` 返回。

我们的语言设计 `void` 和 C#/Java 很类似，不可以赋值，不可以声明变量。为了严谨地处理 `void` 函数调用（即它只能作为语句出现，不能作为表达式参与运算），我们在 `statement` 规则中显式添加了 `functionCall SEMICOLON`。

最后是参数列表的设计了，我们的语言中参数列表每个都是一个表达式，所以我们的设计是这样的：

```antlr
argumentList
    : expression (COMMA expression)*
    ;
```

这样，我们重新生成 ANTLR Source，既可正常解析：
```
int main() {
    __builtin_print("Hello World");
}
```

为：

- 程序
    - 函数 `int main()`
        - 类型：`int`
        - 标识符：`main`
        - 括号：`()`
        - 语句块
            - 左大括号 `{`
            - 语句 `__builtin_print("Hello World");`
                - 函数调用 (`functionCall`)
                    - 标识符 `__builtin_print`
                    - 左括号 `(`
                    - 参数列表 (`argumentList`)
                        - 表达式 (`expression`) -> 字面量：`Hello World`
                    - 右括号 `)`
                - 分号 `;`
            - 右大括号 `}`
    - EOF

现在我们对后面的 AST 设计已经铺下了第一块砖！