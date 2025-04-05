# FetchContent

*在版本 3.11 中添加。*

注意：Using Dependencies Guide 提供了关于这一主题的高层次介绍。它概述了 `FetchContent` 模块在整个大局中的位置，包括它与 `find_package()` 命令的关系。在继续阅读下面的详细信息之前，建议先阅读该指南。

## Overview

此模块允许在配置时通过 `ExternalProject` 模块支持的任何方法填充内容。与 `ExternalProject_Add()` 在构建时下载不同，`FetchContent` 模块使内容立即可用，允许配置步骤在命令如 `add_subdirectory()`、`include()` 或 `file()` 操作中使用这些内容。

应将内容填充的详细信息与执行实际填充的命令分开定义。这种分离确保所有依赖项详细信息都在尝试使用它们填充内容之前已定义。这在更复杂的项目层次结构中尤为重要，因为依赖项可能在多个项目之间共享。

下面展示了一个典型的例子，声明一些依赖项的内容详细信息，然后通过单独调用确保它们被填充：

```cmake
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
)
FetchContent_Declare(
  myCompanyIcons
  URL      https://intranet.mycompany.com/assets/iconset_1.12.tar.gz
  URL_HASH MD5=5588a7b18261c20068beabfb4f530b87
)

FetchContent_MakeAvailable(googletest myCompanyIcons)
```

`FetchContent_MakeAvailable()` 命令确保指定的依赖项已被填充，无论是通过之前的调用填充，还是由它自己进行填充。在执行填充时，如果可能，它还会将这些依赖项添加到主构建中，以便主构建可以使用已填充项目的生成目标等内容。有关这些步骤如何执行的详细信息，请参阅该命令的文档。

在使用分层项目结构时，层次结构中较高级别的项目能够覆盖项目层次结构中较低级别所指定内容的声明细节。对于给定依赖项，最先声明的细节具有优先权，无论它们出现在项目层次结构中的哪个位置。同样，第一个尝试填充依赖项的调用“胜出”，后续的填充操作会重用第一次填充的结果，而不会重复填充。请参阅示例部分，其中演示了这一场景。

`FetchContent` 模块还支持在单个调用中定义并填充内容，而不检查内容是否已在其他地方被填充。这不应在项目中使用，但在 CMake script mode 下填充内容时可能是合适的。有关详细信息，请参阅 `FetchContent_Populate()`。

## Commands

### FetchContent_Declare

```cmake
FetchContent_Declare(
  <name>
  <contentOptions>...
  [EXCLUDE_FROM_ALL]
  [SYSTEM]
  [OVERRIDE_FIND_PACKAGE |
   FIND_PACKAGE_ARGS args...]
)
```

`FetchContent_Declare()` 函数记录描述如何填充指定内容的选项。如果这些细节在此项目的早期（无论在项目层次结构中的哪个位置）已经被记录，则此调用及之后所有针对相同内容 `<name>` 的调用都将被忽略。这种“首次记录，优先”的方法允许分层项目让父项目覆盖子项目的部分内容细节。

内容 `<name>` 可以是任何不含空格的字符串，但良好的实践是仅使用字母、数字和下划线。名称不区分大小写，并且应能明确表示其所代表的内容。通常这会是子项目的名称，或者是其顶级 `project()` 命令赋予的值（如果是 CMake 项目）。对于知名的公共项目，名称通常应该是该项目的官方名称。选择一个不寻常的名字会导致其他需要相同内容的项目不太可能使用相同的名称，从而导致内容被多次填充。

`<contentOptions>` 可以是 `ExternalProject_Add()` 命令所理解的任何下载、更新或补丁选项。配置、构建、安装和测试步骤被显式禁用，因此与这些步骤相关的选项将被忽略。`SOURCE_SUBDIR` 选项是一个例外，请参阅 `FetchContent_MakeAvailable()` 了解它如何影响行为的详细信息。

*自版本 3.30 起更改*：当策略 CMP0168 设置为 NEW 时，一些与输出和目录相关的选项将被忽略。详情请参阅策略文档。

