# Step 12: Packaging Debug and Release

**注意**：此示例适用于单配置生成器，但不适用于多配置生成器（例如 Visual Studio）。

默认情况下，CMake 的模型是构建目录只包含一种配置，无论是 Debug、Release、MinSizeRel 还是 RelWithDebInfo。然而，可以设置 CPack 来捆绑多个构建目录，并构造一个包含同一项目的多种配置的包。

首先，我们需要确保调试和发布版本的库在安装时使用不同的名称。我们使用 `d` 作为调试库的后缀。

在顶级 CMakeLists.txt 文件的开头附近设置 `CMAKE_DEBUG_POSTFIX`：

```cmake
set(CMAKE_DEBUG_POSTFIX d)

add_library(tutorial_compiler_flags INTERFACE)
```

以及在可执行文件 tutorial 上设置 `DEBUG_POSTFIX` 属性：

```cmake
add_executable(Tutorial tutorial.cxx)
set_target_properties(Tutorial PROPERTIES DEBUG_POSTFIX ${CMAKE_DEBUG_POSTFIX})

target_link_libraries(Tutorial PUBLIC MathFunctions tutorial_compiler_flags)
```

让我们为 `MathFunctions` 库添加版本编号。在 `MathFunctions/CMakeLists.txt` 中，设置 `VERSION` 和 `SOVERSION` 属性：

```cmake
set_property(TARGET MathFunctions PROPERTY VERSION "1.0.0")
set_property(TARGET MathFunctions PROPERTY SOVERSION "1")
```

从 `Step12` 目录中，创建 `debug` 和 `release` 子目录。目录结构将如下所示：

```
- Step12
   - debug
   - release
```

现在我们需要设置调试和发布版本的构建。可以使用 `CMAKE_BUILD_TYPE` 来设置配置类型：

```shell
cd debug
cmake -DCMAKE_BUILD_TYPE=Debug ..
cmake --build .
cd ../release
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
```

现在调试和发布版本的构建都已完成，我们可以使用一个自定义配置文件将两个构建打包到一个单独的发布中。在 `Step12` 目录中，创建一个名为 `MultiCPackConfig.cmake` 的文件。在此文件中，首先包含由 `cmake` 可执行文件创建的默认配置文件。

接下来，使用 `CPACK_INSTALL_CMAKE_PROJECTS` 变量来指定要安装的项目。在本例中，我们需要同时安装调试和发布版本。

```cmake
include("release/CPackConfig.cmake")

set(CPACK_INSTALL_CMAKE_PROJECTS
    "debug;Tutorial;ALL;/"
    "release;Tutorial;ALL;/"
    )
```

从 `Step12` 目录中，运行 `cpack`，并通过 `config` 选项指定我们的自定义配置文件：

```shell
cpack --config MultiCPackConfig.cmake
```
