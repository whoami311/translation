# CMake integration

因为我们使用 CMake 来构建 Catch2，所以我们为用户提供了几个集成点。

1. Catch2 导出了一个（带命名空间的）CMake 目标。
2. Catch2 的代码仓库包含用于在 CTest 中自动注册 `TEST_CASE` 的 CMake 脚本。

## CMake targets

Catch2 的 CMake 构建导出了两个目标：`Catch2::Catch2` 和 `Catch2::Catch2WithMain`。如果你不需要自定义的 `main` 函数，你应该使用后者（且仅使用后者）。链接到它会添加适当的包含路径，并将你的目标与实现 Catch2 及其主函数的两个静态库链接在一起。如果你需要自定义 `main` 函数，你应该只链接到 `Catch2::Catch2`。

这意味着，如果 Catch2 已经安装在系统上，你应该只需要执行以下操作：

```cmake
find_package(Catch2 3 REQUIRED)
# These tests can use the Catch2-provided main
add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)

# These tests need their own main
add_executable(custom-main-tests test.cpp test-main.cpp)
target_link_libraries(custom-main-tests PRIVATE Catch2::Catch2)
```

当 Catch2 被用作子目录时，这些目标同样会被提供。假设 Catch2 已被克隆到 `lib/Catch2`，你只需要将 `find_package` 调用替换为 `add_subdirectory(lib/Catch2)`，上面的代码片段仍然可以正常工作。

另一种可能性是使用 [FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html)：

```cmake
Include(FetchContent)

FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        v3.4.0 # or a later release
)

FetchContent_MakeAvailable(Catch2)

add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)
```

## Automatic test registration

Catch2 的代码仓库还包含三个有助于用户自动将他们的 `TEST_CASE` 注册到 CTest 的 CMake 脚本。这些脚本可以在 `extras` 文件夹中找到，分别是：

- `Catch.cmake`（及其依赖项 `CatchAddTests.cmake`）
- `ParseAndAddCatchTests.cmake`（已弃用）
- `CatchShardTests.cmake`（及其依赖项 `CatchShardTestsImpl.cmake`）

如果 Catch2 已经安装在系统中，那么在执行 `find_package(Catch2 REQUIRED)` 后，这些脚本都可以使用。否则，你需要将它们添加到你的 CMake 模块路径中。

### `Catch.cmake` and `CatchAddTests.cmake`

`Catch.cmake` 提供了函数 `catch_discover_tests`，用于从目标中获取测试。该函数通过使用 `--list-test` 标志运行生成的可执行文件，并解析输出以查找所有现有的测试。

#### 用法

```cmake
cmake_minimum_required(VERSION 3.16)

project(baz LANGUAGES CXX VERSION 0.0.1)

find_package(Catch2 REQUIRED)
add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2)

include(CTest)
include(Catch)
catch_discover_tests(tests)
```

当使用 `FetchContent` 时，除非显式更新 `CMAKE_MODULE_PATH` 以包含 `extras` 目录，否则 `include(Catch)` 将失败。

```cmake
# ... FetchContent ...
#
list(APPEND CMAKE_MODULE_PATH ${catch2_SOURCE_DIR}/extras)
include(CTest)
include(Catch)
catch_discover_tests(tests)
```

#### 定制

`catch_discover_tests` 可以接受多个额外参数：

```cmake
catch_discover_tests(target
                     [TEST_SPEC arg1...]
                     [EXTRA_ARGS arg1...]
                     [WORKING_DIRECTORY dir]
                     [TEST_PREFIX prefix]
                     [TEST_SUFFIX suffix]
                     [PROPERTIES name1 value1...]
                     [TEST_LIST var]
                     [REPORTER reporter]
                     [OUTPUT_DIR dir]
                     [OUTPUT_PREFIX prefix]
                     [OUTPUT_SUFFIX suffix]
                     [DISCOVERY_MODE <POST_BUILD|PRE_TEST>]
                     [SKIP_IS_FAILURE]
                     [ADD_TAGS_AS_LABELS]
)
```

- `TEST_SPEC arg1...`