在大多数情况下，`<contentOptions>` 只需定义几个选项来确定下载方法和特定于方法的细节，如提交标签或归档哈希。例如：

```cmake
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
)

FetchContent_Declare(
  myCompanyIcons
  URL      https://intranet.mycompany.com/assets/iconset_1.12.tar.gz
  URL_HASH MD5=5588a7b18261c20068beabfb4f530b87
)

FetchContent_Declare(
  myCompanyCertificates
  SVN_REPOSITORY svn+ssh://svn.mycompany.com/srv/svn/trunk/certs
  SVN_REVISION   -r12345
)
```

当从远程位置获取内容且您无法控制该服务器时，建议为 `GIT_TAG` 使用哈希值而不是分支或标签名称。提交哈希更加安全，并有助于确认下载的内容正是您期望的内容。

*自版本 3.14 起更改*：下载、更新或补丁步骤中的命令可以访问终端。这可能对于类似密码提示或实时显示命令进度的功能是必需的。

*在版本 3.22 中添加*：变量 `CMAKE_TLS_VERIFY`、`CMAKE_TLS_CAINFO`、`CMAKE_NETRC` 和 `CMAKE_NETRC_FILE` 现在为其对应的内容选项提供默认值，就像它们为 `ExternalProject_Add()` 所做的那样。以前，这些变量被 `FetchContent` 模块忽略。

*在版本 3.24 中添加*：

- `FIND_PACKAGE_ARGS`

此选项适用于 `FetchContent_MakeAvailable()` 命令可能首先尝试调用 `find_package()` 来满足 `<name>` 依赖项的场景。默认情况下，这样的调用会是简单的 `find_package(<name>)`，但可以使用 `FIND_PACKAGE_ARGS` 提供附加参数，这些参数会被追加到 `<name>` 后面。`FIND_PACKAGE_ARGS` 也可以在没有任何后续内容的情况下指定，这表示如果 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 设置为 `OPT_IN` 或未设置时，仍然可以调用 `find_package()`。

通常不建议将 `REQUIRED` 指定为 `FIND_PACKAGE_ARGS` 后的附加参数之一。这样做意味着 `find_package()` 调用必须成功，因此 `FetchContent_Declare()` 调用中指定的其他详细信息将没有机会作为回退方案被使用。

`FIND_PACKAGE_ARGS` 关键字之后的所有内容都会被追加到 `find_package()` 调用中，因此所有其他 `<contentOptions>` 必须出现在 `FIND_PACKAGE_ARGS` 关键字之前。如果在调用 `FetchContent_Declare()` 时变量 `CMAKE_FIND_PACKAGE_TARGETS_GLOBAL` 被设置为 `true`，那么如果没有明确指定 `GLOBAL` 关键字，它会被自动追加到 `find_package()` 的参数中。此外，如果未给出 `FIND_PACKAGE_ARGS`，但 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 被设置为 `ALWAYS`，`GLOBAL` 关键字同样会被追加。

当指定了 `FIND_PACKAGE_ARGS` 时，不能使用 `OVERRIDE_FIND_PACKAGE`。

Dependency Providers（依赖提供者） 讨论了另一种重定向 `FetchContent_MakeAvailable()` 调用的方法。`FIND_PACKAGE_ARGS` 旨在用于项目控制，而依赖提供者允许用户覆盖项目行为。

- `OVERRIDE_FIND_PACKAGE`

