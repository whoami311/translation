# Step 11: Adding Export Configuration

在 “Installing and Testing” 教程中，我们添加了 CMake 安装项目库和头文件的能力。在 “Packaging an Installer” 部分，我们添加了打包这些信息的功能，以便可以分发给其他人。

下一步是添加必要的信息，以便其他 CMake 项目可以使用我们的项目，无论是从构建目录、本地安装还是从包中获取。

第一步是更新我们的 `install(TARGETS)` 命令，不仅指定一个 `DESTINATION`，还要指定一个 `EXPORT`。`EXPORT` 关键字生成一个包含代码的 CMake 文件，用于从安装树导入 install 命令中列出的所有目标。因此，让我们通过更新 `MathFunctions/CMakeLists.txt` 中的 `install` 命令来显式 `EXPORT` `MathFunctions` 库，使其看起来像这样：

```cmake
set(installable_libs MathFunctions tutorial_compiler_flags)
if(TARGET SqrtLibrary)
  list(APPEND installable_libs SqrtLibrary)
endif()
install(TARGETS ${installable_libs}
        EXPORT MathFunctionsTargets
        DESTINATION lib)
# install include headers
install(FILES MathFunctions.h DESTINATION include)
```

现在我们已经导出了 `MathFunctions`，还需要显式安装生成的 `MathFunctionsTargets.cmake` 文件。这可以通过在顶级 `CMakeLists.txt` 的底部添加以下内容来实现：

```cmake
install(EXPORT MathFunctionsTargets
  FILE MathFunctionsTargets.cmake
  DESTINATION lib/cmake/MathFunctions
)
```

此时，你应该尝试运行 CMake。如果一切设置正确，你会看到 CMake 生成一个类似于以下内容的错误：

```
Target "MathFunctions" INTERFACE_INCLUDE_DIRECTORIES property contains
path:

  "/Users/robert/Documents/CMakeClass/Tutorial/Step11/MathFunctions"

which is prefixed in the source directory.
```

CMake 提示你在生成导出信息时，它将导出一个与当前机器紧密相关的路径，而该路径在其他机器上将无效。解决此问题的方法是更新 `MathFunctions` 的 `target_include_directories()`，使其能够理解在从构建目录以及从安装/包中使用时需要不同的 `INTERFACE` 路径。这意味着需要将 `MathFunctions` 的 `target_include_directories()` 调用修改为如下形式：

```cmake
target_include_directories(MathFunctions
                           INTERFACE
                            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                            $<INSTALL_INTERFACE:include>
                           )
```

完成此更新后，我们可以重新运行 CMake，并验证它不再发出警告。

此时，我们已经让 CMake 正确打包了所需的目标信息，但仍然需要生成一个 `MathFunctionsConfig.cmake` 文件，以便 CMake 的 `find_package()` 命令能够找到我们的项目。因此，让我们继续在项目的顶层添加一个名为 `Config.cmake.in` 的新文件，其内容如下：

```cmake.in

@PACKAGE_INIT@

include ( "${CMAKE_CURRENT_LIST_DIR}/MathFunctionsTargets.cmake" )
```

然后，为了正确配置并安装该文件，在顶级 `CMakeLists.txt` 的底部添加以下内容：

```cmake
install(EXPORT MathFunctionsTargets
  FILE MathFunctionsTargets.cmake
  DESTINATION lib/cmake/MathFunctions
)

include(CMakePackageConfigHelpers)
```

接下来，我们执行 `configure_package_config_file()`。此命令将以与标准 `configure_file()` 方法略有不同的方式配置提供的文件。为了正确使用该函数，输入文件应包含一行文本 `@PACKAGE_INIT@`，以及所需的内容。该变量将被替换为一段代码块，用于将设置的值转换为相对路径。这些新值可以通过相同名称引用，但需加上 `PACKAGE_` 前缀。

```cmake
install(EXPORT MathFunctionsTargets
  FILE MathFunctionsTargets.cmake
  DESTINATION lib/cmake/MathFunctions
)

include(CMakePackageConfigHelpers)
# generate the config file that includes the exports
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake"
  INSTALL_DESTINATION "lib/cmake/MathFunctions"
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
  )
```

接下来是 `write_basic_package_version_file()`。此命令会写入一个文件，该文件被 `find_package()` 使用，用于记录所需包的版本和兼容性。在这里，我们使用 `Tutorial_VERSION_*` 变量，并指定其兼容性为 `AnyNewerVersion`，这表示此版本或任何更高版本都与请求的版本兼容。

```cmake
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfigVersion.cmake"
  VERSION "${Tutorial_VERSION_MAJOR}.${Tutorial_VERSION_MINOR}"
  COMPATIBILITY AnyNewerVersion
)
```

最后，设置两个生成的文件都被安装：

```cmake
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsConfigVersion.cmake
  DESTINATION lib/cmake/MathFunctions
  )
```

至此，我们已经为项目生成了一个可重定位的 CMake 配置，可以在项目安装或打包后使用。如果我们希望项目也能从构建目录中使用，只需在顶级 `CMakeLists.txt` 的底部添加以下内容：

```cmake
export(EXPORT MathFunctionsTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/MathFunctionsTargets.cmake"
)
```

通过此 export 调用，我们现在生成了一个 `MathFunctionsTargets.cmake`，从而使构建目录中的已配置 `MathFunctionsConfig.cmake` 可以被其他项目使用，而无需安装它。
