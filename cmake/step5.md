# Step 5: Installing and Testing

## Exercise 1 - Install Rules

通常，仅构建可执行文件是不够的，它还应该是可安装的。使用 CMake，我们可以通过 `install()` 命令指定安装规则。在 CMake 中支持本地安装通常只需指定安装位置以及要安装的目标和文件即可。

### Goal

安装 `Tutorial` 可执行文件和 `MathFunctions` 库。

### Helpful Resources

- install()

### Files to Edit

- MathFunctions/CMakeLists.txt
- CMakeLists.txt

### Getting Started

起始代码位于 `Step5` 目录中。在本练习中，完成 `TODO 1` 到 `TODO 4`。

首先，更新 `MathFunctions/CMakeLists.txt`，将 `MathFunctions` 和 `tutorial_compiler_flags` 库安装到 `lib` 目录。在同一文件中，指定将 `MathFunctions.h` 安装到 `include` 目录所需的安装规则。

然后，更新顶级 `CMakeLists.txt`，将 `Tutorial` 可执行文件安装到 `bin` 目录。最后，任何头文件都应安装到 `include` 目录。请记住，`TutorialConfig.h` 位于 `PROJECT_BINARY_DIR` 中。

### Build and Run

创建一个名为 `Step5_build` 的新目录。运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建。

接着，通过命令行使用 `cmake` 命令的 `--install` 选项（在 CMake 3.15 中引入，旧版本的 CMake 必须使用 `make install`）来运行安装步骤。此步骤将会安装适当的头文件、库和可执行文件。例如：

```shell
cmake --install .
```

对于多配置工具，别忘了使用 `--config` 参数来指定配置。

```shell
cmake --install . --config Release
```

如果使用的是 IDE，只需构建 `INSTALL` 目标即可。你也可以通过命令行构建相同的安装目标，如下所示：

```shell
cmake --build . --target install --config Debug
```

CMake 变量 `CMAKE_INSTALL_PREFIX` 用于确定文件安装的根目录。如果使用 `cmake --install` 命令，可以通过 `--prefix` 参数覆盖安装前缀。例如：

```shell
cmake --install . --prefix "/home/myuser/installdir"
```

导航到安装目录，并验证已安装的 `Tutorial` 是否能够运行。

### Solution

我们项目的安装规则非常简单：

- 对于 `MathFunctions`，我们希望将库文件和头文件分别安装到 `lib` 和 `include` 目录。
- 对于 `Tutorial` 可执行文件，我们希望将可执行文件和配置好的头文件分别安装到 `bin` 和 `include` 目录。

因此，在 `MathFunctions/CMakeLists.txt` 的末尾添加以下内容：

```cmake
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()
install(TARGETS ${installable_libs} DESTINATION lib)
```

和

```cmake
install(FILES MathFunctions.h DESTINATION include)
```

`Tutorial` 可执行文件和配置好的头文件的安装规则类似。在顶级 `CMakeLists.txt` 的末尾添加以下内容：

```cmake
install(TARGETS Tutorial DESTINATION bin)
install(FILES "${PROJECT_BINARY_DIR}/TutorialConfig.h"
  DESTINATION include
  )
```

这就是创建教程基本本地安装所需的全部内容。

## Exercise 2 - Testing Support

CTest 提供了一种轻松管理项目测试的方法。可以通过 `add_test()` 命令添加测试。尽管本教程中没有明确介绍，但 CTest 与其他测试框架（如 `GoogleTest`）之间有很多兼容性。

### Goal

使用 `CTest` 为我们的可执行文件创建单元测试。

### Helpful Resources

- enable_testing()
- add_test()
- function()
- set_tests_properties()
- ctest

### Files to Edit

- CMakeLists.txt

### Getting Started

起始源代码位于 `Step5` 目录中。在本练习中，完成 `TODO 5` 到 `TODO 9`。

首先，我们需要启用测试功能。接下来，使用 `add_test()` 开始为我们的项目添加测试。我们将逐步添加 3 个简单的测试，然后你可以根据需要添加更多的测试。

### Build and Run

导航到构建目录并重新构建应用程序。然后，运行 **ctest** 可执行文件：`ctest -N` 和 `ctest -VV`。对于多配置生成器（例如 Visual Studio），必须使用 `-C <mode>` 标志指定配置类型。例如，要在调试模式下运行测试，请从构建目录（而不是 Debug 子目录！）运行 `ctest -C Debug -VV`。发布模式的测试可以从相同位置运行，但需使用 `-C Release`。或者，从 IDE 中构建 `RUN_TESTS` 目标。

### Solution

让我们来测试我们的应用程序。在顶级 `CMakeLists.txt` 文件的末尾，我们首先需要使用 `enable_testing()` 命令启用测试功能。

```cmake
enable_testing()
```

在启用测试功能后，我们将添加一些基础测试，以验证应用程序是否正常工作。首先，我们使用 `add_test()` 创建一个测试，该测试运行 `Tutorial` 可执行文件并传入参数 25。对于此测试，我们不会检查可执行文件计算出的答案。该测试将验证应用程序是否能够运行，不会发生段错误或其他崩溃，并且返回值为零。这是 CTest 测试的基本形式。

```cmake
add_test(NAME Runs COMMAND Tutorial 25)
```

接下来，让我们使用 `PASS_REGULAR_EXPRESSION` 测试属性来验证测试的输出是否包含特定字符串。在这种情况下，验证当提供的参数数量不正确时是否会打印使用说明消息。

```cmake
add_test(NAME Usage COMMAND Tutorial)
set_tests_properties(Usage
  PROPERTIES PASS_REGULAR_EXPRESSION "Usage:.*number"
  )
```

我们接下来要添加的测试将验证计算值是否确实是平方根。

```cmake
add_test(NAME StandardUse COMMAND Tutorial 4)
set_tests_properties(StandardUse
  PROPERTIES PASS_REGULAR_EXPRESSION "4 is 2"
  )
```

仅靠这一个测试还不足以让我们确信它能够对所有输入值都正常工作。我们应该添加更多的测试来验证这一点。为了方便地添加更多测试，我们创建一个名为 `do_test` 的函数，该函数运行应用程序并验证计算出的平方根对于给定输入是否正确。每次调用 `do_test` 时，都会根据传入的参数为项目添加一个新的测试，测试包含名称、输入和预期结果。

```cmake
function(do_test target arg result)
  add_test(NAME Comp${arg} COMMAND ${target} ${arg})
  set_tests_properties(Comp${arg}
    PROPERTIES PASS_REGULAR_EXPRESSION ${result}
    )
endfunction()

# do a bunch of result based tests
do_test(Tutorial 4 "4 is 2")
do_test(Tutorial 9 "9 is 3")
do_test(Tutorial 5 "5 is 2.236")
do_test(Tutorial 7 "7 is 2.645")
do_test(Tutorial 25 "25 is 5")
do_test(Tutorial -25 "-25 is (-nan|nan|0)")
do_test(Tutorial 0.0001 "0.0001 is 0.01")
```