当 `FetchContent_Declare(<name> ...)` 调用包含此选项时，后续对 `find_package(<name> ...)` 的调用将确保已调用 `FetchContent_MakeAvailable(<name>)`，然后使用 `CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 目录中的配置包文件（这些文件通常由 `FetchContent_MakeAvailable()` 创建）。这实际上使 `FetchContent_MakeAvailable()` 覆盖了 `find_package()` 对指定依赖项的行为，从而允许前者满足后者的包需求。当指定了 `OVERRIDE_FIND_PACKAGE` 时，不能使用 `FIND_PACKAGE_ARGS`。

如果设置了 dependency provider（依赖提供者），并且项目针对 `<name>` 依赖项调用了 `find_package()`，则 `OVERRIDE_FIND_PACKAGE` 不会阻止提供者捕获该调用。依赖提供者始终有机会拦截任何直接调用 `find_package()` 的行为，除非该调用包含 `BYPASS_PROVIDER` 选项。

*在版本 3.25 中添加*：

- `SYSTEM`

如果提供了 `SYSTEM` 参数，则由 `FetchContent_MakeAvailable()` 添加的子目录的 `SYSTEM` 目录属性将被设置为 `true`。这将影响作为该命令的一部分创建的非导入目标。有关其影响的详细讨论，请参阅 `SYSTEM` 目标属性的文档。

*在版本 3.28 中添加*：

- `EXCLUDE_FROM_ALL`

如果提供了 `EXCLUDE_FROM_ALL` 参数，则通过 `FetchContent_MakeAvailable()` 添加的子目录中的目标默认不会包含在 `ALL` 目标中，并且可能会从 IDE 项目文件中排除。有关其影响的详细讨论，请参阅目录属性 `EXCLUDE_FROM_ALL` 的文档。

### FetchContent_MakeAvailable

*在版本 3.14 中添加。*

```cmake
FetchContent_MakeAvailable(<name1> [<name2>...])
```

此命令确保在返回时每个命名依赖项都可供项目使用。必须为每个依赖项调用过 `FetchContent_Declare()`，并且第一次这样的调用将控制如何使该依赖项可用，如下所述。

如果 `<lowercaseName>_SOURCE_DIR` 未设置：

- *在版本 3.24 中添加*：如果设置了 dependency provider（依赖提供者），则使用 `FETCHCONTENT_MAKEAVAILABLE_SERIAL` 作为第一个参数调用提供者的命令，随后是针对 `<name>` 的首次 `FetchContent_Declare()` 调用的参数。如果 `SOURCE_DIR` 或 `BINARY_DIR` 不是原始声明参数的一部分，它们将会以默认值添加。如果在声明细节时 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 被设置为 `NEVER`，则任何 `FIND_PACKAGE_ARGS` 都会被省略。`OVERRIDE_FIND_PACKAGE` 关键字也总是被省略。如果提供者满足了请求，`FetchContent_MakeAvailable()` 将认为该依赖项已处理，跳过以下剩余步骤，并继续处理列表中的下一个依赖项。
- *在版本 3.24 中添加*：如果允许，将调用 `find_package(<name> [<args>...])`，其中 `<args>...` 可能由 `FetchContent_Declare()` 中的 `FIND_PACKAGE_ARGS` 选项提供。在调用 `FetchContent_Declare()` 时，`FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 变量的值决定了 `FetchContent_MakeAvailable()` 是否可以调用 `find_package()`。如果在调用 `FetchContent_MakeAvailable()` 时，`CMAKE_FIND_PACKAGE_TARGETS_GLOBAL` 变量被设置为 `true`，即使在声明相关细节时该变量为 `false`，它仍然会影响通过调用 `find_package()` 创建的所有导入目标。

如果依赖项未通过提供者或 `find_package()` 调用得到满足，`FetchContent_MakeAvailable()` 将使用以下逻辑使该依赖项可用：

