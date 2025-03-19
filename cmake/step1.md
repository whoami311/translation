# Step 1: A Basic Starting Point

我应该从哪里开始使用 CMake？本步骤将介绍 CMake 的一些基本语法、命令和变量。在引入这些概念时，我们将完成三个练习，并创建一个简单的 CMake 项目。

本步骤中的每个练习都将从一些背景信息开始。然后，提供目标和有用的资源列表。在 `Files to Edit` （需要编辑的文件）部分中的每个文件都位于 `Step1` 目录中，并包含一个或多个 `TODO` 注释。每个 `TODO` 代表一行或两行需要更改或添加的代码。`TODO` 是按数字顺序设计的，首先完成 `TODO 1`，然后是 `TODO 2`，依此类推。`Getting Started`（入门指南）部分会给出一些有用的提示并引导你完成练习。接着，`Build and Run`（构建与运行）部分将逐步指导你如何构建和测试练习。最后，在每个练习结束时会讨论预期的解决方案。

另请注意，教程中的每一步都是建立在前一步的基础上的。因此，例如，`Step2` 的起始代码就是 `Step1` 的完整解决方案。

## Exercise 1 - Building a Basic Project

最基本的 CMake 项目是使用单个源代码文件构建的可执行文件。对于这样简单的项目，只需要一个包含三个命令的 `CMakeLists.txt` 文件。

注意：尽管 CMake 支持大小写混合的命令，但建议使用小写命令，并且在整个教程中将使用小写命令。

任何项目的顶级 `CMakeLists.txt` 文件必须首先使用 `cmake_minimum_required()` 命令指定一个最低的 CMake 版本。这会建立策略设置，并确保以下 CMake 函数在兼容版本的 CMake 中运行。

要开始一个项目，我们使用 `project()` 命令来设置项目名称。每个项目都需要调用此命令，并且应在 `cmake_minimum_required()` 之后尽快调用。正如我们将在后面看到的，此命令还可以用于指定其他项目级别的信息，例如语言或版本号。

最后，`add_executable()`命令告诉 CMake 使用指定的源代码文件创建一个可执行文件。

### Goal

了解如何创建一个简单的CMake项目。

### Helpful Resources

- add_executable()
- cmake_minimum_required()
- project()

### Files to Edit

- CMakeLists.txt

### Getting Started

`tutorial.cxx` 的源代码位于 `Help/guide/tutorial/Step1` 目录中，可用于计算一个数字的平方根。在此步骤中，此文件无需编辑。

同一目录下还有一个 `CMakeLists.txt` 文件，你需要完成它。从 `TODO 1` 开始，并依次完成到 `TODO 3`。

### Build and Run

一旦完成了 `TODO 1` 到 `TODO 3`，我们就可以准备构建并运行我们的项目了！首先，运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建。

例如，从命令行我们可以导航到 CMake 源代码树中的 `Help/guide/tutorial` 目录，并创建一个构建目录：

```shell
mkdir Step1_build
```

接下来，导航到该构建目录并运行 `cmake` 来配置项目并生成本地构建系统：

```shell
cd Step1_build
cmake ../Step1
```

然后调用该构建系统来实际编译/链接项目：

```shell
cmake --build .
```

对于多配置生成器（例如 Visual Studio），首先导航到相应的子目录，例如：

```shell
cd Debug
```

最后，尝试使用新构建的 `Tutorial`：

```shell
Tutorial 4294967296
Tutorial 10
Tutorial
```

**注意**：根据 shell 的不同，正确的语法可能是 `Tutorial`、`./Tutorial` 或 `.\Tutorial`。为简单起见，练习中将始终使用 `Tutorial`。

### Solution

如上所述，一个三行的 `CMakeLists.txt` 文件就足以让我们开始运行。第一行是使用 `cmake_minimum_required()` 来设置 CMake 版本，如下所示：

```cmake
cmake_minimum_required(VERSION 3.10)
```

创建一个基本项目的下一步是使用 `project()` 命令来设置项目名称，如下所示：

```cmake
project(Tutorial)
```

对于一个基本项目，要调用的最后一个命令是 `add_executable()`。我们按如下方式调用它：

```cmake
add_executable(Tutorial tutorial.cxx)
```

## Exercise 2 - Specifying the C++ Standard

CMake 有一些特殊的变量，这些变量要么是在幕后创建的，要么在项目代码设置时对 CMake 有特定含义。许多这样的变量以 `CMAKE_` 开头。在为你的项目创建变量时，避免使用这种命名约定。其中两个可由用户设置的特殊变量是 `CMAKE_CXX_STANDARD` 和 `CMAKE_CXX_STANDARD_REQUIRED`。这两个变量可以一起使用，以指定构建项目所需的 C++ 标准。

### Goal

添加一个需要 C++11 的功能。

### Helpful Resources

- CMAKE_CXX_STANDARD
- CMAKE_CXX_STANDARD_REQUIRED
- set()

### Files to Edit

- CMakeLists.txt
- tutorial.cxx

### Getting Started

继续编辑 `Step1` 目录中的文件。从 `TODO 4` 开始，并完成到 `TODO 6`。

首先，通过添加一个需要 C++11 的功能来编辑 `tutorial.cxx`。然后更新 `CMakeLists.txt` 以要求使用 C++11。

### Build and Run

让我们再次构建我们的项目。由于我们已经为练习 1 创建了构建目录并运行了 CMake，因此我们可以直接跳到构建步骤：

