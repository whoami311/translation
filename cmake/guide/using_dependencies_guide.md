# Using Dependencies Guide

## 简介

项目经常会依赖其他项目、资源和制品。CMake 提供了多种方法将这些内容集成到构建中。项目和用户可以根据自身需求灵活选择最适合的方法。

引入依赖项的主要方法是 `find_package()` 命令和 `FetchContent` 模块。有时也会使用 `FindPkgConfig` 模块，尽管它缺乏其他两种方法的一些集成特性，在本指南中不再进一步讨论。

依赖项也可以通过自定义 dependency provider（依赖提供者）来提供。这可能是第三方包管理器，也可能是开发者实现的自定义代码。依赖提供者与上述主要方法协同工作，以扩展其灵活性。

## 通过 find_package() 使用预构建包

项目所需的包可能已经构建并存在于用户系统上的某个位置。该包可能是由 CMake 构建的，也可能完全使用了不同的构建系统。它甚至可能只是一个无需构建的文件集合。CMake 提供了 `find_package()` 命令来应对这些情况。它会搜索众所周知的位置以及由项目或用户提供的额外提示和路径。它还支持包组件和可选包。结果变量允许项目根据是否找到包或特定组件来自定义其行为。

在大多数情况下，项目通常应使用基本签名（Basic Signature）。多数时候，这仅涉及包名称，可能包含版本约束，以及如果依赖项是必需的，则使用 `REQUIRED` 关键字。也可以指定一组包组件。

```cmake
find_package(Catch2)
find_package(GTest REQUIRED)
find_package(Boost 1.79 COMPONENTS date_time)
```

`find_package()` 命令支持两种主要方法来执行搜索：

**Config mode**

    使用这种方法，命令会查找通常由包本身提供的文件。这是两种方法中更可靠的一种，因为包详情应该始终与包保持同步。

**Module mode**

    并非所有包都是 CMake-aware。许多包不提供支持配置模式所需的文件。对于这些情况，可以由项目或 CMake 单独提供一个查找模块（Find module）文件。查找模块通常是一种启发式实现，了解包通常提供的内容以及如何将该包呈现给项目。由于查找模块通常与包分开分发，因此它们不如配置模式可靠。它们通常单独维护，并且可能遵循不同的发布周期，因此很容易过时。

根据使用的参数，`find_package()` 可能会使用上述一种或两种方法。通过将选项限制为仅基本签名，配置模式和模块模式都可以用于满足依赖项。其他选项的存在可能会将调用限制为仅使用这两种方法中的一种，这可能降低命令查找依赖项的能力。有关这一复杂主题的完整详情，请参阅 `find_package()` 文档。

对于两种搜索方法，用户也可以在 `cmake(1)` 命令行中设置缓存变量，或在 `ccmake(1)` 或 `cmake-gui(1)` UI 工具中设置以影响和覆盖查找包的位置。有关如何设置缓存变量的更多信息，请参阅 User Interaction Guide。

### Config-file packages

第三方提供可执行文件、库、头文件及其他文件供 CMake 使用的首选方式是提供配置文件。这些是与包一起分发的文本文件，定义了 CMake 目标、变量、命令等。配置文件是一个普通的 CMake 脚本，由 `find_package()` 命令读取。

配置文件通常可以在名为 `lib/cmake/<PackageName>` 的目录中找到，尽管它们也可能位于其他位置（参见 Config Mode Search Procedure）。`<PackageName>` 通常是 `find_package()` 命令的第一个参数，甚至可能是唯一的参数。也可以使用 `NAMES` 选项指定替代名称：

```cmake
find_package(SomeThing
  NAMES
    SameThingOtherName   # Another name for the package
    SomeThing            # Also still look for its canonical name
)
```