- 如果该依赖项在本次运行中已经被填充，则以与调用 `FetchContent_GetProperties()` 相同的方式设置 `<lowercaseName>_POPULATED`、`<lowercaseName>_SOURCE_DIR` 和 `<lowercaseName>_BINARY_DIR` 变量，然后跳过以下剩余步骤，并继续处理列表中的下一个依赖项。
- 使用之前调用 `FetchContent_Declare()` 记录的细节来填充依赖项。如果没有记录这样的细节，则停止并报出致命错误。可以使用 `FETCHCONTENT_SOURCE_DIR_<uppercaseName>` 来覆盖声明的细节，并改用指定位置提供的内容。
- *在版本 3.24 中添加*：确保 `CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 目录包含 `<lowercaseName>-config.cmake` 和 `<lowercaseName>-config-version.cmake` 文件（或者等效地，`<name>Config.cmake` 和 `<name>ConfigVersion.cmake` 文件）。`CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 变量指向的目录在每次 CMake 运行开始时都会被清空。如果在上一步填充依赖项后没有配置文件存在，则会写入一个最小化的文件，该文件包含任意的 `<lowercaseName>-extra.cmake` 或 `<name>Extra.cmake` 文件（带有 `OPTIONAL` 标志，因此这些文件可以缺失而不会生成警告）。同样地，如果没有配置版本文件存在，将会写入一个非常简单的版本文件，设置 `PACKAGE_VERSION_COMPATIBLE` 和 `PACKAGE_VERSION_EXACT` 为 `true`。这确保了所有未来对依赖项的 `find_package()` 调用都将使用重定向的配置文件，无论任何版本要求如何。CMake 无法自动确定任意依赖项的版本，因此不能设置 `PACKAGE_VERSION`。当依赖项通过下一步中的 `add_subdirectory()` 引入时，它可以选择覆盖 `CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 中生成的配置版本文件，并设置 `PACKAGE_VERSION`。依赖项还可以编写 `<lowercaseName>-extra.cmake` 或 `<name>Extra.cmake` 文件来执行自定义处理，或定义其正常（已安装）包配置文件通常会定义的任何变量（许多项目不进行任何自定义处理或设置任何变量，因此不需要这样做）。如果需要，主项目可以在依赖项项目未执行此操作的情况下编写这些文件。这允许主项目添加旧依赖项中缺少的细节，这些依赖项尚未或无法更新以支持此功能。有关示例，请参阅 Integrating With find_package()。
- 如果填充内容的顶层目录包含一个 `CMakeLists.txt` 文件，则调用 `add_subdirectory()` 将其添加到主构建中。没有 `CMakeLists.txt` 文件并不会导致错误，这使得该命令可以用于那些将下载内容放置在已知位置、但不需要或不支持直接添加到构建中的依赖项。
  - *在版本 3.18 中添加*：可以在声明的细节中提供 `SOURCE_SUBDIR` 选项，以查找顶层目录下的某个路径（即与 `ExternalProject_Add()` 命令使用 `SOURCE_SUBDIR` 的方式相同）。`SOURCE_SUBDIR` 提供的路径必须是相对路径，并且会被视为相对于顶层目录。它还可以指向不包含 `CMakeLists.txt` 文件的目录，甚至是不存在的目录。这可以用来避免添加在其顶层目录中包含 `CMakeLists.txt` 文件的项目。
  - *在版本 3.25 中添加*：如果在调用 `FetchContent_Declare()` 时包含了 `SYSTEM` 关键字，则 `SYSTEM` 关键字会被添加到 `add_subdirectory()` 命令中。
  - *在版本 3.28 中添加*：如果在调用 `FetchContent_Declare()` 时包含了 `EXCLUDE_FROM_ALL` 关键字，则 `EXCLUDE_FROM_ALL` 关键字会被添加到 `add_subdirectory()` 命令中。
  - *在版本 3.29 中添加*：在调用 `add_subdirectory()` 之前，`CMAKE_EXPORT_FIND_PACKAGE_NAME` 会被设置为依赖项名称。

项目应在其调用 `FetchContent_MakeAvailable()` 为任何一个依赖项之前，声明它们可能使用的所有依赖项的详细信息。这确保了如果这些依赖项也是其他一个或多个依赖项的子依赖项时，主项目仍然控制将要使用的详细信息（因为它会在依赖项有机会调用之前先声明它们）。在以下代码示例中，假设 `uses_other` 依赖项也在内部使用 `FetchContent` 来添加 `other` 依赖项：

```cmake
# WRONG: Should declare all details first
FetchContent_Declare(uses_other ...)
FetchContent_MakeAvailable(uses_other)