指定要传递给 Catch 可执行文件的测试用例、通配符测试用例、标签和标签表达式，与 `--list-test-names-only` 标志一起使用。

- `EXTRA_ARGS arg1...`  

要在命令行中传递给每个测试用例的任何额外参数。

- `WORKING_DIRECTORY dir`  

指定运行发现的测试用例的目录。如果未提供此选项，则使用当前二进制目录。

- `TEST_PREFIX prefix`  

指定要添加到每个发现的测试用例名称的前缀。当同一个测试可执行文件在多次调用 `catch_discover_tests()` 时使用不同的 `TEST_SPEC` 或 `EXTRA_ARGS`，这会很有用。

- `TEST_SUFFIX suffix`  

与 `TEST_PREFIX` 类似，但指定的是测试名称的后缀。可以同时指定 `TEST_PREFIX` 和 `TEST_SUFFIX`。

- `PROPERTIES name1 value1...`  

指定要在此调用 `catch_discover_tests` 发现的所有测试上设置的附加属性。

- `TEST_LIST var`  

将测试列表存储在变量 `var` 中，而不是默认的 `<target>_TESTS`。当同一个测试可执行文件在多次调用 `catch_discover_tests()` 时使用，这会很有用。注意，此变量仅在 CTest 中可用。

- `REPORTER reporter`  

在运行测试用例时使用指定的报告器。该报告器将作为 `--reporter reporter` 传递给测试运行器。

- `OUTPUT_DIR dir`  

如果指定，参数将作为 `--out dir/<test_name>` 传递给测试可执行文件。实际文件名与测试名称相同。在使用并行测试执行时，应使用此参数代替 `EXTRA_ARGS --out foo`，以避免写入结果输出时的竞争条件。

- `OUTPUT_PREFIX prefix`  

可与 `OUTPUT_DIR` 一起使用。如果指定，`prefix` 将添加到每个输出文件名前，例如 `--out dir/prefix<test_name>`。

- `OUTPUT_SUFFIX suffix`  

可与 `OUTPUT_DIR` 一起使用。如果指定，`suffix` 将添加到每个输出文件名后，例如 `--out dir/<test_name>suffix`。这可用于为输出文件名添加文件扩展名，例如 `.xml`。

- `DISCOVERY_MODE mode`  

如果指定，允许控制何时执行测试发现。值为 `POST_BUILD`（默认值）时，测试发现在构建时执行；值为 `PRE_TEST` 时，测试发现会延迟到测试执行之前（例如，在交叉编译环境中很有用）。如果调用 `catch_discover_tests` 时未传递此参数，`DISCOVERY_MODE` 默认为变量 `CMAKE_CATCH_DISCOVER_TESTS_DISCOVERY_MODE` 的值。这提供了一种机制，用于全局选择首选的测试发现行为。

- `SKIP_IS_FAILURE`  

跳过的测试将被标记为失败。

- `ADD_TAGS_AS_LABELS`  

将测试中的标签作为标签添加到 CTest。

### `ParseAndAddCatchTests.cmake`

