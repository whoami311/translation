# Magic Enum C++

这是一个仅头文件的C++17库，提供枚举类型的静态反射功能，可以在不使用任何宏或模板代码的情况下处理任意枚举类型。

## Features & Examples

- Basic

```c++
#include <magic_enum/magic_enum.hpp>
#include <iostream>

enum class Color : { RED = -10, BLUE = 0, GREEN = 10 };

int main() {
  Color c1 = Color::RED;
  std::cout << magic_enum::enum_name(c1) << std::endl; // RED
  return 0;
}
```

- Enum value to string

```c++
Color color = Color::RED;
auto color_name = magic_enum::enum_name(color);
// color_name -> "RED"
```

- String to enum value

```c++
std::string color_name{"GREEN"};
auto color = magic_enum::enum_cast<Color>(color_name);
if (color.has_value()) {
  // color.value() -> Color::GREEN
}

// case insensitive enum_cast
auto color = magic_enum::enum_cast<Color>(value, magic_enum::case_insensitive);

// enum_cast with BinaryPredicate
auto color = magic_enum::enum_cast<Color>(value, [](char lhs, char rhs) { return std::tolower(lhs) == std::tolower(rhs); }

// enum_cast with default
auto color_or_default = magic_enum::enum_cast<Color>(value).value_or(Color::NONE);
```

- Integer to enum value

```c++
int color_integer = 2;
auto color = magic_enum::enum_cast<Color>(color_integer);
if (color.has_value()) {
  // color.value() -> Color::BLUE
}

auto color_or_default = magic_enum::enum_cast<Color>(value).value_or(Color::NONE);
```

- Indexed access to enum value

```c++
std::size_t i = 0;
Color color = magic_enum::enum_value<Color>(i);
// color -> Color::RED
```

- Enum value sequence

```c++
constexpr auto colors = magic_enum::enum_values<Color>();
// colors -> {Color::RED, Color::BLUE, Color::GREEN}
// colors[0] -> Color::RED
```

- Number of enum elements

```c++
constexpr std::size_t color_count = magic_enum::enum_count<Color>();
// color_count -> 3
```

- Enum value to integer

```c++
Color color = Color::RED;
auto color_integer = magic_enum::enum_integer(color); // or magic_enum::enum_underlying(color);
// color_integer -> 1
```

- Enum names sequence

```c++
constexpr auto color_names = magic_enum::enum_names<Color>();
// color_names -> {"RED", "BLUE", "GREEN"}
// color_names[0] -> "RED"
```

- Enum entries sequence

```c++
constexpr auto color_entries = magic_enum::enum_entries<Color>();
// color_entries -> {{Color::RED, "RED"}, {Color::BLUE, "BLUE"}, {Color::GREEN, "GREEN"}}
// color_entries[0].first -> Color::RED
// color_entries[0].second -> "RED"
```

- Enum fusion for multi-level switch/case statements

```c++
switch (magic_enum::enum_fuse(color, direction).value()) {
  case magic_enum::enum_fuse(Color::RED, Directions::Up).value(): // ...
  case magic_enum::enum_fuse(Color::BLUE, Directions::Down).value(): // ...
// ...
}
```

- Enum switch runtime value as constexpr constant

```c++
Color color = Color::RED;
magic_enum::enum_switch([] (auto val) {
  constexpr Color c_color = val;
  // ...
}, color);
```

- Enum iterate for each enum as constexpr constant

```c++
magic_enum::enum_for_each<Color>([] (auto val) {
  constexpr Color c_color = val;
  // ...
});
```

- Check if enum contains

```c++
magic_enum::enum_contains(Color::GREEN); // -> true
magic_enum::enum_contains<Color>(2); // -> true
magic_enum::enum_contains<Color>(123); // -> false
magic_enum::enum_contains<Color>("GREEN"); // -> true
magic_enum::enum_contains<Color>("fda"); // -> false
```

- Enum index in sequence

