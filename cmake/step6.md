# Step 6: Adding Support for a Testing Dashboard

为提交我们的测试结果到仪表板添加支持非常简单。我们已经在 `Testing Support` 中为项目定义了一些测试。现在，我们只需运行这些测试并将它们提交到 CDash。

## Exercise 1 - Send Results to a Testing Dashboard

### Goal

使用 CDash 显示我们的 CTest 结果。

### Helpful Resources

- ctest(1)
- include()
- CTest

### Files to Edit

- CMakeLists.txt

### Getting Started

对于本练习，在顶级 `CMakeLists.txt` 中通过包含 `CTest` 模块来完成 `TODO 1`。这将启用使用 CTest 进行测试以及向 CDash 提交仪表板数据，因此我们可以安全地移除对 `enable_testing()` 的调用。

我们还需要获取一个 `CTestConfig.cmake` 文件并将其放置在顶级目录中。当运行 `ctest` 可执行文件时，它会读取此文件以收集有关测试仪表板的信息。该文件包含：

- 项目名称
- 项目的 "Nightly" 开始时间
  - 该项目 24 小时 “一天” 的开始时间
- 提交生成的文档将被发送到的 CDash 实例的 URL

对于本教程，使用的是一个公共仪表板服务器，并且相应的 `CTestConfig.cmake` 文件已在本步骤的根目录中提供。实际上，此文件应从 CDash 实例上的项目 `Settings` 页面下载。一旦从 CDash 下载后，不应在本地修改该文件。

```cmake
set(CTEST_PROJECT_NAME "CMakeTutorial")
set(CTEST_NIGHTLY_START_TIME "00:00:00 EST")

set(CTEST_DROP_METHOD "http")
set(CTEST_DROP_SITE "my.cdash.org")
set(CTEST_DROP_LOCATION "/submit.php?project=CMakeTutorial")
set(CTEST_DROP_SITE_CDASH TRUE)
```

### Build and Run

请注意，作为向 CDash 提交的一部分，有关你的开发系统的一些信息（例如站点名称或完整路径名）可能会被公开显示。

为了创建一个简单的测试仪表板，运行 `cmake` 可执行文件或 `cmake-gui` 来配置项目，但暂时不要构建它。相反，导航到构建目录并运行：

```shell
ctest [-VV] -D Experimental
```

请记住，对于多配置生成器（例如 Visual Studio），必须指定配置类型：

```shell
ctest [-VV] -C Debug -D Experimental
```

或者，从 IDE 中构建 `Experimental` 目标。

`ctest` 可执行文件将会构建项目、运行所有测试，并将结果提交到 Kitware 的公共仪表板：[https://my.cdash.org/index.php?project=CMakeTutorial](https://my.cdash.org/index.php?project=CMakeTutorial)。

### Solution

此步骤中唯一需要更改的 CMake 代码是通过在顶级 `CMakeLists.txt` 中包含 `CTest` 模块，以启用向 CDash 提交仪表板数据：

```cmake
include(CTest)
```
