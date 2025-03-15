# Tutorial

## Getting Catch2

理想情况下，你应该通过 [CMake integration](https://github.com/catchorg/Catch2/blob/devel/docs/cmake-integration.md#top)（CMake 集成）来使用 Catch2。Catch2 还提供了 pkg-config 文件和两个文件（头文件 + cpp 文件）的分发方式，但本教程将假设你正在使用 CMake。如果你使用的是两个文件的分发方式，请记得将包含的头文件替换为 `catch_amalgamated.hpp`（[逐步说明](https://github.com/catchorg/Catch2/blob/devel/docs/migrate-v2-to-v3.md#how-to-migrate-projects-from-v2-to-v3)）。

## Writing tests

让我们从一个非常简单的例子开始（[代码](https://github.com/catchorg/Catch2/blob/devel/examples/010-TestCase.cpp)）。假设你编写了一个计算阶乘的函数，现在你想对它进行测试（暂时不考虑 TDD）。

```c++
unsigned int Factorial( unsigned int number ) {
    return number <= 1 ? number : Factorial(number-1)*number;
}
```

```c++
#include <catch2/catch_test_macros.hpp>

unsigned int Factorial( unsigned int number ) {
    return number <= 1 ? number : Factorial(number-1)*number;
}

TEST_CASE( "Factorials are computed", "[factorial]" ) {
    REQUIRE( Factorial(1) == 1 );
    REQUIRE( Factorial(2) == 2 );
    REQUIRE( Factorial(3) == 6 );
    REQUIRE( Factorial(10) == 3628800 );
}
```

这将编译成一个完整的可执行文件，该文件响应[命令行参数](https://github.com/catchorg/Catch2/blob/devel/docs/command-line.md#top)。如果你不带任何参数运行它，它将执行所有测试用例（在本例中只有一个），报告任何失败情况，并报告有多少测试通过和失败的摘要，最后返回失败的测试数量（这对于你只需要一个“是/否”的答案来说非常有用，例如：“它工作了吗？”）。

无论如何，如上所述的测试将会通过，但这里有一个错误。问题是 `Factorial(0)` 应该返回 1（根据其定义）。让我们在测试用例中添加这个断言：

```c++
TEST_CASE( "Factorials are computed", "[factorial]" ) {
    REQUIRE( Factorial(0) == 1 );
    REQUIRE( Factorial(1) == 1 );
    REQUIRE( Factorial(2) == 2 );
    REQUIRE( Factorial(3) == 6 );
    REQUIRE( Factorial(10) == 3628800 );
}
```

在另一个编译和运行周期后，我们将看到测试失败。输出将类似于以下内容：

```c++
Example.cpp:9: FAILED:
  REQUIRE( Factorial(0) == 1 )
with expansion:
  0 == 1
```

请注意，输出中包含原始表达式 `REQUIRE( Factorial(0) == 1 )` 以及调用 `Factorial` 函数时返回的实际值：`0`。

我们可以通过稍微修改 `Factorial` 函数来修复这个错误：

```c++
unsigned int Factorial( unsigned int number ) {
  return number > 1 ? Factorial(number-1)*number : 1;
}
```

### What did we do here?

尽管这是一个简单的测试，但它足以展示一些关于如何使用 Catch2 的要点。在继续之前，让我们花点时间考虑一下这些要点：

- 我们使用 `TEST_CASE` 宏引入测试用例。这个宏接受一个或两个字符串参数——一个自由形式的测试名称和可选的一个或多个标签（更多详情见[测试用例和部分](https://github.com/catchorg/Catch2/blob/devel/docs/tutorial.md#test-cases-and-sections)）。
- 测试用例会自动向测试运行器自我注册，用户不需要做任何额外的工作来确保它被测试框架识别。*请注意，你可以通过[命令行](https://github.com/catchorg/Catch2/blob/devel/docs/command-line.md#top)运行特定的测试或一组测试。*
- 单个测试断言是使用 `REQUIRE` 宏编写的。它接受一个布尔表达式，并使用表达式模板将其内部分解，以便在测试失败时可以单独地将其转换为字符串。

关于最后一点，请注意还有更多的测试宏可用，因为并不是所有有用的检查都可以表示为简单的布尔表达式。例如，检查一个表达式是否抛出异常是通过 `REQUIRE_THROWS` 宏完成的。稍后会有更多介绍。

## Test cases and sections

像大多数测试框架一样，Catch2 支持基于类的固件机制，其中单个测试是类的方法，而设置和清理工作可以在类型的构造函数/析构函数中完成。

然而，它们在 Catch2 中的使用很少见，因为惯用的 Catch2 测试会使用 **节（sections）** 来在测试代码之间共享设置和清理代码。这通过一个例子（[代码](https://github.com/catchorg/Catch2/blob/devel/examples/100-Fix-Section.cpp)）可以更好地解释：

```c++
TEST_CASE( "vectors can be sized and resized", "[vector]" ) {
    // This setup will be done 4 times in total, once for each section
    std::vector<int> v( 5 );

    REQUIRE( v.size() == 5 );
    REQUIRE( v.capacity() >= 5 );

    SECTION( "resizing bigger changes size and capacity" ) {
        v.resize( 10 );

        REQUIRE( v.size() == 10 );
        REQUIRE( v.capacity() >= 10 );
    }
    SECTION( "resizing smaller changes size but not capacity" ) {
        v.resize( 0 );

        REQUIRE( v.size() == 0 );
        REQUIRE( v.capacity() >= 5 );
    }
    SECTION( "reserving bigger changes capacity but not size" ) {
        v.reserve( 10 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 10 );
    }
    SECTION( "reserving smaller does not change size or capacity" ) {
        v.reserve( 0 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 5 );
    }
}
```

对于每个 `SECTION`，`TEST_CASE` 会**从头开始执行**。这意味着每次进入节时，都会有一个新构造的向量 `v`，我们知道它的大小为 5，容量至少为 5，因为这两个断言在进入节之前也会被检查。这种行为可能不适合设置成本较高的测试。每次通过测试用例将只执行一个且仅一个叶节（leaf section）。

节也可以嵌套，在这种情况下，父节可以多次进入，每次进入一个叶节。当多个测试共享部分设置时，嵌套节非常有用。继续上面的向量示例，你可以添加一个检查，确保 `std::vector::reserve` 不会移除未使用的多余容量，如下所示：

```c++
    SECTION( "reserving bigger changes capacity but not size" ) {
        v.reserve( 10 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 10 );
        SECTION( "reserving down unused capacity does not change capacity" ) {
            v.reserve( 7 );
            REQUIRE( v.size() == 5 );
            REQUIRE( v.capacity() >= 10 );
        }
    }
```

另一种看待节（sections）的方式是，它们是一种定义测试路径树的方法。每个节代表一个节点，最终的树以深度优先的方式遍历，每条路径只访问一个叶节点。

实际上，嵌套节的数量没有限制，只要你的编译器能够处理它们即可，但要记住，过度嵌套的节可能会变得难以阅读。根据经验，超过三层嵌套的节通常非常难以跟踪，并且不值得为了减少重复而这样做。

## BDD style testing

Catch2 还提供了一些对 BDD 风格测试的基本支持。它为 `TEST_CASE` 和 `SECTION` 提供了宏别名，你可以使用这些别名使生成的测试读起来像 BDD 规范。`SCENARIO` 作为 `TEST_CASE` 的别名，并带有 "Scenario: " 名称前缀。然后有 `GIVEN`、`WHEN`、`THEN`（以及带有 `AND_` 前缀的变体），它们作为 `SECTION` 的别名，同样以前缀宏名称命名。

更多关于这些宏的详细信息，请参阅参考文档中的[测试用例和部分](https://github.com/catchorg/Catch2/blob/devel/docs/test-cases-and-sections.md#top)部分，或者查看[使用 BDD 宏编写的向量示例](https://github.com/catchorg/Catch2/blob/devel/examples/120-Bdd-ScenarioGivenWhenThen.cpp)。

## Data and Type driven tests

Catch2 中的测试用例还可以由类型、输入数据或两者同时驱动。

更多详细信息，请参阅 Catch2 参考文档中的[类型参数化测试用例](https://github.com/catchorg/Catch2/blob/devel/docs/test-cases-and-sections.md#type-parametrised-test-cases)或[数据生成器](https://github.com/catchorg/Catch2/blob/devel/docs/generators.md#top)部分。

## Next steps

本页面是一个简短的介绍，帮助你快速上手 Catch2，并展示 Catch2 的基本功能。这里提到的功能可以让你走得相当远，但还有更多功能等待探索。不过，你可以在不断增长的文档[参考部分](https://github.com/catchorg/Catch2/blob/devel/docs/Readme.md#top)中逐步了解这些内容。
