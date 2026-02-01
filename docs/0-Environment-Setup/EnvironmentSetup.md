# 环境配置

欢迎来到最最最基础的部分，并且也是实打实最重要的部分——环境配置！这里假设你已经学会了如何使用 `git`, `vcpkg` 等工具，我们使用 JetBrains CLion 作为开发 IDE。

## 安装 `vcpkg`

- 克隆 `vcpkg` 仓库：
```Bash
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
```

- 进入 `vcpkg` 仓库中，根据你的系统不同运行不同的 Bootstrap 脚本，其中：

| 系统 | 文件 |
|:-- | :-- |
| Windows | `bootstrap-vcpkg.bat` |
| macOS/Linux | `bootstrap-vcpkg.sh` |

- 设置环境变量（可选，但推荐）：

```PowerShell
# Windows
$env:VCPKG_ROOT = "vcpkg-path"
[Environment]::SetEnvironmentVariable("VCPKG_ROOT", $env:VCPKG_ROOT, "User")
```

```Shell
# macOS / Linux
echo 'export VCPKG_ROOT="$HOME/vcpkg"' >> ~/.bashrc
echo 'export PATH="$VCPKG_ROOT:$PATH"' >> ~/.bashrc
```

## 设置开发环境

打开 JetBrains CLion，新建 C++ Executable 项目，此时你应看到一个 Hello World 例子。

接下来开始设置 `CMakeLists.txt`：

### 配置 vcpkg 工具链

```CMake
cmake_minimum_required(VERSION 3.15)
project(RyntraProject)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_TOOLCHAIN_FILE "vcpkg-path" CACHE STRING "vcpkg toolchain")

add_executable(RyntraProject main.cpp)
```

此时，配置好后 **一定一定记住删除 `cmake-build-debug` 和 `cmake-build-release`（如有）文件夹，并重新配置 CMake 项目**！

### 安装 ANTLR4 Runtime

> 因为我们的项目词法分析器、语法分析器基于 ANTLR4，所以这一步是必要的。本教程为了方便选择 ANTLR。

打开终端，运行 `vcpkg install antlr4-runtime`，此时 vcpkg 会自动安装 ANTLR4 Runtime。

### 配置 CMake 项目

> 修改 `TargetName` 为你的目标名称

```CMake
find_package(antlr4-runtime REQUIRED)
target_link_libraries(TargetName PRIVATE antlr4_shared)
```

!!! note "如果 CMake 提示找不到 ANTLR 怎么办"
    
    如果 CMake 提示找不到 ANTLR Runtime，可以打开 vcpkg 安装路径，并导航到 `vcpkg-dir/installed/triplet/share/antlr4-runtime`，复制路径，设置为 `antlr4-runtime_DIR` 后重新加载 CMake 项目。

## 测试开发环境

首先，先来在任意地方新建一个 `Ryntra.g4` 文件（或你自己的命名），虽然说是任意地方，但是放在一个清晰的文件夹中永远是好习惯。

在 `.g4` 文件中，添加如下的 ANTLR 代码：
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

这里不多赘述关于此 `.g4` 文件的内容和作用，我们在实现 Hello World 的章节（下一章）会仔细说这段 ANTLR 语法的作用和意义。

接下来，你可创建一个 `.ps1` 或 `.sh` 文件，调用他就可以生成对应的 ANTLR Source，我在此处给出一个 PowerShell 例子：

> 替换 `ANTLR_JAR_DIR` 和 `NAMESPACE` 为你自己的内容。
> 此脚本会把 ANTLR 生成的文件放在 `antlr\antlr-generated\antlr` 文件夹下，由于大小写不敏感所以只要符合这些字母都可以被存放。

```PowerShell
java -jar ANTLR_JAR_DIR -Dlanguage=Cpp -visitor -package NAMESPACE -o .\antlr\antlr-generated .\antlr\Ryntra.g4

if ($LASTEXITCODE -ne 0) {
    Write-Error "Error occurred during generate ANTLR Sources."
    exit $LASTEXITCODE
} else {
    Write-Output "Done."
}
```

调用它，生成 ANTLR 的 Lexer 和 Parser 以及 Visitor。接下来，在 CMake 中将这些文件添加到 `add_executable()` 中：

```CMake
set(ANTLR_SOURCE
    ANTLR/antlr-generated/antlr/RyntraBaseListener.cpp
    ANTLR/antlr-generated/antlr/RyntraBaseVisitor.cpp
    ANTLR/antlr-generated/antlr/RyntraLexer.cpp
    ANTLR/antlr-generated/antlr/RyntraParser.cpp
    ANTLR/antlr-generated/antlr/RyntraListener.cpp
    ANTLR/antlr-generated/antlr/RyntraVisitor.cpp
)

add_executable(RyntraProject main.cpp ${ANTLR_SOURCE})
target_include_directories(RyntraProject PRIVATE ANTLR/antlr-generated)
```

然后，在 `main.cpp` 中添加如下的内容：

```Cpp
#include <iostream>
#include <antlr4-runtime.h>
#include <antlr/RyntraLexer.h>
#include <antlr/RyntraParser.h>
int main() {
    std::string Source = R"(int main() {
})";

    antlr4::ANTLRInputStream input(Source);
    Ryntra::antlr::RyntraLexer lexer(&input);
    antlr4::CommonTokenStream tokens(&lexer);
    tokens.fill();
    Ryntra::antlr::RyntraParser parser(&tokens);
    auto tree = parser.program();
    std::cout << tree->toStringTree(&parser) << std::endl;

    return 0;
}
```

此时，点击运行——你应该会看到如下的解析树：
```
(program (function (typeSpecifier int) main ( ) (block { })) <EOF>)
```

!!! note "如果提示 0xC0000135 怎么办"

    确保你在你的 `cmake-build-debug` 中复制了对应的 `.dll` 文件，如果是 debug，请复制 `vcpkg-install-dir\installed\triplet\debug\bin` 下的 `antlr4-runtime.dll`，否则是 `vcpkg-install-dir\installed\triplet\bin` 下的 `antlr4-runtime.dll`

恭喜你！你现在已经拥有了开发编译器（除 LLVM 库）中的整个开发环境。