```c++
constexpr auto color_index = magic_enum::enum_index(Color::BLUE);
// color_index.value() -> 1
// color_index.has_value() -> true
```

- Functions for flags

```c++
enum Directions : std::uint64_t {
  Left = 1,
  Down = 2,
  Up = 4,
  Right = 8,
};
template <>
struct magic_enum::customize::enum_range<Directions> {
  static constexpr bool is_flags = true;
};

magic_enum::enum_flags_name(Directions::Up | Directions::Right); // -> "Directions::Up|Directions::Right"
magic_enum::enum_flags_contains(Directions::Up | Directions::Right); // -> true
magic_enum::enum_flags_cast(3); // -> "Directions::Left|Directions::Down"
```

- Enum type name

```c++
Color color = Color::RED;
auto type_name = magic_enum::enum_type_name<decltype(color)>();
// type_name -> "Color"
```

- IOstream operator for enum

```c++
using magic_enum::iostream_operators::operator<<; // out-of-the-box ostream operators for enums.
Color color = Color::BLUE;
std::cout << color << std::endl; // "BLUE"
```

```c++
using magic_enum::iostream_operators::operator>>; // out-of-the-box istream operators for enums.
Color color;
std::cin >> color;
```

- Bitwise operator for enum

```c++
enum class Flags { A = 1 << 0, B = 1 << 1, C = 1 << 2, D = 1 << 3 };
using namespace magic_enum::bitwise_operators; // out-of-the-box bitwise operators for enums.
// Support operators: ~, |, &, ^, |=, &=, ^=.
Flags flags = Flags::A | Flags::B & ~Flags::C;
```

- Checks whether type is an Unscoped enumeration.

```c++
enum color { red, green, blue };
enum class direction { left, right };

magic_enum::is_unscoped_enum<color>::value -> true
magic_enum::is_unscoped_enum<direction>::value -> false
magic_enum::is_unscoped_enum<int>::value -> false

// Helper variable template.
magic_enum::is_unscoped_enum_v<color> -> true
```

- Checks whether type is an Scoped enumeration.

```c++
enum color { red, green, blue };
enum class direction { left, right };

magic_enum::is_scoped_enum<color>::value -> false
magic_enum::is_scoped_enum<direction>::value -> true
magic_enum::is_scoped_enum<int>::value -> false

// Helper variable template.
magic_enum::is_scoped_enum_v<direction> -> true
```

- Static storage enum variable to string This version is much lighter on the compile times and is not restricted to the enum_range limitation.

```c++
constexpr Color color = Color::BLUE;
constexpr auto color_name = magic_enum::enum_name<color>();
// color_name -> "BLUE"
```

- `containers::array` array container for enums.

```c++
magic_enum::containers::array<Color, RGB> color_rgb_array {};
color_rgb_array[Color::RED] = {255, 0, 0};
color_rgb_array[Color::GREEN] = {0, 255, 0};
color_rgb_array[Color::BLUE] = {0, 0, 255};
magic_enum::containers::get<Color::BLUE>(color_rgb_array) // -> RGB{0, 0, 255}
```

- `containers::bitset` bitset container for enums.

```c++
constexpr magic_enum::containers::bitset<Color> color_bitset_red_green {Color::RED|Color::GREEN};
bool all = color_bitset_red_green.all();
// all -> false
// Color::BLUE is missing
bool test = color_bitset_red_green.test(Color::RED);
// test -> true
```

- `containers::set` set container for enums.

```c++
auto color_set = magic_enum::containers::set<Color>();
bool empty = color_set.empty();
// empty -> true
color_set.insert(Color::GREEN);
color_set.insert(Color::BLUE);
color_set.insert(Color::RED);
std::size_t size = color_set.size();
// size -> 3
```

- Improved UB-free "SFINAE-friendly" underlying_type.

```c++
magic_enum::underlying_type<color>::type -> int

// Helper types.
magic_enum::underlying_type_t<Direction> -> int
```