⚠此脚本在 Catch2 2.13.4 版本中已被弃用，并由上述使用 `catch_discover_tests` 的方法取代。详情请参阅 [#2092](https://github.com/catchorg/Catch2/issues/2092)。

`ParseAndAddCatchTests` 通过解析与提供的目标关联的所有实现文件，并通过 CTest 的 `add_test` 注册它们来工作。这种方法有一些限制，例如，即使注释掉的测试也会被注册。更严重的是，当前 Catch 中仅有一部分断言宏可以被此脚本检测到，使用无法解析的宏的测试将被*静默忽略*。

#### 用法

```cmake
cmake_minimum_required(VERSION 3.16)

project(baz LANGUAGES CXX VERSION 0.0.1)

find_package(Catch2 REQUIRED)
add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2)

include(CTest)
include(ParseAndAddCatchTests)
ParseAndAddCatchTests(tests)
```

#### 定制

`ParseAndAddCatchTests` 提供了一些自定义选项：

- `PARSE_CATCH_TESTS_VERBOSE` —— 当设置为 `ON` 时，脚本会打印调试信息。默认为 `OFF`。

- `PARSE_CATCH_TESTS_NO_HIDDEN_TESTS` —— 当设置为 `ON` 时，隐藏的测试（使用 `[.]` 或 `[.foo]` 标记的测试）将不会被注册。默认为 `OFF`。

- `PARSE_CATCH_TESTS_ADD_FIXTURE_IN_TEST_NAME` —— 当设置为 `ON` 时，会将夹具类名称添加到 CTest 中的测试名称中。默认为 `ON`。

- `PARSE_CATCH_TESTS_ADD_TARGET_IN_TEST_NAME` —— 当设置为 `ON` 时，会将目标名称添加到 CTest 中的测试名称中。默认为 `ON`。

- `PARSE_CATCH_TESTS_ADD_TO_CONFIGURE_DEPENDS` —— 当设置为 `ON` 时，会将测试文件添加到 `CMAKE_CONFIGURE_DEPENDS` 中。这意味着当测试文件发生变化时，CMake 配置步骤将重新运行，从而使新测试能够自动被发现。默认为 `OFF`。

此外，可以通过在调用 `ParseAndAddCatchTests` 之前设置变量 `OptionalCatchTestLauncher`，指定一个启动命令来运行测试。例如，为了使用 `MPI` 运行某些测试，而其他测试顺序运行，可以这样写：

```cmake
set(OptionalCatchTestLauncher ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} ${NUMPROC})
ParseAndAddCatchTests(mpi_foo)
unset(OptionalCatchTestLauncher)
ParseAndAddCatchTests(bar)
```

### `CatchShardTests.cmake`

> `CatchShardTests.cmake` 于 Catch2 3.1.0 版本中引入。

`CatchShardTests.cmake` 提供了一个函数 `catch_add_sharded_tests(TEST_BINARY)`，用于将 `TEST_BINARY` 中的测试拆分为多个分片（shard）。每个分片中的测试及其顺序是随机的，并且每次调用 CTest 时种子都会改变。

目前，此脚本提供了 3 个自定义选项：

- SHARD_COUNT - 将目标测试拆分的分片数量
- REPORTER - 用于测试的报告器规范
- TEST_SPEC - 用于过滤测试的测试规范

示例用法：

```c++
include(CatchShardTests)

catch_add_sharded_tests(foo-tests
  SHARD_COUNT 4
  REPORTER "xml::out=-"
  TEST_SPEC "A"
)

catch_add_sharded_tests(tests
  SHARD_COUNT 8
  REPORTER "xml::out=-"
  TEST_SPEC "B"
)
```

这将注册总共 12 个 CTest 测试（4 + 8 个分片），以运行来自 `foo-tests` 测试二进制文件的分片，并通过测试规范进行过滤。

需要注意的是，此脚本目前是一个概念验证，用于在每次 CTest 运行时重新生成分片种子，因此不支持（目前也不打算支持）[`catch_discover_tests`](https://github.com/catchorg/Catch2/blob/devel/docs/cmake-integration.md#catch_discover_tests) 的所有自定义选项。

## CMake project options

Catch2 的 CMake 项目还为使用它的其他项目提供了一些选项。这些选项包括：

- `BUILD_TESTING`  
  当设置为 `ON` 且项目未被用作子项目时，将构建 Catch2 的测试二进制文件。默认为 `ON`。

- `CATCH_INSTALL_DOCS`  
  当设置为 `ON` 时，Catch2 的文档将包含在安装中。默认为 `ON`。

- `CATCH_INSTALL_EXTRAS`  
  当设置为 `ON` 时，Catch2 的 `extras` 文件夹（上述的 CMake 脚本、调试器助手等）将包含在安装中。默认为 `ON`。

- `CATCH_DEVELOPMENT_BUILD`  
  当设置为 `ON` 时，配置构建以进行 Catch2 的开发。这意味着启用测试项目、警告等。默认为 `OFF`。

启用 `CATCH_DEVELOPMENT_BUILD` 还会启用进一步的配置自定义选项：

- `CATCH_BUILD_TESTING`  
  当设置为 `ON` 时，将构建 Catch2 的 SelfTest 项目。默认为 `ON`。注意，Catch2 也遵守 `BUILD_TESTING` CMake 变量，因此需要两者都设置为 `ON` 才能构建 SelfTest，任意一个设置为 `OFF` 都会禁用 SelfTest 的构建。

- `CATCH_BUILD_EXAMPLES`  
  当设置为 `ON` 时，将构建 Catch2 的使用示例。默认为 `OFF`。

- `CATCH_BUILD_EXTRA_TESTS`  
  当设置为 `ON` 时，将构建 Catch2 的额外测试。默认为 `OFF`。

- `CATCH_BUILD_FUZZERS`  
  当设置为 `ON` 时，将构建 Catch2 的模糊测试入口点。默认为 `OFF`。

- `CATCH_ENABLE_WERROR`  
  当设置为 `ON` 时，添加 `-Werror` 或等效标志到编译命令中。默认为 `ON`。

- `CATCH_BUILD_SURROGATES`  
  当设置为 `ON` 时，每个 Catch2 头文件将单独编译，以确保它们是自给自足的。默认为 `OFF`。

## CATCH_CONFIG_* customization options in CMake

> 对 `CATCH_CONFIG_*` 选项的 CMake 支持是在 Catch2 3.0.1 版本中引入的。

由于新的独立编译模型，所有来自[编译时配置文档](https://github.com/catchorg/Catch2/blob/devel/docs/configuration.md#top)的选项也可以通过 Catch2 的 CMake 进行设置。要设置这些选项，将你想要的选项定义为 `ON`，例如 `-DCATCH_CONFIG_NOSTDOUT=ON`。

请注意，将选项设置为 `OFF` 并不会禁用它。要强制禁用一个选项，你需要将该选项的 `_NO_` 形式设置为 `ON`，例如 `-DCATCH_CONFIG_NO_COLOUR_WIN32=ON`。

为了总结配置选项的行为，以下是一个示例：

| `-DCATCH_CONFIG_COLOUR_WIN32` | `-DCATCH_CONFIG_NO_COLOUR_WIN32` |      Result |
|-------------------------------|----------------------------------|-------------|
|                          `ON` |                             `ON` |       error |
|                          `ON` |                            `OFF` |    force-on |
|                         `OFF` |                             `ON` |   force-off |
|                         `OFF` |                            `OFF` | auto-detect |

## Installing Catch2 from git repository

如果你无法通过包管理器安装 Catch2（例如，Ubuntu 16.04 仅提供版本为 1.2.0 的 Catch），你可能希望从代码仓库安装它。假设你有足够的权限，你可以直接将其安装到默认位置，如下所示：

```shell
$ git clone https://github.com/catchorg/Catch2.git
$ cd Catch2
$ cmake -B build -S . -DBUILD_TESTING=OFF
$ sudo cmake --build build/ --target install
```

如果你没有超级用户权限，则在配置构建时还需要指定 [CMAKE_INSTALL_PREFIX](https://cmake.org/cmake/help/latest/variable/CMAKE_INSTALL_PREFIX.html)，然后相应地修改对 [find_package](https://cmake.org/cmake/help/latest/command/find_package.html) 的调用。

## Installing Catch2 from vcpkg

或者，你可以使用 vcpkg 依赖管理器来构建和安装 Catch2：

```shell
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.sh
./vcpkg integrate install
./vcpkg install catch2
```

vcpkg 中的 catch2 端口由微软团队成员和社区贡献者保持更新。如果版本过时，请在 vcpkg 仓库中创建一个问题或拉取请求。

## Installing Catch2 from Bazel

Catch2 现已成为 Bazel Central Registry 中的受支持模块。你只需在 `MODULE.bazel` 文件中添加一行即可；有关最新支持的版本，请参阅 [https://registry.bazel.build/modules/catch2](https://registry.bazel.build/modules/catch2)。

然后，你可以按照以下方式将 `catch2_main` 添加到每个 C++ 测试构建规则中：

```
cc_test(
    name = "example_test",
    srcs = ["example_test.cpp"],
    deps = [
        ":example",
        "@catch2//:catch2_main",
    ],
)
```