```cmake
cd Step1_build
cmake --build .
```

现在我们可以尝试使用新构建的 `Tutorial`，使用的命令与之前相同：

```cmake
Tutorial 4294967296
Tutorial 10
Tutorial
```

### Solution

我们首先通过在 `tutorial.cxx` 中将 `atof` 替换为 `std::stod`，为我们的项目添加一些 C++11 特性。其代码如下所示：

```cpp
  double const inputValue = std::stod(argv[1]);
```

要完成 `TODO 5`，只需移除 `#include <cstdlib>`。

我们需要在 CMake 代码中明确声明应使用正确的编译标志。在 CMake 中启用特定 C++ 标准支持的一种方法是使用 `CMAKE_CXX_STANDARD` 变量。在本教程中，将 `CMakeLists.txt` 文件中的 `CMAKE_CXX_STANDARD` 变量设置为 `11`，并将 `CMAKE_CXX_STANDARD_REQUIRED` 设置为 `True`。确保在调用 `add_executable()` 之前添加 `CMAKE_CXX_STANDARD` 的声明。

```cmake
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)
```

## Exercise 3 - Adding a Version Number and Configured Header File

有时，在 `CMakeLists.txt` 文件中定义的变量在源代码中也能使用可能会很有用。在这种情况下，我们希望打印项目的版本信息。

实现这一目标的一种方法是使用配置头文件（configured header file）。我们创建一个输入文件，并在其中定义一个或多个需要替换的变量。这些变量采用特殊的语法，例如 `@VAR@`。然后，我们使用 `configure_file()` 命令将输入文件复制到指定的输出文件中，并用 `CMakeLists.txt` 文件中 `VAR` 的当前值替换这些变量。

虽然我们可以直接在源代码中编辑版本信息，但使用此功能更为推荐，因为它创建了单一的真实来源，避免了重复。

### Goal

定义并报告项目的版本号。

### Helpful Resources

- \<PROJECT-NAME>_VERSION_MAJOR
- \<PROJECT-NAME>_VERSION_MINOR
- configure_file()
- target_include_directories()

### Files to Edit

- CMakeLists.txt
- tutorial.cxx
- TutorialConfig.h.in

### Getting Started

继续编辑 `Step1` 中的文件。从 `TODO 7` 开始，并完成到 `TODO 12`。在本练习中，我们首先在 `CMakeLists.txt` 中添加项目的版本号。在同一文件中，使用 `configure_file()` 将给定的输入文件复制到输出文件，并替换输入文件内容中的某些变量值。

接下来，创建一个输入头文件 `TutorialConfig.h.in`，定义版本号，它将接受从 `configure_file()` 传递的变量。

最后，更新 `tutorial.cxx` 以打印其版本号。

### Build and Run

让我们再次构建我们的项目。和之前一样，我们已经创建了构建目录并运行了 CMake，因此我们可以直接跳到构建步骤：

```cmake
cd Step1_build
cmake --build .
```

验证在运行可执行文件且不带任何参数时，现在是否报告了版本号。

### Solution

在本练习中，我们通过打印版本号来改进我们的可执行文件。虽然我们完全可以在源代码中实现这一点，但使用 `CMakeLists.txt` 可以让我们为版本号维护单一的数据来源。

首先，我们修改 `CMakeLists.txt` 文件，使用 `project()` 命令来设置项目名称和版本号。当调用 `project()` 命令时，CMake 会在幕后定义 `Tutorial_VERSION_MAJOR` 和 `Tutorial_VERSION_MINOR`。

```cmake
project(Tutorial VERSION 1.0)
```

然后我们使用 `configure_file()` 来复制输入文件，并替换其中指定的 CMake 变量：

```cmake
configure_file(TutorialConfig.h.in TutorialConfig.h)
```

由于配置生成的文件会被写入项目的二进制目录，因此我们必须将该目录添加到包含文件搜索路径的列表中。

**注意**：在本教程中，我们会交替使用“项目构建目录”和“项目二进制目录”这两个术语。它们是相同的，并不特指某个 `bin/` 目录。

我们使用了 `target_include_directories()` 来指定可执行目标应查找头文件的位置。

```cmake
target_include_directories(Tutorial PUBLIC
                           "${PROJECT_BINARY_DIR}"
                           )
```

`TutorialConfig.h.in` 是需要配置的输入头文件。当 `CMakeLists.txt` 中调用 `configure_file()` 时，`@Tutorial_VERSION_MAJOR@` 和 `@Tutorial_VERSION_MINOR@` 的值将被项目中对应的版本号替换，生成 `TutorialConfig.h`。

```cmake
// the configured options and settings for Tutorial
#define Tutorial_VERSION_MAJOR @Tutorial_VERSION_MAJOR@
#define Tutorial_VERSION_MINOR @Tutorial_VERSION_MINOR@
```

接下来，我们需要修改 `tutorial.cxx`，以包含配置生成的头文件 `TutorialConfig.h`。

```cpp
#include "TutorialConfig.h"
```

最后，我们通过如下方式更新 `tutorial.cxx`，打印出可执行文件的名称和版本号：

```c++
  if (argc < 2) {
    // report version
    std::cout << argv[0] << " Version " << Tutorial_VERSION_MAJOR << "."
              << Tutorial_VERSION_MINOR << std::endl;
    std::cout << "Usage: " << argv[0] << " number" << std::endl;
    return 1;
  }
```