## Remarks

- `magic_enum` 并不假装是枚举反射的灵丹妙药，它最初是为小型枚举设计的。
- 在使用之前，请阅读功能的限制。

## Integration

- 你需要添加所需的文件 magic_enum.hpp，以及可选的其他头文件，来自 include 目录或发布包。或者，你可以使用 CMake 构建该库。

- 如果你的项目使用 vcpkg 管理外部依赖，那么你可以使用 magic-enum 包。

- 如果你使用 Conan 来管理依赖，只需将 `magic_enum/x.y.z` 添加到 Conan 的 requires 中，其中 `x.y.z` 是你想使用的版本。

- 如果你使用 Build2 来构建和管理依赖，将 `depends: magic_enum ^x.y.z` 添加到清单文件中，其中 `x.y.z` 是你想使用的版本。然后，你可以使用 `magic_enum%lib{magic_enum}` 来导入目标。

- 或者，你也可以使用类似 CPM 的工具，它基于 CMake 的 `Fetch_Content` 模块。

  ```cmake
  CPMAddPackage(
      NAME magic_enum
      GITHUB_REPOSITORY Neargye/magic_enum
      GIT_TAG x.y.z # Where `x.y.z` is the release version you want to use.
  )
  ```

- Bazel 也被支持，只需将以下内容添加到你的 `WORKSPACE` 文件中：

  ```
  http_archive(
      name = "magic_enum",
      strip_prefix = "magic_enum-<commit>",
      urls = ["https://github.com/Neargye/magic_enum/archive/<commit>.zip"],
  )
  ```

  要在仓库内部使用 Bazel，可以执行以下操作：

  ```
  bazel build //...
  bazel test //...
  bazel run //example
  ```

  （请注意，您必须使用支持的编译器，或者通过 `export CC=<compiler>` 指定编译器。）

- 如果你使用的是 ROS，可以通过将 `<depend>magic_enum</depend>` 添加到 `package.xml` 中来包含此包，并在你的工作空间中包含该包。在 `CMakeLists.txt` 中添加以下内容：

  ```cmake
  find_package(magic_enum CONFIG REQUIRED)
  ...
  target_link_libraries(your_executable magic_enum::magic_enum)
  ```

## Limitations

- 此库使用了基于 `__PRETTY_FUNCTION__` / `__FUNCSIG__` 的编译器特定技巧。

- 要检查你的编译器是否支持 magic_enum，可以使用宏 `MAGIC_ENUM_SUPPORTED` 或 `constexpr` 常量 `magic_enum::is_magic_enum_supported`。如果在不受支持的编译器上使用 magic_enum，将会发生编译错误。要抑制此错误，可以定义宏 `MAGIC_ENUM_NO_CHECK_SUPPORT`。

- magic_enum 无法对枚举的前向声明进行反射。

### Enum Flags

- 对于枚举标志，向特定枚举类型的 `enum_range` 特化中添加 `is_flags`。`enum_range` 的特化必须注入到 `magic_enum::customize` 命名空间中。

  ```c++
  enum class Directions { Up = 1 << 0, Down = 1 << 1, Right = 1 << 2, Left = 1 << 3 };
  template <>
  struct magic_enum::customize::enum_range<Directions> {
    static constexpr bool is_flags = true;
  };
  ```

- `MAGIC_ENUM_RANGE_MAX` / `MAGIC_ENUM_RANGE_MIN` 不会影响枚举标志的最大数量。

- 如果一个枚举被声明为标志枚举，它的零值将不会被反射。

### Enum Range

- 枚举值必须在范围 `[MAGIC_ENUM_RANGE_MIN, MAGIC_ENUM_RANGE_MAX]` 内。

- 默认情况下，`MAGIC_ENUM_RANGE_MIN = -128`，`MAGIC_ENUM_RANGE_MAX = 127`。

