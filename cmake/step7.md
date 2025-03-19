# Step 7: Adding System Introspection

让我们考虑为项目添加一些依赖于目标平台可能不具备的功能的代码。在本例中，我们将添加一些代码，这些代码依赖于目标平台是否具有 `log` 和 `exp` 函数。当然，几乎每个平台都具备这些函数，但在本教程中，我们假设它们并不常见。

## Exercise 1 - Assessing Dependency Availability

### Goal

根据可用的系统依赖项更改实现。

### Helpful Resources

- CheckCXXSourceCompiles
- target_compile_definitions()

### Files to Edit

- MathFunctions/CMakeLists.txt
- MathFunctions/mysqrt.cxx

### Getting Started

起始源代码位于 `Step7` 目录中。在本练习中，完成 `TODO 1` 到 `TODO 5`。

首先，编辑 `MathFunctions/CMakeLists.txt`。包含 `CheckCXXSourceCompiles` 模块。然后，使用 `check_cxx_source_compiles` 来确定是否可以从 `cmath` 中获取 `log` 和 `exp`。如果它们可用，使用 `target_compile_definitions()` 将 `HAVE_LOG` 和 `HAVE_EXP` 指定为编译定义。

在 `MathFunctions/mysqrt.cxx` 中，包含 `cmath`。然后，如果系统具有 `log` 和 `exp`，则使用它们来计算平方根。

### Build and Run

创建一个名为 `Step7_build` 的新目录。运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，然后使用你选择的构建工具进行构建，并运行 `Tutorial` 可执行文件。

这可以如下所示：

```shell
mkdir Step7_build
cd Step7_build
cmake ../Step7
cmake --build .
```

现在哪个函数提供了更好的结果，`sqrt` 还是 `mysqrt`？

### Solution

在本练习中，我们将使用 `CheckCXXSourceCompiles` 模块中的函数，因此首先必须在 `MathFunctions/CMakeLists.txt` 中包含它。

```cmake
  include(CheckCXXSourceCompiles)
```

然后使用 `check_cxx_source_compiles` 检查 `log` 和 `exp` 是否可用。这个函数允许我们在真正编译源代码之前，尝试使用所需的依赖项编译简单的代码。生成的变量 `HAVE_LOG` 和 `HAVE_EXP` 表示这些依赖项是否可用。

```cmake
  check_cxx_source_compiles("
    #include <cmath>
    int main() {
      std::log(1.0);
      return 0;
    }
  " HAVE_LOG)
  check_cxx_source_compiles("
    #include <cmath>
    int main() {
      std::exp(1.0);
      return 0;
    }
  " HAVE_EXP)
```

接下来，我们需要将这些 CMake 变量传递给我们的源代码。这样，我们的源代码可以知道哪些资源是可用的。如果 `log` 和 `exp` 都可用，则使用 `target_compile_definitions()` 将 `HAVE_LOG` 和 `HAVE_EXP` 指定为 `PRIVATE` 编译定义。

```cmake
  if(HAVE_LOG AND HAVE_EXP)
    target_compile_definitions(SqrtLibrary
                               PRIVATE "HAVE_LOG" "HAVE_EXP"
                               )
  endif()

  target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
endif()
```

由于我们可能会使用 `log` 和 `exp`，因此需要修改 `mysqrt.cxx` 以包含 `cmath`。

```cpp
#include <cmath>
```

如果系统中存在 `log` 和 `exp`，则在 `mysqrt` 函数中使用它们来计算平方根。`MathFunctions/mysqrt.cxx` 中的 `mysqrt` 函数将如下所示：

```cpp
#if defined(HAVE_LOG) && defined(HAVE_EXP)
  double result = std::exp(std::log(x) * 0.5);
  std::cout << "Computing sqrt of " << x << " to be " << result
            << " using log and exp" << std::endl;
#else
  double result = x;

  // do ten iterations
  for (int i = 0; i < 10; ++i) {
    if (result <= 0) {
      result = 0.1;
    }
    double delta = x - (result * result);
    result = result + 0.5 * delta / result;
    std::cout << "Computing sqrt of " << x << " to be " << result << std::endl;
  }
#endif
```
