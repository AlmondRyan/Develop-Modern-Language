# 欢迎

欢迎来到《从零开始开发现代编程语言》！这是一本基于 MkDocs 的电子书（同时连载在知乎）。在这里，你将会学到如何利用现代 C++ 开发一门编程语言的编译器和 VM，以及对应的工具链开发（IDE、反编译器、调试器、标准库）。

本书适合有一定编译器开发基础的人阅读。如果你没有编译器开发基础，请至少学习 **编译过程、AST、语义分析** 后阅读本书，本书不会直接讲解基础知识。

本书将带你开发一门叫做 Ryntra 的通用编程语言，下文将介绍 Ryntra 的一些基础语法，方便你对 Ryntra 进行简单的认识。

如果你准备好了，那么点击左侧的导航栏，开始配置开发环境吧！希望你能从中学到一些知识，开发出你梦寐以求的编程语言！

## Ryntra 语言

### 基础形式

Ryntra 语言的源文件由如下格式组成：

```
<Package Definition>
[Import Statements]

[Class Definition]
[Function Definition]

[Main Function (Entry Function)]
```

- Package Definition（包声明，必须）：在每一个源文件中，**必须只能有一个** 包声明，并且应和其他源文件的包名不相同，支持嵌套包名如 `Package.PackageLayer0.PackageLayer1`。通常形为 `declare(package) PackageName`。
- Import Statements（导入语句，可选）：源文件可能有零个或多个导入语句，通常形为 `import PackageName`，代表导入包 —— 当然也可以不导入，若你的源文件只用到了语言特性而非任何标准库特性，完全可以不导入任何包。
- Class Definition（类声明，可选）：你的源文件可以包括类声明，如果需要的话。
- Function Definition（函数声明，可选）：你的源文件也可以包括函数声明，如果需要的话。
- Main Function（主函数声明，必须/可选）：你的每一个 Target[^1] 都 **必须只能有一个** 入口函数（主函数），整个程序都会在这里开始运行，就像其它语言一样。

[^1]: 暂且不考虑构建系统的开发，至少在整个语言和标准库成型之前，不会考虑开发构建系统，这里假设每一个项目就是一个 Target。