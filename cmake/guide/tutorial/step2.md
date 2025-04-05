# Step 2: Adding a Library

至此，我们已经了解了如何使用 CMake 创建一个基本项目。在本步骤中，我们将学习如何在项目中创建和使用库。我们还将看到如何使库的使用成为可选的。

## Exercise 1 - Creating a Library

要在 CMake 中添加库，使用 `add_library()` 命令并指定应由哪些源文件组成该库。

我们可以将所有源文件放在一个目录下，但也可以通过一个或多个子目录来组织项目。在这种情况下，我们将创建一个专门用于我们库的子目录。在这里，我们可以添加一个新的 `CMakeLists.txt` 文件和一个或多个源文件。在顶级 `CMakeLists.txt` 文件中，我们将使用 `add_subdirectory()` 命令将子目录添加到构建中。

一旦创建了库，就可以使用 `target_include_directories()` 和 `target_link_libraries()` 将其连接到我们的可执行目标。

### Goal

添加并使用一个库。

### Helpful Resources

- add_library()
- add_subdirectory()
- target_include_directories()
- target_link_libraries()
- PROJECT_SOURCE_DIR

### Files to Edit

- CMakeLists.txt
- tutorial.cxx
- MathFunctions/CMakeLists.txt

### Getting Started

在本练习中，我们将向项目中添加一个库，该库包含我们自己实现的计算数字平方根的功能。然后，可执行文件可以使用这个库而不是编译器提供的标准平方根函数。

对于本教程，我们将把库放在一个名为 `MathFunctions` 的子目录中。这个目录已经包含了头文件 `MathFunctions.h` 和 `mysqrt.h`。相应的源文件 `MathFunctions.cxx` 和 `mysqrt.cxx` 也已提供。我们不需要修改这些文件。`mysqrt.cxx` 中有一个名为 `mysqrt` 的函数，它提供了与编译器的 `sqrt` 函数类似的功能。`MathFunctions.cxx` 包含一个 `sqrt` 函数，用于隐藏 `sqrt` 实现的细节。

从 `Help/guide/tutorial/Step2` 目录开始，从 `TODO 1` 开始并完成到 `TODO 6`。

首先，填写 `MathFunctions` 子目录中的一行 `CMakeLists.txt` 文件。

接下来，编辑顶级 `CMakeLists.txt` 文件。

最后，在 `tutorial.cxx` 中使用新创建的 `MathFunctions` 库。

### Build and Run

运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建。

以下是从命令行执行此操作的复习示例：

```shell
mkdir Step2_build
cd Step2_build
cmake ../Step2
cmake --build .
```

尝试使用新构建的 `Tutorial`，并确保它仍然能够生成准确的平方根值。

### Solution

在 `MathFunctions` 目录中的 `CMakeLists.txt` 文件里，我们使用 `add_library()` 创建了一个名为 `MathFunctions` 的库目标。库的源文件作为参数传递给 `add_library()`。其代码如下所示：

```cmake
add_library(MathFunctions MathFunctions.cxx mysqrt.cxx)
```

为了使用这个新库，我们将在顶级 `CMakeLists.txt` 文件中添加一个 `add_subdirectory()` 调用，以确保该库能够被构建。

```cmake
add_subdirectory(MathFunctions)
```

接下来，使用 `target_link_libraries()` 将新库目标链接到可执行目标。

```cmake
target_link_libraries(Tutorial PUBLIC MathFunctions)
```

最后，我们需要指定库的头文件位置。修改现有的 `target_include_directories()` 调用，将 `MathFunctions` 子目录添加为包含目录，以便能够找到 `MathFunctions.h` 头文件。

```cmake
target_include_directories(Tutorial PUBLIC
                          "${PROJECT_BINARY_DIR}"
                          "${PROJECT_SOURCE_DIR}/MathFunctions"
                          )
```

现在让我们来使用我们的库。在 `tutorial.cxx` 中，包含 `MathFunctions.h`：

```cpp
#include "MathFunctions.h"
```

最后，将 `sqrt` 替换为包装函数 `mathfunctions::sqrt`。

```c++
  double const outputValue = mathfunctions::sqrt(inputValue);
```

## Exercise 2 - Adding an Option

现在让我们在 MathFunctions 库中添加一个选项，允许开发者选择自定义的平方根实现或内置的标准实现。虽然对于本教程来说并没有实际需要这样做，但在较大的项目中这种情况很常见。

CMake 可以使用 `option()` 命令来实现这一点。这为用户提供了一个变量，他们可以在配置 CMake 构建时更改这个变量。此设置将存储在缓存中，因此用户不需要每次在构建目录上运行 CMake 时都设置该值。

### Goal

添加一个选项，允许在不使用 `MathFunctions` 的情况下进行构建。