配置文件必须命名为 `<PackageName>Config.cmake` 或 `<LowercasePackageName>-config.cmake`（本指南后续部分使用前者，但两者均受支持）。此文件是 CMake 访问包的入口点。在同一目录中可能存在一个可选的单独文件，名为 `<PackageName>ConfigVersion.cmake` 或 `<LowercasePackageName>-config-version.cmake`。CMake 使用此文件来确定包的版本是否满足 `find_package()` 调用中的任何版本约束。即使存在 `<PackageName>ConfigVersion.cmake` 文件，在调用 `find_package()` 时指定版本也是可选的。

如果找到 `<PackageName>Config.cmake` 文件且满足任何版本约束，则 `find_package()` 命令认为已找到该包，并假定整个包按设计完成。

可能会有额外的文件提供 CMake 命令或导入的目标供你使用。CMake 对这些文件不强制执行任何命名约定。它们通过使用 CMake 的 `include()` 命令与主 `<PackageName>Config.cmake` 文件相关联。通常，`<PackageName>Config.cmake` 文件会自动包含这些文件，因此除了调用 `find_package()` 外，通常不需要额外步骤。

如果包的位置在 CMake 已知的目录中，`find_package()` 调用应成功。CMake 已知的目录是平台特定的。例如，使用标准系统包管理器安装在 Linux 上的包会自动在 `/usr` 前缀下找到。同样地，安装在 Windows 的 `Program Files` 中的包也会自动找到。

如果包位于 CMake 不知道的位置（如 `/opt/mylib` 或 `$HOME/dev/prefix`），则不会自动找到这些包，除非给予帮助。这是一种常见情况，CMake 提供了几种方法让用户指定在哪里找到此类库。

可以在调用 CMake 时设置 `CMAKE_PREFIX_PATH` 变量。它被视为搜索配置文件的基本路径列表。安装在 `/opt/somepackage` 中的包通常会在 `/opt/somepackage/lib/cmake/somePackage/SomePackageConfig.cmake` 等位置安装配置文件。在这种情况下，`/opt/somepackage` 应添加到 `CMAKE_PREFIX_PATH` 中。

环境变量 `CMAKE_PREFIX_PATH` 也可以填充为搜索包的前缀。类似于 `PATH` 环境变量，这是一个列表，但它需要使用平台特定的环境变量列表项分隔符（Unix 上为 `:`，Windows 上为 `;`）。

`CMAKE_PREFIX_PATH` 变量在需要指定多个前缀或多包位于同一前缀下的情况下提供了便利。也可以通过设置与 `<PackageName>_DIR` 匹配的变量（如 `SomePackage_DIR`）来指定包的路径。请注意，这不是一个前缀，而应是包含配置样式包文件的目录的完整路径，如上述示例中的 `/opt/somepackage/lib/cmake/SomePackage`。有关可以影响搜索的其他 CMake 变量和环境变量，请参阅 `find_package()` 文档。

### Find Module Files

即使包不提供配置文件，如果存在 `FindSomePackage.cmake` 文件，仍可以使用 `find_package()` 命令找到这些包。查找模块文件与配置文件不同之处在于：

1. 查找模块文件不应由包本身提供。
  
2. `Find<PackageName>.cmake` 文件的存在并不表示包或其任何特定部分的可用性。

3. CMake 不会在 `CMAKE_PREFIX_PATH` 变量指定的位置中搜索 `Find<PackageName>.cmake` 文件。相反，CMake 会在 `CMAKE_MODULE_PATH` 变量给出的位置中搜索此类文件。用户在运行 CMake 时设置 `CMAKE_MODULE_PATH` 是常见的做法，CMake 项目也常常追加到 `CMAKE_MODULE_PATH` 以允许使用本地查找模块文件。

4. CMake 为一些第三方包提供了 `Find<PackageName>.cmake` 文件。这些文件对 CMake 来说是一个维护负担，它们往往落后于所关联包的最新版本。通常，不再向 CMake 添加新的查找模块。项目应鼓励上游包尽可能提供配置文件。如果未能成功，项目应为其提供的包编写自己的查找模块。

有关如何编写查找模块文件的详细讨论，请参阅查找模块（Find Modules）。

### Imported Targets