- 如果你需要为所有枚举类型设置不同的默认范围，可以重新定义宏 `MAGIC_ENUM_RANGE_MIN` 和 `MAGIC_ENUM_RANGE_MAX`：

  ```c++
  #define MAGIC_ENUM_RANGE_MIN 0
  #define MAGIC_ENUM_RANGE_MAX 256
  #include <magic_enum/magic_enum.hpp>
  ```

- 如果你需要为特定的枚举类型设置不同的范围，可以为该枚举类型添加 `enum_range` 的特化。`enum_range` 的特化必须注入到 `magic_enum::customize` 命名空间中。

  ```c++
  #include <magic_enum/magic_enum.hpp>

  enum class number { one = 100, two = 200, three = 300 };

  template <>
  struct magic_enum::customize::enum_range<number> {
    static constexpr int min = 100;
    static constexpr int max = 300;
    // (max - min) must be less than UINT16_MAX.
  };
  ```

### Aliasing

如果一个值被别名，`magic_enum` 将无法正常工作。`magic_enum` 如何与别名一起工作是由编译器的实现决定的。

```c++
enum ShapeKind {
  ConvexBegin = 0,
  Box = 0, // Won't work.
  Sphere = 1,
  ConvexEnd = 2,
  Donut = 2, // Won't work too.
  Banana = 3,
  COUNT = 4
};
// magic_enum::enum_cast<ShapeKind>("Box") -> nullopt
// magic_enum::enum_name(ShapeKind::Box) -> "ConvexBegin"
```

解决此问题的一种可能方法是，在定义别名之前，先定义你希望被反射的枚举值：

```c++
enum ShapeKind {
  // Convex shapes, see ConvexBegin and ConvexEnd below.
  Box = 0,
  Sphere = 1,

  // Non-convex shapes.
  Donut = 2,
  Banana = 3,

  COUNT = Banana + 1,

  // Non-reflected aliases.
  ConvexBegin = Box,
  ConvexEnd = Sphere + 1
};
// magic_enum::enum_cast<ShapeKind>("Box") -> ShapeKind::Box
// magic_enum::enum_name(ShapeKind::Box) -> "Box"

// Non-reflected aliases.
// magic_enum::enum_cast<ShapeKind>("ConvexBegin") -> nullopt
// magic_enum::enum_name(ShapeKind::ConvexBegin) -> "Box"
```

在某些编译器中，枚举别名不被支持，例如 Visual Studio 2017，此时宏 `MAGIC_ENUM_SUPPORTED_ALIASES` 将未定义。

```c++
enum Number {
  one = 1,
  ONE = 1
};
// magic_enum::enum_cast<Number>("one") -> nullopt
// magic_enum::enum_name(Number::one) -> ""
// magic_enum::enum_cast<Number>("ONE") -> nullopt
// magic_enum::enum_name(Number::ONE) -> ""
```

### Other Compiler Issues