FetchContent_Declare(other ...)    # Will be ignored, uses_other beat us to it
FetchContent_MakeAvailable(other)  # Would use details declared by uses_other
```

```cmake
# CORRECT: All details declared first, so they will take priority
FetchContent_Declare(uses_other ...)
FetchContent_Declare(other ...)
FetchContent_MakeAvailable(uses_other other)
```

需要注意的是，`CMAKE_VERIFY_INTERFACE_HEADER_SETS` 在进入 `FetchContent_MakeAvailable()` 时会被显式设置为 `false`，并在命令返回之前恢复为其原始值。开发者通常只想验证主项目的头文件集，而不是任何依赖项的头文件集。对 `CMAKE_VERIFY_INTERFACE_HEADER_SETS` 变量的这种局部操作提供了这种直观的行为。你可以使用类似 `CMAKE_PROJECT_INCLUDE` 或 `CMAKE_PROJECT_<PROJECT-NAME>_INCLUDE` 的变量来为所有或部分依赖项重新启用验证。你还可以设置单个目标的 `VERIFY_INTERFACE_HEADER_SETS` 属性。

### FetchContent_Populate

TODO

### FetchContent_GetProperties

### FetchContent_SetPopulated

## Variables

一些缓存变量可以影响使用 `FetchContent_Declare()` 调用的详细信息来填充内容的行为。

注意：所有这些变量都是为了开发者自定义行为而设计的。它们通常不应由项目设置。

### FETCHCONTENT_BASE_DIR
### FETCHCONTENT_QUIET
### FETCHCONTENT_FULLY_DISCONNECTED
### FETCHCONTENT_UPDATES_DISCONNECTED
### FETCHCONTENT_TRY_FIND_PACKAGE_MODE
### FETCHCONTENT_SOURCE_DIR_<uppercaseName>
### FETCHCONTENT_UPDATES_DISCONNECTED_<uppercaseName>

## Examples

### Typical Case

这个首先较为直接的例子确保了一些流行的测试框架可用于主构建：

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
)
FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        605a34765aa5d5ecbf476b4598a862ada971b0cc # v3.0.1
)

# After the following call, the CMake targets defined by googletest and
# Catch2 will be available to the rest of the build
FetchContent_MakeAvailable(googletest Catch2)
```

### Integrating With find_package()

对于前面的例子，如果用户希望首先尝试通过 `find_package()` 查找 `googletest` 和 `Catch2`，再尝试从源码下载和构建它们，可以将 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 变量设置为 `ALWAYS`。但这可能会影响项目中所有其他对 `FetchContent_Declare()` 的调用，这可能是不可接受的。可以通过在声明的详细信息中添加 `FIND_PACKAGE_ARGS` 并保持 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 未设置或设置为 `OPT_IN`，来只为这两个依赖项启用此行为：

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
  FIND_PACKAGE_ARGS NAMES GTest
)
FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        605a34765aa5d5ecbf476b4598a862ada971b0cc # v3.0.1
  FIND_PACKAGE_ARGS
)

# This will try calling find_package() first for both dependencies
FetchContent_MakeAvailable(googletest Catch2)
```

对于 `Catch2`，不需要向 `find_package()` 提供额外的参数，因此在 `FIND_PACKAGE_ARGS` 关键字后不提供额外的参数。对于 `googletest`，其包更常被称为 `GTest`，因此需要添加参数以支持通过该名称查找。

如果用户希望禁止 `FetchContent_MakeAvailable()` 为任何依赖项调用 `find_package()`，即使它在其声明的详细信息中提供了 `FIND_PACKAGE_ARGS`，也可以将 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 设置为 `NEVER`。

如果项目希望指示这两个依赖项应从源码下载并构建，并且 `find_package()` 调用应重定向到使用已构建的依赖项，则在声明内容详细信息时应使用 `OVERRIDE_FIND_PACKAGE` 选项：

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
  OVERRIDE_FIND_PACKAGE
)
FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        605a34765aa5d5ecbf476b4598a862ada971b0cc # v3.0.1
  OVERRIDE_FIND_PACKAGE
)

# The following will automatically forward through to FetchContent_MakeAvailable()
find_package(googletest)
find_package(Catch2)
```

CMake 提供了一个 FindGTest 模块，该模块定义了一些变量，旧项目可能会使用这些变量，而不是链接到导入的目标。为了支持这些情况，我们可以提供一个额外的文件。遵循 `FetchContent` 的“先定义者优先”理念，只有在其他内容尚未完成定义时，我们才会写入该文件。

