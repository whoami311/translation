# Step 4: Adding Generator Expressions

`Generator expressions` 在生成构建系统时进行评估，以生成特定于每个构建配置的信息。

`Generator expressions` 允许在许多目标属性的上下文中使用，例如 `LINK_LIBRARIES`、`INCLUDE_DIRECTORIES`、`COMPILE_DEFINITIONS` 等。它们也可以在使用命令填充这些属性时使用，例如 `target_link_libraries()`、`target_include_directories()`、`target_compile_definitions()` 等。

`Generator expressions` 可用于启用条件链接、编译时使用的条件定义、条件包含目录等。条件可以基于构建配置、目标属性、平台信息或任何其他可查询的信息。

`Generator expressions` 有几种不同的类型，包括逻辑表达式、信息表达式和输出表达式。

逻辑表达式用于创建条件输出。基本表达式是 `0` 和 `1` 表达式。`$<0:...>` 结果为空字符串，而 `$<1:...>` 结果为 `...` 的内容。它们也可以嵌套。

## Exercise 1 - Adding Compiler Warning Flags with Generator Expressions

`Generator expressions` 的一个常见用法是条件地添加编译器标志，例如语言级别或警告的标志。一个很好的模式是将这些信息与一个 `INTERFACE` 目标关联，从而使这些信息得以传播。

### Goal

在构建时添加编译器警告标志，但不对已安装的版本添加。

### Helpful Resources

- cmake-generator-expressions(7)
- cmake_minimum_required()
- set()
- target_compile_options()

### Files to Edit

- CMakeLists.txt

### Getting Started

打开文件 `Step4/CMakeLists.txt` 并完成 `TODO 1` 到 `TODO 4`。

首先，在顶级 `CMakeLists.txt` 文件中，我们需要将 `cmake_minimum_required()` 设置为 `3.15`。在本练习中，我们将使用一个在 CMake 3.15 中引入的生成器表达式。

接下来，我们添加项目所需的编译器警告标志。由于警告标志因编译器而异，我们使用 `COMPILE_LANG_AND_ID` 生成器表达式来根据语言和一组编译器 ID 控制应用哪些标志。

### Build and Run

创建一个名为 `Step4_build` 的新目录，运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建，或者通过在构建目录中运行 `cmake --build .` 进行构建。

```shell
mkdir Step4_build
cd Step4_build
cmake ../Step4
cmake --build .
```

### Solution

更新 `cmake_minimum_required()`，要求至少使用 CMake 版本 3.15：

```cmake
cmake_minimum_required(VERSION 3.15)
```

接下来我们确定系统当前使用的是哪种编译器来构建，因为警告标志会根据所用编译器而有所不同。这是通过 `COMPILE_LANG_AND_ID` 的生成器表达式完成的。我们将结果存储在变量 `gcc_like_cxx` 和 `msvc_cxx` 中，如下所示：

```cmake
set(gcc_like_cxx "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc_cxx "$<COMPILE_LANG_AND_ID:CXX,MSVC>")
```

接下来，我们添加项目所需的编译器警告标志。使用我们的变量 `gcc_like_cxx` 和 `msvc_cxx`，我们可以使用另一个生成器表达式来仅在这些变量为真时应用相应的标志。我们使用 `target_compile_options()` 将这些标志应用到我们的接口库。

```cmake
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>"
  "$<${msvc_cxx}:-W3>"
)
```

最后，我们只想在构建时使用这些警告标志。我们已安装项目的使用者不应继承我们的警告标志。为此，我们使用 `BUILD_INTERFACE` 条件将 TODO 3 中的标志包裹在一个生成器表达式中。最终完整的代码如下所示：

```cmake
target_compile_options(tutorial_compiler_flags INTERFACE
  "$<${gcc_like_cxx}:$<BUILD_INTERFACE:-Wall;-Wextra;-Wshadow;-Wformat=2;-Wunused>>"
  "$<${msvc_cxx}:$<BUILD_INTERFACE:-W3>>"
)
```