- 如果你遇到类似的消息：

  ```
  [...]
  note: constexpr evaluation hit maximum step limit; possible infinite loop?
  ```

  更改 `constexpr` 计算的深度限制：
    - MSVC: `/constexpr:depthN`，`/constexpr:stepsN` [https://docs.microsoft.com/en-us/cpp/build/reference/constexpr-control-constexpr-evaluation](https://docs.microsoft.com/en-us/cpp/build/reference/constexpr-control-constexpr-evaluation)
    - Clang: `-fconstexpr-depth=N`，`-fconstexpr-steps=N` [https://clang.llvm.org/docs/UsersManual.html#controlling-implementation-limits](https://clang.llvm.org/docs/UsersManual.html#controlling-implementation-limits)
    - GCC: `-fconstexpr-depth=N`，`-fconstexpr-loop-limit=N`，`-fconstexpr-ops-limit=N` [https://gcc.gnu.org/onlinedocs/gcc-9.2.0/gcc/C_002b_002b-Dialect-Options.html](https://gcc.gnu.org/onlinedocs/gcc-9.2.0/gcc/C_002b_002b-Dialect-Options.html)

- Visual Studio 的 IntelliSense 可能在分析 magic_enum 时出现一些问题。
- 模板中的枚举可能无法正确工作（尤其是在 Clang 上）。请参阅问题 #164 和 #65。

## Reference

- `enum_cast`：从字符串或整数中获取枚举值。
- `enum_value`：返回指定索引处的枚举值。
- `enum_values`：获取枚举值序列。
- `enum_count`：返回枚举值的数量。
- `enum_integer`：从枚举值获取整数值。
- `enum_name`：返回枚举值的名称。
- `enum_names`：获取字符串枚举名称序列。
- `enum_entries`：获取包含（枚举值，枚举名称）对的序列。
- `enum_index`：从枚举值获取枚举值序列中的索引。
- `enum_contains`：检查枚举是否包含指定值的枚举项。
- `enum_reflected`：如果枚举值在可反射值的范围内，则返回 `true`。
- `enum_type_name`：返回枚举类型的名称。
- `enum_fuse`：返回枚举值的双射混合。
- `enum_switch`：允许将运行时枚举值转换为 `constexpr` 上下文。
- `enum_for_each`：对所有 `constexpr` 枚举值调用函数。
- `enum_flags_*`：用于处理枚举标志的函数。
- `is_unscoped_enum`：检查类型是否是无作用域枚举。
- `is_scoped_enum`：检查类型是否是有作用域枚举。
- `underlying_type`：改进的、UB-free 且 “SFINAE-friendly” 基础类型。
- `ostream_operators`：用于枚举类型的 `ostream` 操作符。
- `istream_operators`：用于枚举类型的 `istream` 操作符。
- `bitwise_operators`：用于枚举类型的位运算操作符。
- `containers::array`：用于枚举类型的数组容器。
- `containers::bitset`：用于枚举类型的位集容器。
- `containers::set`：用于枚举类型的集合容器。

### Synopsis

- 在使用之前，请阅读功能的限制。

- 要检查 magic_enum 是否在编译器中受支持，可以使用宏 `MAGIC_ENUM_SUPPORTED` 或 `constexpr` 常量 `magic_enum::is_magic_enum_supported`。
  如果在不受支持的编译器上使用 magic_enum，将会发生编译错误。要抑制此错误，可以定义宏 `MAGIC_ENUM_NO_CHECK_SUPPORT`。
- 要添加自定义的枚举或类型名称，请参见示例。
- 要更改字符串类型或可选类型，请使用特定的宏：
  ```c++
  #include <my_lib/string.hpp>
  #include <my_lib/string_view.hpp>
  #define MAGIC_ENUM_USING_ALIAS_STRING using string = my_lib::String;
  #define MAGIC_ENUM_USING_ALIAS_STRING_VIEW using string_view = my_lib::StringView;
  #define MAGIC_ENUM_USING_ALIAS_OPTIONAL template <typename T> using optional = my_lib::Optional<T>;
  #include <magic_enum/magic_enum.hpp>
  ```
- 可选地，定义 `MAGIC_ENUM_CONFIG_FILE`，即在你的构建系统中指定包含已定义宏或常量的头文件路径，例如：
  ```c++
  #define MAGIC_ENUM_CONFIG_FILE "my_magic_enum_cfg.hpp"
  ```
  my_magic_enum_cfg.hpp:
  ```c++
  #include <my_lib/string.hpp>
  #include <my_lib/string_view.hpp>
  #define MAGIC_ENUM_USING_ALIAS_STRING using string = my_lib::String;
  #define MAGIC_ENUM_USING_ALIAS_STRING_VIEW using string_view = my_lib::StringView;
  #define MAGIC_ENUM_USING_ALIAS_OPTIONAL template <typename T> using optional = my_lib::Optional<T>;
  #define MAGIC_ENUM_RANGE_MIN 0
  #define MAGIC_ENUM_RANGE_MAX 255
  ```