```cmake
FetchContent_MakeAvailable(googletest)

if(NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/googletest-extra.cmake AND
   NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/googletestExtra.cmake)
  file(WRITE ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/googletest-extra.cmake
[=[
if("${GTEST_LIBRARIES}" STREQUAL "" AND TARGET GTest::gtest)
  set(GTEST_LIBRARIES GTest::gtest)
endif()
if("${GTEST_MAIN_LIBRARIES}" STREQUAL "" AND TARGET GTest::gtest_main)
  set(GTEST_MAIN_LIBRARIES GTest::gtest_main)
endif()
if("${GTEST_BOTH_LIBRARIES}" STREQUAL "")
  set(GTEST_BOTH_LIBRARIES ${GTEST_LIBRARIES} ${GTEST_MAIN_LIBRARIES})
endif()
]=])
endif()
```

项目更可能使用 `find_package(GTest)` 而不是 `find_package(googletest)`，但可以利用 `CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 区域将后者作为前者的依赖项引入。这很可能足以满足典型的 `find_package(GTest)` 调用。

```cmake
FetchContent_MakeAvailable(googletest)

if(NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/gtest-config.cmake AND
   NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/GTestConfig.cmake)
  file(WRITE ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/gtest-config.cmake
[=[
include(CMakeFindDependencyMacro)
find_dependency(googletest)
]=])
endif()

