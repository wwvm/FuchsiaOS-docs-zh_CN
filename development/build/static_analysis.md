<!-- # Run Clang static analysis -->
# 运行 Clang 静态分析

<!-- Static analysis is a way of analyzing source code without
executing it. One of its applications is to find
[code smells](https://en.wikipedia.org/wiki/Code_smell) and bugs. -->
静态分析是一种不执行源代码的分析手段。其应用之一就是发现[代码异味](https://en.wikipedia.org/wiki/Code_smell)以及缺陷。

<!-- Fuchsia uses Clang as its compiler. Clang has several tools
to analyze the code statically. Fuchsia enables a large set
of useful warning messages and compiles with warnings as errors. -->
Fuchsia 使用 Clang 作为编译器。Clang 有一些工具可以对代码做静态分析。Fuchsia
开启大量有用告警信息也能将告警当成错误提示。

<!-- ## Prerequisites -->
## 前提条件

<!-- Before you do static analysis, make sure you have the following: -->
在进行静态分析之前，确保您有如下内容：

<!-- * Toolchain: You can either use the prebuilt toolchain or compile compile a toolchain. For
  information on compiling a toolchain, see the [toolchain guide][toolchain].
  Note: This guide assumes that you are using the prebuilt toolchain.
* Compilation database: You need the compilation database to use `clang-tidy` and Clang static
  analyzer. It is created at the root of your build directory automatically by `fx set`.
   -->
* 工具链：您可以直接使用预编译的工具链或者自行编译一个工具链。编译工具链参考 [工具链指南][toolchain]。
注意：该指南假定您使用预编译的工具链。
* 编译数据库：您需要编译数据库以使用 `clang-tidy` 和 Clang 静态分析器。它由 `fx set` 自动在您构建的根目录中创建。

<!-- ## Clang tidy -->
## Clang 清理

<!-- There is a more detailed guide available [here][lint]. -->
更详细指南在 [这里][lint]。

<!-- ## Clang static analyzer -->
## Clang 静态分析器

<!-- ### Prerequisites -->
### 前提条件

<!-- Install `scan-build-py`: -->
安装 `scan-build-py`

```
pip install scan-build --user
```

<!-- You might get a warning that `~/.local/bin` is not part of the `PATH`. Either
add it to your `PATH` environment variable or install `scan-build` globally (without the `--user` flag). -->
如果 `~/.local/bin` 不在 `PATH` 中，您可能会收到一条警告信息。将 `~/.local/bin` 加到 `PATH` 环境变量或者全局
安装（去掉 `--user` 标记） `scan-build` 均可解决。

<!-- ### Run -->
### 运行

<!-- From your Fuchsia directory, run the Clang static analyzer: -->
从您的 Fuchsia 目录中运行 Clang 静态分析器：

```
analyze-build --cdb compile_commands.json --use-analyzer path/to/checkout/prebuilt/third_party/clang/linux-x64/bin/clang --output path/to/output
```

<!-- ### View the results -->
### 查看结果

<!-- View the results of Clang static analyzer with Chrome: -->
使用 Chrome 浏览器查看 Clang 静态分析器分析结果：

```
chrome path/to/output/scan-build-date-hash/index.html
```

<!-- ## Resources -->
## 资源

<!-- * [Clang Tidy](https://clang.llvm.org/extra/clang-tidy/)
* [Clang Static Analyzer](https://clang.llvm.org/docs/ClangStaticAnalyzer.html)

[toolchain]: /docs/development/build/toolchain.md
[lint]: /docs/development/languages/c-cpp/lint.md -->
* [Clang 清理](https://clang.llvm.org/extra/clang-tidy/)
* [Clang 静态分析器](https://clang.llvm.org/docs/ClangStaticAnalyzer.html)

[工具链]: /docs/development/build/toolchain.md
[lint]: /docs/development/languages/c-cpp/lint.md
