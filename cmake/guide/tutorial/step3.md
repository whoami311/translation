# Step 3: Adding Usage Requirements for a Library

## Exercise 1 - Adding Usage Requirements for a Library

目标参数的使用需求允许对库或可执行文件的链接和包含行进行更好的控制，同时也提供了对 CMake 中目标传递属性的更多控制。利用使用需求的主要命令包括：

- target_compile_definitions()
- target_compile_options()
- target_include_directories()
- target_link_directories()
- target_link_options()
- target_precompile_headers()
- target_sources()

### Goal

为库添加使用需求。

### Helpful Resources

- CMAKE_CURRENT_SOURCE_DIR

### Files to Edit

- MathFunctions/CMakeLists.txt
- CMakeLists.txt

### Getting Started

在本练习中，我们将重构“Adding a Library”中的代码，以使用现代 CMake 方法。我们将让库定义自己的使用需求，以便这些需求可以传递给其他目标。在这种情况下，`MathFunctions` 将自行指定任何所需的包含目录。然后，使用该库的目标 `Tutorial` 只需链接到 `MathFunctions`，而无需担心任何额外的包含目录。

起始源代码位于 `Step3` 目录中。在本练习中，完成 `TODO 1` 到 `TODO 3`。

首先，在 `MathFunctions/CMakeLists.txt` 中添加对 `target_include_directories()` 的调用。记住，`CMAKE_CURRENT_SOURCE_DIR` 是当前正在处理的源目录的路径。

然后，更新（并简化！）顶级 `CMakeLists.txt` 文件中的 `target_include_directories()` 调用。

### Build and Run

创建一个名为 `Step3_build` 的新目录，运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建，或者通过在构建目录中运行 `cmake --build .` 进行构建。以下是命令行操作的复习示例：

```shell
mkdir Step3_build
cd Step3_build
cmake ../Step3
cmake --build .
```

接下来，使用新构建的 `Tutorial` 可执行文件，并验证其是否按预期工作。

### Solution

让我们更新上一步的代码，采用现代 CMake 的使用需求方法。

我们需要声明，任何链接到 `MathFunctions` 的人都需要包含当前源目录，而 `MathFunctions` 本身不需要。这可以通过 `INTERFACE` 使用需求来表达。记住，`INTERFACE` 表示消费者需要但生产者不需要的内容。

在 `MathFunctions/CMakeLists.txt` 的末尾，使用带有 `INTERFACE` 关键字的 `target_include_directories()`，如下所示：

```cmake
target_include_directories(MathFunctions
                           INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}
                           )
```

现在我们已经为 `MathFunctions` 指定了使用需求，因此可以安全地从顶级 `CMakeLists.txt` 中移除对 `EXTRA_INCLUDES` 变量的使用。

移除这一行：

```cmake
list(APPEND EXTRA_INCLUDES "${PROJECT_SOURCE_DIR}/MathFunctions")
```

并从 `target_include_directories` 中移除 `EXTRA_INCLUDES`：

```cmake
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```

请注意，通过这种方法，我们的可执行目标要使用库时唯一需要做的就是用库目标的名字调用 `target_link_libraries()`。在较大的项目中，手动指定库依赖关系的经典方法会很快变得非常复杂。

## Exercise 2 - Setting the C++ Standard with Interface Libraries

现在我们已经将代码切换到更现代的方法，接下来让我们演示一种为多个目标设置属性的现代技术。

我们将重构现有代码以使用一个 `INTERFACE` 库。在下一步中，我们会用这个库来展示 `generator expressions` 的一种常见用途。

### Goal

添加一个 `INTERFACE` 库目标，以指定所需的 C++ 标准。

### Helpful Resources

- add_library()
- target_compile_features()
- target_link_libraries()

### Files to Edit

- CMakeLists.txt
- MathFunctions/CMakeLists.txt

### Getting Started

在本练习中，我们将重构代码，使用一个 `INTERFACE` 库来指定 C++ 标准。

从 Step3 练习 1 的结束部分开始本练习。你需要完成 `TODO 4` 到 `TODO 7`。

首先，编辑顶级 `CMakeLists.txt` 文件。创建一个名为 `tutorial_compiler_flags` 的 `INTERFACE` 库目标，并指定 `cxx_std_11` 作为目标编译器特性。

修改 `CMakeLists.txt` 和 `MathFunctions/CMakeLists.txt`，使得所有目标都包含对 `tutorial_compiler_flags` 的 `target_link_libraries()` 调用。

### Build and Run

由于我们已经在练习 1 中配置好了构建目录，只需通过调用以下命令重新构建我们的代码：

```shell
cd Step3_build
cmake --build .
```

接下来，使用新构建的 `Tutorial` 可执行文件，并验证其是否按预期工作。

### Solution

让我们更新上一步的代码，使用接口库来设置我们的 C++ 要求。

首先，我们需要移除对变量 `CMAKE_CXX_STANDARD` 和 `CMAKE_CXX_STANDARD_REQUIRED` 的两个 `set()` 调用。需要移除的具体行如下：

```cmake
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

接下来，我们需要创建一个接口库 `tutorial_compiler_flags`。然后使用 `target_compile_features()` 添加编译器特性 `cxx_std_11`。

```cmake
add_library(tutorial_compiler_flags INTERFACE)
target_compile_features(tutorial_compiler_flags INTERFACE cxx_std_11)
```

最后，在我们的接口库设置完成后，我们需要将可执行文件 `Tutorial`、`SqrtLibrary` 库以及 `MathFunctions` 库链接到新的 `tutorial_compiler_flags` 库。相应的代码如下所示：

```cmake
target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
```

这样：

```cmake
  target_link_libraries(SqrtLibrary PUBLIC tutorial_compiler_flags)
```

和这样：

```cmake
target_link_libraries(MathFunctions PUBLIC tutorial_compiler_flags)
```

通过这种方式，我们的所有代码仍然需要 C++11 才能构建。但请注意，使用这种方法，我们可以明确指定哪些目标需要满足特定的要求。此外，我们在接口库中创建了一个单一的真实来源。