if(NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/gtest-config-version.cmake AND
   NOT EXISTS ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/GTestConfigVersion.cmake)
  file(WRITE ${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/gtest-config-version.cmake
[=[
include(${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/googletest-config-version.cmake OPTIONAL)
if(NOT PACKAGE_VERSION_COMPATIBLE)
  include(${CMAKE_FIND_PACKAGE_REDIRECTS_DIR}/googletestConfigVersion.cmake OPTIONAL)
endif()
]=])
endif()
```

### Overriding Where To Find CMakeLists.txt

如果子项目的 `CMakeLists.txt` 文件不在其源码树的顶层目录，可以使用 `SOURCE_SUBDIR` 选项告诉 `FetchContent` 在哪里找到它。下面的例子展示了如何使用该选项，并在将其引入主构建之前设置一个对子项目有意义的变量（设置为 `INTERNAL` 缓存变量以避免与策略 `CMP0077` 发生问题）：

```cmake
include(FetchContent)
FetchContent_Declare(
  protobuf
  GIT_REPOSITORY https://github.com/protocolbuffers/protobuf.git
  GIT_TAG        ae50d9b9902526efd6c7a1907d09739f959c6297 # v3.15.0
  SOURCE_SUBDIR  cmake
)
set(protobuf_BUILD_TESTS OFF CACHE INTERNAL "")
FetchContent_MakeAvailable(protobuf)
```

### Complex Dependency Hierarchies

在更复杂的项目层次结构中，依赖关系可能会更加复杂。考虑一个层次结构，其中 `projA` 是顶层项目，它直接依赖于项目 `projB` 和 `projC`。`projB` 和 `projC` 都可以独立构建，同时它们也都依赖于另一个项目 `projD`。此外，`projB` 还依赖于 `projE`。本示例假设这五个项目都托管在公司的 Git 服务器上。每个项目的 `CMakeLists.txt` 文件可能包含如下部分：

projA：

```cmake
include(FetchContent)
FetchContent_Declare(
  projB
  GIT_REPOSITORY git@mycompany.com:git/projB.git
  GIT_TAG        4a89dc7e24ff212a7b5167bef7ab079d
)
FetchContent_Declare(
  projC
  GIT_REPOSITORY git@mycompany.com:git/projC.git
  GIT_TAG        4ad4016bd1d8d5412d135cf8ceea1bb9
)
FetchContent_Declare(
  projD
  GIT_REPOSITORY git@mycompany.com:git/projD.git
  GIT_TAG        origin/integrationBranch
)
FetchContent_Declare(
  projE
  GIT_REPOSITORY git@mycompany.com:git/projE.git
  GIT_TAG        v2.3-rc1
)

# Order is important, see notes in the discussion further below
FetchContent_MakeAvailable(projD projB projC)
```

projB：

```cmake
include(FetchContent)
FetchContent_Declare(
  projD
  GIT_REPOSITORY git@mycompany.com:git/projD.git
  GIT_TAG        20b415f9034bbd2a2e8216e9a5c9e632
)
FetchContent_Declare(
  projE
  GIT_REPOSITORY git@mycompany.com:git/projE.git
  GIT_TAG        68e20f674a48be38d60e129f600faf7d
)

FetchContent_MakeAvailable(projD projE)
```

projC：

```cmake
include(FetchContent)
FetchContent_Declare(
  projD
  GIT_REPOSITORY git@mycompany.com:git/projD.git
  GIT_TAG        7d9a17ad2c962aa13e2fbb8043fb6b8a
)

FetchContent_MakeAvailable(projD)
```

在上述内容中应注意几个关键点：

- `projB` 和 `projC` 为 `projD` 定义了不同的内容细节，但 `projA` 也为 `projD` 定义了一组内容细节。因为 `projA` 将首先定义这些细节，所以来自 `projB` 和 `projC` 的细节将不会被使用。由 `projA` 定义的覆盖细节不需要与 `projB` 或 `projC` 中的任一细节匹配，但高层项目有责任确保其定义的细节对子项目仍然有意义。

- 在 `projA` 调用 `FetchContent_MakeAvailable()` 时，`projD` 被列在 `projB` 和 `projC` 之前，因此它将在 `projB` 或 `projC` 之前被填充。虽然 `projA` 不需要这样做，但这样做可以确保 `projA` 完全控制 `projD` 被引入构建时的环境（目录属性在此特别相关）。

- 虽然 `projA` 为 `projE` 定义了内容细节，但它不需要显式调用 `FetchContent_MakeAvailable(projE)` 或 `FetchContent_Populate(projD)`。相反，它将此任务留给子项目 `projB`。对于高层项目，通常只需定义覆盖内容细节，并将实际填充工作留给子项目即可。这避免了在项目层次结构的每一层不必要地重复相同的内容，但这仅应在依赖项设置的目录属性不影响共享依赖项（本例中的 `projE`）填充的情况下进行。

### Populating Content Without Adding It To The Build

项目并不总是需要将填充的内容添加到构建中。有时，项目只是希望将下载的内容放置在一个可预测的位置以供使用。下一个示例确保一组标准的公司工具链文件（甚至可能包括工具链二进制文件本身）足够早地可用，以便在同一次构建中使用。

```cmake
cmake_minimum_required(VERSION 3.14)

include(FetchContent)
FetchContent_Declare(
  mycom_toolchains
  URL  https://intranet.mycompany.com//toolchains_1.3.2.tar.gz
)
FetchContent_MakeAvailable(mycom_toolchains)

project(CrossCompileExample)
```

项目可以配置为使用下载的工具链之一，如下所示：

```shell
cmake -DCMAKE_TOOLCHAIN_FILE=_deps/mycom_toolchains-src/toolchain_arm.cmake /path/to/src
```

当 CMake 处理 `CMakeLists.txt` 文件时，它会将 tarball 下载并解压到构建目录下的 `_deps/mycompany_toolchains-src` 目录中。`CMAKE_TOOLCHAIN_FILE` 变量直到达到 `project()` 命令时才会被使用，在这一点上，CMake 会相对于构建目录查找命名的工具链文件。因为此时 tarball 已经被下载并解包，工具链文件将会就位，即使是在构建目录中首次运行 **cmake** 时也是如此。

### Populating Content In CMake Script Mode

最后一个示例演示了如何使用 CMake 的脚本模式下载并解压固件 tarball。对 `FetchContent_Populate()` 的调用指定了所有的内容细节，解压后的固件将被放置在当前工作目录下的 `firmware` 目录中。

```cmake
# NOTE: Intended to be run in script mode with cmake -P
include(FetchContent)
FetchContent_Populate(
  firmware
  URL        https://mycompany.com/assets/firmware-1.23-arm.tar.gz
  URL_HASH   MD5=68247684da89b608d466253762b0ff11
  SOURCE_DIR firmware
)
```
