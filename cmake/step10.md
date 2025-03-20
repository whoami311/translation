# Step 10: Selecting Static or Shared Libraries

在本节中，我们将展示如何使用 `BUILD_SHARED_LIBS` 变量来控制 `add_library()` 的默认行为，并允许对未明确指定类型（`STATIC`、`SHARED`、`MODULE` 或 `OBJECT`）的库的构建方式进行控制。

为了实现这一点，我们需要将 `BUILD_SHARED_LIBS` 添加到顶级 `CMakeLists.txt` 中。我们使用 `option()` 命令，因为它允许用户选择该值是 `ON` 还是 `OFF`。

```cmake
option(BUILD_SHARED_LIBS "Build using shared libraries" ON)
```

接下来，我们需要为静态库和共享库指定输出目录。

```cmake
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

option(BUILD_SHARED_LIBS "Build using shared libraries" ON)
```

最后，更新 `MathFunctions/MathFunctions.h` 以使用 DLL 导出定义：

```cpp
#if defined(_WIN32)
#  if defined(EXPORTING_MYMATH)
#    define DECLSPEC __declspec(dllexport)
#  else
#    define DECLSPEC __declspec(dllimport)
#  endif
#else // non windows
#  define DECLSPEC
#endif

namespace mathfunctions {
double DECLSPEC sqrt(double x);
}
```

此时，如果你构建所有内容，你可能会注意到链接失败了，因为我们将没有位置无关代码的静态库与有位置无关代码的库结合在一起。解决这个问题的方法是，在构建共享库时，显式地将 SqrtLibrary 的 `POSITION_INDEPENDENT_CODE` 目标属性设置为 `True`。

```cmake
  # state that SqrtLibrary need PIC when the default is shared libraries
  set_target_properties(SqrtLibrary PROPERTIES
                        POSITION_INDEPENDENT_CODE ${BUILD_SHARED_LIBS}
                        )
```

定义 `EXPORTING_MYMATH`，表明在 Windows 上构建时我们使用 `__declspec(dllexport)`。

```cmake
# define the symbol stating we are using the declspec(dllexport) when
# building on windows
target_compile_definitions(MathFunctions PRIVATE "EXPORTING_MYMATH")
```

**练习**：我们修改了 `MathFunctions.h` 以使用 dll 导出定义。查阅 CMake 文档，你能否找到一个辅助模块来简化这一过程？