### Helpful Resources

- if()
- option()
- target_compile_definitions()

### Files to Edit

- MathFunctions/CMakeLists.txt
- MathFunctions/MathFunctions.cxx

### Getting Started

从练习 1 的结果文件开始。完成 `TODO 7` 到 `TODO 14`。

首先，在 `MathFunctions/CMakeLists.txt` 中使用 `option()` 命令创建一个变量 `USE_MYMATH`。在同一文件中，使用该选项为 `MathFunctions` 库传递一个编译定义。

然后，更新 `MathFunctions.cxx`，根据 `USE_MYMATH` 选项重定向编译。

最后，在 `MathFunctions/CMakeLists.txt` 文件的 `USE_MYMATH` 块内，通过将 `mysqrt.cxx` 作为一个独立库来防止在 `USE_MYMATH` 开启时对其进行编译。

### Build and Run

由于我们在练习 1 中已经配置好了构建目录，因此只需调用以下命令即可重新构建：

```shell
cd ../Step2_build
cmake --build .
```

接下来，运行 `Tutorial` 可执行文件对一些数字进行测试，以验证其仍然正确。

现在让我们将 `USE_MYMATH` 的值更新为 `OFF`。最简单的方法是使用 `cmake-gui` 或在终端中使用 `ccmake`。或者，如果你想从命令行更改选项，可以尝试：

```shell
cmake ../Step2 -DUSE_MYMATH=OFF
```

现在，使用以下命令重新构建代码：

```shell
cmake --build .
```

然后，再次运行可执行文件，以确保在 `USE_MYMATH` 设置为 `OFF` 时它仍然可以正常工作。哪个函数提供了更好的结果，是 `sqrt` 还是 `mysqrt`？

### Solution

第一步是在 `MathFunctions/CMakeLists.txt` 中添加一个选项。此选项将显示在 `cmake-gui` 和 `ccmake` 中，默认值为 `ON`，用户可以对其进行更改。

```cmake
option(USE_MYMATH "Use tutorial provided math implementation" ON)
```

接下来，使用这个新选项使我们的库和 `mysqrt` 函数的构建与链接变得有条件。

创建一个 `if()` 语句来检查 `USE_MYMATH` 的值。在 `if()` 块中，放置 `target_compile_definitions()` 命令，并添加编译定义 `USE_MYMATH`。

```cmake
if (USE_MYMATH)
  target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
endif()
```

当 `USE_MYMATH` 为 `ON` 时，将设置编译定义 `USE_MYMATH`。然后我们可以使用这个编译定义来启用或禁用源代码中的某些部分。

对源代码进行相应的修改非常直接。在 `MathFunctions.cxx` 中，我们让 `USE_MYMATH` 控制使用哪个平方根函数：

```cpp
#ifdef USE_MYMATH
  return detail::mysqrt(x);
#else
  return std::sqrt(x);
#endif
```

接下来，如果定义了 `USE_MYMATH`，我们需要包含 `mysqrt.h`。

```cpp
#ifdef USE_MYMATH
#  include "mysqrt.h"
#endif
```

最后，由于我们现在使用了 `std::sqrt`，因此需要包含 `<cmath>`。

```cpp
#include <cmath>
```

此时，如果 `USE_MYMATH` 为 `OFF`，`mysqrt.cxx` 将不会被使用，但由于 `MathFunctions` 目标在其源文件列表中包含了 `mysqrt.cxx`，它仍然会被编译。

有几种方法可以解决这个问题。第一个选项是在 `USE_MYMATH` 块内使用 `target_sources()` 来添加 `mysqrt.cxx`。另一个选项是在 `USE_MYMATH` 块内创建一个额外的库，专门负责编译 `mysqrt.cxx`。为了本教程的目的，我们将创建一个额外的库。

首先，在 `USE_MYMATH` 块内创建一个名为 `SqrtLibrary` 的库，其源文件为 `mysqrt.cxx`。

```cmake
  add_library(SqrtLibrary STATIC
              mysqrt.cxx
              )

  # TODO 6: Link SqrtLibrary to tutorial_compiler_flags

  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif()
```

接下来，当 `USE_MYMATH` 启用时，我们将 `SqrtLibrary` 链接到 `MathFunctions`。

```cmake
  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
```

最后，我们可以从 `MathFunctions` 库的源文件列表中移除 `mysqrt.cxx`，因为它将在包含 `SqrtLibrary` 时被引入。

```cmake
add_library(MathFunctions MathFunctions.cxx)
```

通过这些更改，`mysqrt` 函数现在对于构建和使用 `MathFunctions` 库的用户来说是完全可选的。用户可以通过切换 `USE_MYMATH` 来控制在构建中使用哪个库。