配置文件和查找模块文件都可以定义导入的目标（Imported Targets）。这些目标通常具有类似 `SomePrefix::ThingName` 的名称。如果这些目标可用，项目应优先使用它们，而不是使用可能提供的任何 CMake 变量。此类目标通常携带使用要求，并自动将头文件搜索路径、编译器定义等应用到链接到它们的其他目标（例如，使用 `target_link_libraries()`）。这比尝试手动使用变量应用相同的内容更健壮且更方便。请检查包或查找模块的文档，查看其定义了哪些导入目标（如果有）。

导入的目标还应该封装任何特定于配置的路径。这包括二进制文件（库、可执行文件）的位置、编译器标志及任何其他依赖于配置的数量。查找模块在提供这些细节方面可能不如配置文件可靠。

一个完整的示例，找到第三方包并使用其中的库，可能如下所示：

```cmake
cmake_minimum_required(VERSION 3.10)
project(MyExeProject VERSION 1.0.0)

# Make project-provided Find modules available
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

find_package(SomePackage REQUIRED)
add_executable(MyExe main.cpp)
target_link_libraries(MyExe PRIVATE SomePrefix::LibName)
```

需要注意的是，上述对 `find_package()` 的调用可以通过配置文件或查找模块来解析。它仅使用了基本签名（Basic Signature）支持的基本参数。例如，`FindSomePackage.cmake` 文件位于 `${CMAKE_CURRENT_SOURCE_DIR}/cmake` 目录中时，将允许 `find_package()` 命令通过模块模式成功执行。如果没有这样的模块文件存在，系统将会搜索配置文件。

## 通过 FetchContent 从源码下载并构建

依赖项不一定需要预先构建才能与 CMake 一起使用。它们可以作为主项目的一部分从源码构建。`FetchContent` 模块提供了下载内容（通常是源码，但也可以是任何内容）并将其添加到主项目中的功能，前提是依赖项也使用 CMake。依赖项的源码将与项目的其余部分一起构建，就像这些源码是项目自身源码的一部分一样。

一般的模式是项目应首先声明所有希望使用的依赖项，然后请求使这些依赖项可用。以下示例演示了这一原则（更多示例见示例部分）：

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
FetchContent_MakeAvailable(googletest Catch2)
```

支持多种下载方法，包括从 URL 下载并解压归档文件（支持多种归档格式），以及包括 Git、Subversion 和 Mercurial 在内的多种仓库格式。还可以使用自定义下载、更新和补丁命令来支持任意用例。

当使用 `FetchContent` 将依赖项添加到项目时，项目链接到依赖项的目标就像链接到项目中的任何其他目标一样。如果依赖项提供了形式为 `SomePrefix::ThingName` 的命名空间目标，项目应链接到这些目标而不是任何非命名空间目标。请参阅下一节了解推荐这样做的原因。

并非所有依赖项都能通过这种方式引入项目。一些依赖项定义的目标名称可能与其他项目目标或其他依赖项发生冲突。由 `add_executable()` 和 `add_library()` 创建的具体可执行文件和库目标是全局的，因此在整个构建过程中每个目标都必须是唯一的。如果一个依赖项会添加一个冲突的目标名称，则不能直接通过此方法将其引入构建。

## FetchContent 与 find_package() 的集成

*添加于版本 3.24。*

一些依赖项支持通过 `find_package()` 或 `FetchContent` 添加。此类依赖项必须确保在已安装和从源码构建的情景中定义相同的命名空间目标。这样一来，使用这些依赖项的项目可以透明地处理这两种情景，只要项目不使用仅由一种方法提供的其他内容即可。

项目可以通过在 `FetchContent_Declare()` 中使用 `FIND_PACKAGE_ARGS` 选项来表明它接受任一方法添加的依赖项。这允许 `FetchContent_MakeAvailable()` 首先尝试使用 `find_package()` 来满足依赖项，使用 `FIND_PACKAGE_ARGS` 关键字后的参数（如果有）。如果这种方法未能找到依赖项，则会如前所述从源码构建该依赖项。

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        703bd9caab50b139428cea1aaff9974ebee5742e # release-1.10.0
  FIND_PACKAGE_ARGS NAMES GTest
)
FetchContent_MakeAvailable(googletest)

add_executable(ThingUnitTest thing_ut.cpp)
target_link_libraries(ThingUnitTest GTest::gtest_main)
```

