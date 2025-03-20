# Step 9: Packaging an Installer

接下来假设我们希望将项目分发给其他人以便他们可以使用。我们希望在各种平台上提供二进制和源代码分发。这与我们在 “Installing and Testing” 部分中所做的安装略有不同，当时我们是从源代码构建的二进制文件进行安装。在这个例子中，我们将构建支持二进制安装和包管理功能的安装包。为此，我们将使用 CPack 创建特定于平台的安装程序。具体来说，我们需要在顶级 `CMakeLists.txt` 文件的底部添加几行代码。

```cmake
include(InstallRequiredSystemLibraries)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/License.txt")
set(CPACK_PACKAGE_VERSION_MAJOR "${Tutorial_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${Tutorial_VERSION_MINOR}")
set(CPACK_GENERATOR "TGZ")
set(CPACK_SOURCE_GENERATOR "TGZ")
include(CPack)
```

就是这么简单。我们首先包含 `InstallRequiredSystemLibraries`。这个模块会包含当前平台下项目所需的所有运行时库。接下来，我们设置一些 CPack 变量，指向我们存储该项目的许可证和版本信息的位置。版本信息在本教程的前面部分已经设置，许可证文件 `License.txt` 已经包含在本步骤的顶级源目录中。`CPACK_GENERATOR` 和 `CPACK_SOURCE_GENERATOR` 变量分别选择用于二进制和源代码安装的生成器。

最后，我们包含 `CPack` 模块，它将使用这些变量以及当前系统的一些其他属性来设置安装程序。

下一步是以常规方式构建项目，然后运行 `cpack` 可执行文件。要构建一个二进制分发包，请从构建目录运行：

```shell
cpack
```

要指定二进制生成器，请使用 `-G` 选项。对于多配置构建，请使用 `-C` 指定配置。例如：

```shell
cpack -G ZIP -C Debug
```

要查看可用生成器的列表，请参阅 `cpack-generators(7)` 或运行 `cpack --help`。像 ZIP 这样的 `archive generator` 会创建所有*已安装*文件的压缩归档。

要为*完整的*源代码树创建归档，你可以输入：

```shell
cpack --config CPackSourceConfig.cmake
```

或者，运行 `make package`，或者在 IDE 中右键点击 `Package` 目标并选择 `Build Project`。

运行二进制目录中找到的安装程序。然后运行已安装的可执行文件并验证其是否正常工作。
