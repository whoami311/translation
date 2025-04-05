# Step 8: Adding a Custom Command and Generated File

假设为了本教程的目的，我们决定永远不使用平台的 `log` 和 `exp` 函数，而是希望生成一个预计算值的表格，并在 `mysqrt` 函数中使用。在本节中，我们将作为构建过程的一部分创建该表格，然后将其编译到我们的应用程序中。

首先，让我们从 `MathFunctions/CMakeLists.txt` 中移除对 `log` 和 `exp` 函数的检查。然后从 `mysqrt.cxx` 中移除对 `HAVE_LOG` 和 `HAVE_EXP` 的检查。同时，我们可以移除 `#include <cmath>`。

在 `MathFunctions` 子目录中，提供了一个名为 `MakeTable.cxx` 的新源文件来生成表格。

在查看文件后，我们可以看到该表格作为有效的 C++ 代码生成，并且输出文件名作为参数传入。

下一步是创建 `MathFunctions/MakeTable.cmake`。然后，向文件中添加适当的命令来构建 `MakeTable` 可执行文件，并将其作为构建过程的一部分运行。这需要几个命令来完成。

首先，我们为 `MakeTable` 添加一个可执行文件。

```cmake
add_executable(MakeTable MakeTable.cxx)
```

在创建可执行文件后，我们使用 `target_link_libraries()` 将 `tutorial_compiler_flags` 添加到我们的可执行文件中。

```cmake
target_link_libraries(MakeTable PRIVATE tutorial_compiler_flags)
```

然后，我们添加一个自定义命令，指定通过运行 MakeTable 来生成 `Table.h` 的方法。

```cmake
add_custom_command(
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  COMMAND MakeTable ${CMAKE_CURRENT_BINARY_DIR}/Table.h
  DEPENDS MakeTable
  )
```

接下来，我们必须让 CMake 知道 `mysqrt.cxx` 依赖于生成的文件 `Table.h`。这是通过将生成的 `Table.h` 添加到库 `SqrtLibrary` 的源文件列表中来实现的。

```cmake
  add_library(SqrtLibrary STATIC
              mysqrt.cxx
              ${CMAKE_CURRENT_BINARY_DIR}/Table.h
              )
```

我们还需要将当前的二进制目录添加到包含目录列表中，以便 `Table.h` 能够被 `mysqrt.cxx` 找到并包含。

```cmake
  target_include_directories(SqrtLibrary PRIVATE
                             ${CMAKE_CURRENT_BINARY_DIR}
                             )

  # link SqrtLibrary to tutorial_compiler_flags
```

作为最后一步，我们需要在 `MathFunctions/CMakeLists.txt` 的顶部包含 `MakeTable.cmake`。

```cmake
  include(MakeTable.cmake)
```

现在让我们使用生成的表格。首先，修改 `mysqrt.cxx` 以包含 `Table.h`。接下来，我们可以重写 `mysqrt` 函数以使用该表格：

```cpp
double mysqrt(double x)
{
  if (x <= 0) {
    return 0;
  }

  // use the table to help find an initial value
  double result = x;
  if (x >= 1 && x < 10) {
    std::cout << "Use the table to help find an initial value " << std::endl;
    result = sqrtTable[static_cast<int>(x)];
  }

  // do ten iterations
  for (int i = 0; i < 10; ++i) {
    if (result <= 0) {
      result = 0.1;
    }
    double delta = x - (result * result);
    result = result + 0.5 * delta / result;
    std::cout << "Computing sqrt of " << x << " to be " << result << std::endl;
  }

  return result;
}
}
}
```

运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建。

当构建此项目时，它会首先构建 `MakeTable` 可执行文件。然后，它将运行 `MakeTable` 以生成 `Table.h`。最后，它将编译包含 `Table.h` 的 `mysqrt.cxx` 以生成 `MathFunctions` 库。

运行 `Tutorial` 可执行文件并验证它是否正在使用该表格。