上述示例首先调用 `find_package(googletest NAMES GTest)`。CMake 提供了一个 `FindGTest` 模块，因此如果找到安装在某处的 GTest 包，它将会使该包可用，并且不会从源码构建依赖项。如果没有找到 GTest 包，则会从源码构建该依赖项。在这两种情况下，都期望定义 `GTest::gtest_main` 目标，因此我们将单元测试可执行文件链接到该目标。

通过 `FETCHCONTENT_TRY_FIND_PACKAGE_MODE` 变量也可以进行高级控制。可以将其设置为 `NEVER` 以禁用所有重定向到 `find_package()` 的操作。也可以设置为 `ALWAYS`，即使未指定 `FIND_PACKAGE_ARGS` 也尝试使用 `find_package()`（应谨慎使用）。

项目也可能决定某个特定依赖项必须从源码构建。这可能是因为需要依赖项的修补版本或未发布的版本，或者是为了满足要求所有依赖项都从源码构建的某些策略。项目可以通过在 `FetchContent_Declare()` 中添加 `OVERRIDE_FIND_PACKAGE` 关键字来强制执行此操作。对该依赖项的 `find_package()` 调用将被重定向到 `FetchContent_MakeAvailable()`。

```cmake
include(FetchContent)
FetchContent_Declare(
  Catch2
  URL https://intranet.mycomp.com/vendored/Catch2_2.13.4_patched.tgz
  URL_HASH MD5=abc123...
  OVERRIDE_FIND_PACKAGE
)

# The following is automatically redirected to FetchContent_MakeAvailable(Catch2)
find_package(Catch2)
```

对于更高级的用例，请参阅 `CMAKE_FIND_PACKAGE_REDIRECTS_DIR` 变量。

## 依赖提供者

*添加于版本 3.24。*

前一节讨论了项目可以用来指定其依赖项的技术。理想情况下，只要依赖项提供了预期的内容（通常只是一些导入的目标），项目不应该真的关心依赖项来自哪里。项目说明它需要什么，并且在没有其他细节的情况下也可能指定从哪里获取，以便它仍然能够开箱即用。

另一方面，开发者可能对控制依赖项如何提供给项目更感兴趣。你可能想要使用你自己构建的某个特定版本的包。你可能想要使用第三方包管理器。出于安全或性能原因，你可能想要将某些请求重定向到你控制下的不同URL。CMake通过依赖提供者（Dependency Providers）支持这些场景。

可以设置一个依赖提供者来拦截 `find_package()` 和 `FetchContent_MakeAvailable()` 调用。在回退到内置实现之前，提供者有机会满足此类请求，如果提供者未能履行，则会使用默认的方式。

只能设置一个依赖提供者，并且只能在 CMake 运行的非常早期的一个特定点进行设置。`CMAKE_PROJECT_TOP_LEVEL_INCLUDES` 变量列出了在处理第一个 `project()` 调用时（且仅在此调用期间）读取的 CMake 文件。这是唯一可以设置依赖提供者的时间点。整个项目中最多期望使用一个单一的提供者。

对于某些场景，用户不需要知道依赖提供者是如何设置的具体细节。第三方可能提供一个文件，可以添加到 `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` 中，代表用户设置依赖提供者。对于包管理器，推荐使用这种方法。开发者可以如下使用这样一个文件：

```shell
cmake -DCMAKE_PROJECT_TOP_LEVEL_INCLUDES=/path/to/package_manager/setup.cmake ...
```

有关如何实现自定义依赖提供者的详细信息，请参阅 `cmake_language(SET_DEPENDENCY_PROVIDER)` 命令。
